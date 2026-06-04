# GitHub Copilot Prompts — Token Exchange Service Testing

> Use these prompts in GitHub Copilot Chat inside IntelliJ.
> Each prompt is self-contained — paste as-is, adjust placeholders marked with `< >`.

---

## Approach 1 — Dark Launch (Request Forking)

### Context

> Give Copilot this context first before using any prompt below.

```
I have a Spring Boot WebFlux token exchange service.
It exposes POST /token/exchange.
Request body: { "subject_token": "...", "grant_type": "...", "client_id": "..." }
Blue pod is live on ingress: cbxd.prod.com
Green pod is new version, no traffic, ingress: cbx.pvt.com
I want to fork requests silently from blue to green for testing.
Green response is always discarded. Errors are always suppressed.
Blue must never be affected by green's behaviour.
```

---

### Prompt 1 — Generate WebFilter to fork requests

```
Create a Spring WebFlux WebFilter called DarkLaunchFilter.

Requirements:
- Intercept POST /token/exchange requests only
- Cache the request body using ServerWebExchangeUtils.cacheRequestBodyAndRequest
  so it can be read twice without conflict
- Fork the request asynchronously to https://cbx.pvt.com/token/exchange
  using WebClient
- Forward the original request body and Content-Type header
- Discard green pod response completely
- Suppress all errors from green pod — never let them affect blue pod
- Use Schedulers.boundedElastic() for the async fork
- Add timeout of 5 seconds on green pod call
- Log [DARK_LAUNCH] green OK on success
- Log [DARK_LAUNCH] green FAILED: <reason> on error
- Dark launch is controlled by AtomicBoolean enabled field
- Expose POST /internal/dark-launch/start to set enabled = true
- Expose POST /internal/dark-launch/stop to set enabled = false
- Expose GET /internal/dark-launch/status to return current state

Use Spring WebFlux, WebClient, and Project Reactor only. No blocking calls.
```

---

### Prompt 2 — Generate WebClient bean for green ingress

```
Create a Spring @Bean for WebClient configured for green pod ingress.

Requirements:
- Base URL injected from property: dark.launch.green.ingress.url
- Connection timeout: 3 seconds
- Response timeout: 5 seconds
- Bean name: greenIngressClient
- Use ReactorClientHttpConnector with HttpClient

Inject this bean into DarkLaunchFilter by constructor injection.
```

---

### Prompt 3 — Generate unit test for DarkLaunchFilter

```
Write a Spring WebFlux unit test for DarkLaunchFilter.

Test cases:
1. When enabled=true and path is /token/exchange
   → chain.filter() is called (blue always serves)
   → WebClient is called with correct body and headers
2. When enabled=false
   → chain.filter() is called
   → WebClient is never called
3. When green pod returns 500
   → chain.filter() still completes normally
   → no exception propagates
4. When green pod times out
   → chain.filter() still completes normally
   → timeout error is suppressed

Use MockWebServer to simulate green ingress.
Use StepVerifier to assert reactive flows.
```

---

### Prompt 4 — Generate application-prod.yml config

```
Generate the Spring Boot application-prod.yml config entries
for DarkLaunchFilter.

Include:
- dark.launch.green.ingress.url = https://cbx.pvt.com
- dark.launch.enabled = false   (off by default, enabled via API)
- smoke.test.scheduler.interval-ms = 60000

Add inline comments explaining each property.
```

---

---

## Approach 2 — Synthetic Token Capture + Scheduler

### Context

> Give Copilot this context first before using any prompt below.

```
I have a Spring Boot WebFlux token exchange service.
It exposes POST /token/exchange.
Request body: { "subject_token": "...", "grant_type": "...", "client_id": "..." }
Sentry runs a synthetic test every few minutes.
Synthetic test requests are identified by client_id = oidctestagent.
Blue pod is live on ingress: cbxd.prod.com
Green pod is new version, no traffic, ingress: cbx.pvt.com

I want to:
1. Capture the subject_token from Sentry synthetic test requests passively
2. Store it in memory
3. Use a scheduler to call green pod directly with that token
4. Log the result — never affect blue pod
```

---

### Prompt 1 — Generate token capture filter

```
Create a Spring WebFlux WebFilter called SyntheticTokenCaptureFilter.

Requirements:
- Intercept POST /token/exchange requests only
- Cache request body using ServerWebExchangeUtils.cacheRequestBodyAndRequest
- Deserialize body into TokenExchangeRequest using ObjectMapper
- If client_id == "oidctestagent":
    → extract subject_token
    → store in SyntheticTokenStore component
    → log [CAPTURE] Sentry synthetic subject_token captured
- Always call chain.filter() — capture is a side effect only
- Never block the request
- Suppress all errors silently — log warning only

TokenExchangeRequest fields (JSON):
  subject_token  (String)
  grant_type     (String)
  client_id      (String)
Annotate with @JsonProperty and @JsonIgnoreProperties(ignoreUnknown = true)
```

---

### Prompt 2 — Generate token store

```
Create a Spring @Component called SyntheticTokenStore.

Requirements:
- Store latest subject_token as AtomicReference<String>
- Expose store(String token) method — overwrites previous value
- Expose Optional<String> get() method
- Expose void clear() method
- Thread safe — multiple requests may call store() concurrently

Add Javadoc explaining this holds the latest Sentry synthetic test token only.
```

---

### Prompt 3 — Generate smoke test scheduler

```
Create a Spring @Component called GreenPodSmokeTestScheduler.

Requirements:
- Inject SyntheticTokenStore
- Inject WebClient for green ingress base URL from property: dark.launch.green.ingress.url
- Inject AtomicBoolean enabled to control whether scheduler runs
- Expose POST /internal/smoke-test/start → sets enabled = true
- Expose POST /internal/smoke-test/stop  → sets enabled = false, clears token store
- Expose GET /internal/smoke-test/status → returns enabled state and whether token is captured

Scheduler method:
- Annotate with @Scheduled(fixedDelayString = "${smoke.test.scheduler.interval-ms:60000}")
- If enabled = false → return immediately
- If SyntheticTokenStore is empty → log [SMOKE_TEST] waiting — no subject_token captured yet → return
- Otherwise → call green pod POST /token/exchange with captured subject_token
- On success → decode access_token claims (without signature verification) and log:
    [SMOKE_TEST] green OK | sub=<sub> | roles=<roles> | exp=<exp>
- On error → log [SMOKE_TEST] green FAILED: <reason>
- Always suppress errors — never throw
- Use timeout of 10 seconds
- Use Schedulers.boundedElastic()
- Never log raw token value
```

---

### Prompt 4 — Generate unit test for SyntheticTokenCaptureFilter

```
Write a Spring WebFlux unit test for SyntheticTokenCaptureFilter.

Test cases:
1. Request with client_id = oidctestagent
   → SyntheticTokenStore.store() is called with correct subject_token
   → chain.filter() completes normally
2. Request with client_id = some-other-client
   → SyntheticTokenStore.store() is never called
   → chain.filter() completes normally
3. Request body is malformed JSON
   → no exception propagates
   → chain.filter() completes normally
4. Non /token/exchange path
   → filter does nothing
   → chain.filter() completes normally

Mock SyntheticTokenStore with Mockito.
Use StepVerifier to assert reactive flows.
```

---

### Prompt 5 — Generate unit test for GreenPodSmokeTestScheduler

```
Write a unit test for GreenPodSmokeTestScheduler.

Test cases:
1. enabled=false → scheduler does nothing, WebClient never called
2. enabled=true, token store empty → WebClient never called, logs waiting message
3. enabled=true, token present → WebClient called with correct body
4. enabled=true, token present, green returns 500 → error suppressed, no exception
5. enabled=true, token present, green times out → timeout suppressed, no exception

Use MockWebServer for green ingress simulation.
Use Mockito for SyntheticTokenStore.
```

---

---

## Shared Prompts

### Prompt — Secure internal endpoints

```
Add security to these internal endpoints so they are not accessible publicly:

POST /internal/dark-launch/start
POST /internal/dark-launch/stop
GET  /internal/dark-launch/status
POST /internal/smoke-test/start
POST /internal/smoke-test/stop
GET  /internal/smoke-test/status

Approach:
- Add a shared secret header check: X-Internal-Secret
- Secret value injected from property: internal.api.secret
- Return 403 if header is missing or value does not match
- Apply this check via a shared method or filter — not duplicated in each endpoint
```

---

### Prompt — Add expiry check before calling green

```
In GreenPodSmokeTestScheduler, before calling green pod,
decode the captured subject_token without signature verification
and check if it is already expired.

If expired:
  → log [SMOKE_TEST] subject_token expired — waiting for next Sentry synthetic
  → do not call green pod
  → do not clear the token store (Sentry will refresh it on next synthetic run)

If valid:
  → proceed with green pod call as normal

Use JJWT to decode without verification.
Handle all parsing exceptions — log warning and skip the call.
```

---

### Prompt — Integration test for full flow

```
Write a Spring Boot @SpringBootTest integration test
that covers the full Synthetic Token Capture → Green Pod Smoke Test flow.

Setup:
- Start MockWebServer to simulate green ingress (cbx.pvt.com)
- Override dark.launch.green.ingress.url to MockWebServer URL

Test steps:
1. POST /token/exchange with client_id = oidctestagent and a fake subject_token
   → assert SyntheticTokenStore contains the captured token
2. Call GreenPodSmokeTestScheduler.run() directly (bypass scheduler timing)
   → assert MockWebServer received POST /token/exchange
   → assert request body contains correct subject_token
3. MockWebServer returns a fake access_token response
   → assert [SMOKE_TEST] green OK is logged

Use @ActiveProfiles("test") and application-test.yml for config overrides.
```

---

## Quick Reference

```
Approach              When to use
────────────────────────────────────────────────────────
Dark Launch           Real traffic forking — test green
                      under actual Sentry load pattern

Synthetic Capture     Controlled — you decide when to test
                      green. Uses real Sentry token but
                      you construct the request yourself.
```

```
Component                      Responsibility
────────────────────────────────────────────────────────
SyntheticTokenCaptureFilter    Captures token passively
SyntheticTokenStore            Holds latest token in memory
GreenPodSmokeTestScheduler     Calls green on schedule
DarkLaunchFilter               Forks live requests to green
```

```
Endpoint                        Purpose
────────────────────────────────────────────────────────
POST /internal/smoke-test/start  Start scheduled green calls
POST /internal/smoke-test/stop   Stop + clear token
GET  /internal/smoke-test/status Check state
POST /internal/dark-launch/start Enable request forking
POST /internal/dark-launch/stop  Disable request forking
GET  /internal/dark-launch/status Check state
```
