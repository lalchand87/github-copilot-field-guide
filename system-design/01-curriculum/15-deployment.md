# Curriculum · Deployment

[← System Design index](../README.md)

> 8 lessons in **Deployment**. Part of the Curriculum — Build the fundamentals.

## Contents

- **Deployment** (8): [Container images and reproducible releases](#container-images-and-reproducible-releases) · [Orchestration control loops](#orchestration-control-loops) · [Rolling deployments](#rolling-deployments) · [Blue-green deployment](#blue-green-deployment) · [Canary releases](#canary-releases) · [Feature flags](#feature-flags) · [Expand-contract schema migrations](#expand-contract-schema-migrations) · [Autoscaling control and hysteresis](#autoscaling-control-and-hysteresis)

## Deployment

### Container images and reproducible releases

*Package the application and its dependencies into an immutable, content-addressed artefact so that what was tested is exactly what runs.*

**Flow:** `Source commit` → `Build` → `Immutable image digest` → `Registry` → `Identical runtime`

> **The 30-second version**  
> Build one immutable image per commit with pinned dependencies and a pinned base, promote it by digest through every environment, and keep configuration and secrets outside it.

**The problem**

A release passes every test in staging and fails in production. The investigation eventually finds that staging had a different minor version of a system library, because both environments installed the latest available package at build time and those builds happened three weeks apart.

The same class of problem appears as a build that succeeded last month and fails today with no source changes, or a rollback that does not restore the previous behaviour because the tag it points at has been overwritten. In each case the artefact was not actually fixed, so what was tested and what ran were different things.

> **Immutability plus content addressing is what makes a release verifiable**  
> If an image is identified by the hash of its contents, then the identifier is a guarantee: the same digest is byte-identical everywhere, cannot be changed after the fact, and can be verified on pull. Mutable tags break this — a tag is a pointer that someone can move — which is why deployments should reference digests even though tags are what humans read.

**Mental model**

A build turns a source commit into an immutable artefact identified by a content digest. Everything downstream — testing, promotion, deployment, rollback — refers to that digest rather than rebuilding, so only one artefact ever exists for a given release.

1. **Build once** — One artefact per commit, produced once and promoted through environments.
2. **Layers** — Ordered filesystem deltas that are cached and shared, so builds and pulls are incremental.
3. **Digest** — The content hash identifying the image — immutable and verifiable.
4. **Tag** — A human-readable pointer to a digest, which can be moved and therefore cannot be trusted.
5. **Provenance** — The record linking an image back to its source, build and inputs.

> **Tags are mutable pointers, so deploying by tag is deploying something unspecified**  
> A tag can be reassigned to a different image at any time, so two deployments referencing the same tag can run different code — and a rollback to a previous tag may not restore what previously ran. The version label that humans use for communication is fine; what the deployment system records and references must be the digest, or the entire reproducibility argument collapses at the last step.

**How it works**

**Layer caching and why order matters**

```text
A Dockerfile produces ORDERED LAYERS.
A layer is rebuilt if it or anything before it changed.

BAD ORDER
  COPY . /app              <- application source
  RUN install-dependencies <- rebuilt on EVERY source
                              change
  -> every commit reinstalls all dependencies
  -> slow builds, and dependency versions can drift
     between builds

GOOD ORDER
  COPY dependency-manifest /app
  RUN install-dependencies  <- cached unless the
                               manifest changed
  COPY . /app               <- only this rebuilds
  -> dependency layer reused across commits
  -> builds are fast and dependencies stable

RULE
  order from least to most frequently changing

LAYER SHARING
  the dependency layer is identical across images
  -> stored once in the registry
  -> pulled once per host
  -> a 500 MB image with a 480 MB shared base
     transfers 20 MB on update

MULTI-STAGE BUILDS
  stage 1: compiler, build tools, test dependencies
  stage 2: copy ONLY the built output into a minimal
           base
  -> the runtime image excludes compilers and build
     tooling
  -> smaller, faster to pull, and a much smaller
     attack surface
```

1. **Pin every dependency version** — Installing latest makes the build a function of when it ran, which is the root of environment drift.
2. **Order layers from least to most volatile** — Dependency installation before source copying keeps the expensive layer cached across commits.
3. **Use multi-stage builds** — Shipping compilers and build tooling in the runtime image adds size and attack surface for no benefit.
4. **Build once and promote the digest** — Rebuilding per environment means the thing tested is not the thing deployed.
5. **Reference digests in deployments** — Tags are mutable pointers; only a digest guarantees what runs.
6. **Do not bake configuration or secrets into images** — An image with environment-specific configuration cannot be promoted, and a baked secret is permanent in the layer history.

**Build once, promote everywhere**

```text
WRONG: REBUILD PER ENVIRONMENT
  commit -> build -> deploy to staging
  commit -> build -> deploy to production
  -> two different builds from the same source
  -> different base image patches, different
     transitive dependency versions
  -> staging tested something production never ran

RIGHT: BUILD ONCE, PROMOTE THE DIGEST
  commit -> build -> image@sha256:abc...
                       |
                  test in staging
                       |
             promote SAME digest to production
  -> what passed tests is byte-identical to what runs

CONFIGURATION STAYS OUTSIDE
  image contains: code and dependencies only
  environment supplies: endpoints, credentials,
    feature flags, resource limits
  -> one image runs in every environment
  -> baking config in would require a rebuild per
     environment, destroying the whole property

SECRETS MUST NEVER BE BAKED
  a secret added in one layer and deleted in a later
  layer is STILL PRESENT in the image history
  -> it can be extracted by anyone who can pull it
  -> deletion in a subsequent layer hides it from the
     filesystem, not from the image
```

> **Deleting a secret in a later layer does not remove it from the image**  
> Layers are additive deltas, and the full history ships with the image. A credential copied in during a build step and removed in a subsequent step remains recoverable by anyone who can pull the image, even though it is absent from the final filesystem. Secrets must be supplied at runtime or through build-time mechanisms that do not persist into layers — and a leaked one requires rotation, not a rebuild.

**Worked example**

A build and release pipeline designed so that what ships is what was tested.

**Pipeline design**

```text
BUILD  (once per commit)
  multi-stage:
    stage 1  compile and run unit tests with full
             toolchain
    stage 2  copy the binary into a minimal runtime
             base
  all dependency versions pinned by lockfile
  base image referenced BY DIGEST, not by tag
    -> otherwise "latest stable base" changes
       underneath you
  output: registry/app@sha256:abc123...
  tagged additionally as v1.4.2 for humans

PROVENANCE RECORDED
  source commit, build id, dependency lockfile hash,
  base image digest, builder identity
  -> answers "what exactly is running, and where did
     it come from" during an incident

PROMOTION
  staging deploys app@sha256:abc123
  integration tests pass
  production deploys app@sha256:abc123
  -> same digest, no rebuild
  -> the deployment manifest records the digest;
     v1.4.2 is a label, not the reference

CONFIGURATION
  supplied by environment: endpoints, credentials,
  limits, flags
  image is environment-agnostic

ROLLBACK
  redeploy the previous digest
  -> exact, because digests are immutable
  -> a tag-based rollback would be a guess

IMAGE SIZE
  build stage ~1.2 GB with toolchain
  runtime stage ~80 MB
  -> faster pulls, smaller attack surface
```

| Metric | Value | Note |
|---|---|---|
| Build | once per commit | promoted by digest |
| Runtime image | 80 MB | **vs 1.2 GB build** |
| Reference | digest | tags are pointers |
| Rollback | previous digest | exact |

> **Pinning the base image by digest closes the last drift channel**  
> Teams pin their application dependencies carefully and then reference a base image by a tag like stable or a major version, which is republished regularly. The result is that two builds of identical source produce different images because the base changed underneath. Referencing the base by digest — and updating it deliberately as a reviewed change — makes the build genuinely a function of its inputs rather than of when it ran.

**When to use it**

- **Any deployed service**, where consistency between tested and running code matters.
- **Multi-environment pipelines**, where promotion rather than rebuilding is the correct model.
- **Rollback requirements**, which need immutable artefacts to be exact.
- **Regulated environments**, where provenance and reproducibility are auditable requirements.
- **Shared base images**, where layer reuse reduces storage and transfer substantially.

**When to avoid it**

- **Do not rebuild per environment**, which means testing one artefact and shipping another.
- **Do not deploy by mutable tag**, which leaves what runs unspecified.
- **Do not bake configuration into images**, which forces a rebuild per environment.
- **Do not put secrets in build steps**, since layer history preserves them permanently.
- **Do not install unpinned latest dependencies**, which makes builds a function of the calendar.

**Advantages**

- **What was tested is exactly what runs**, byte for byte.
- **Exact rollback** by redeploying a previous digest.
- **Layer sharing** makes storage and transfer incremental.
- **Environment-agnostic artefacts** promoted rather than rebuilt.
- **Verifiable provenance** linking a running image to its source and inputs.
- **Smaller attack surface** when multi-stage builds exclude build tooling.

**Disadvantages**

- **Image size and registry storage** require management.
- **Layer ordering must be understood** or build caching is wasted.
- **Base images need patching**, so a pinned digest is a maintenance obligation.
- **Digests are unreadable**, so tooling must present tags while referencing digests.
- **Build reproducibility is not automatic** — timestamps and ordering can still vary.

**Trade-offs**

**Image and build trade-offs**

| Choice | Benefit | Cost |
|---|---|---|
| Multi-stage build | Small runtime image; less attack surface | More complex build definition |
| Pinned base digest | Truly reproducible builds | Must be updated deliberately for patches |
| Floating base tag | Automatic patching | Builds differ over time; drift |
| Build once, promote | Tested equals deployed | Requires a promotion pipeline |
| Rebuild per environment | Simple pipeline | Different artefacts; drift between environments |
| Minimal base image | Small, secure | Fewer debugging tools available |

The base image pinning trade-off is real: a pinned digest gives reproducibility but means security patches require a deliberate update. The usual resolution is automated dependency updates that propose a new digest as a reviewed change, which keeps both properties rather than sacrificing one.

**How it fails**

**Image and release failures**

| Failure | Cause | Fix |
|---|---|---|
| Works in staging, fails in production | Rebuilt per environment | Build once; promote the digest |
| Rollback did not restore behaviour | Deployed by mutable tag | Reference digests in deployments |
| Identical source produces different images | Unpinned dependencies or floating base tag | Pin everything, including the base digest |
| Secret found in a public image | Credential added during build | Runtime secrets; rotate the leaked one |
| Builds slow on every commit | Source copied before dependency installation | Order layers least to most volatile |
| Huge images, slow deployments | Build toolchain shipped in the runtime image | Multi-stage build |
| Cannot determine what is running | No provenance recorded | Record commit, inputs and build identity |

**Limits**

> **Practical figures**
>
> - **Multi-stage reduction**: runtime images are frequently an order of magnitude smaller than build images.
> - **Layer sharing**: only changed layers transfer, so updates to a large image can be a few megabytes.
> - **Digest**: the only identifier that is immutable and verifiable; tags are pointers.
> - **Secrets**: recoverable from layer history even if deleted in a later step — rotation is the only remedy.
> - **Base image patching** requires a deliberate digest update when pinned, ideally automated as a proposed change.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Container images | General service deployment | Registry and image management |
| Machine images | VM-based infrastructure | Slower builds; larger artefacts |
| Language-native packages | Single-language environments | Runtime dependencies not captured |
| Source deployment | Simple scripting environments | Dependencies resolved at deploy time; drift |
| Nix-style reproducible builds | Strict reproducibility guarantees | Steeper learning curve; smaller ecosystem |
| Serverless bundles | Function deployment | Platform-specific; limited runtime control |

Strictly reproducible build systems go further than container images by making the entire dependency graph content-addressed, so identical inputs provably produce identical outputs. Container images approximate this well enough for most purposes provided dependencies and base images are pinned, which is where most of the practical benefit lies.

**In real systems**

- **Digest-based deployment references** are standard in mature pipelines, because tags are mutable pointers and a tag-based rollback is not guaranteed to restore anything.
- **Multi-stage builds** are the norm for compiled languages, keeping compilers and test tooling out of the runtime image.
- **Build-once-promote-many pipelines** ensure the artefact tested in staging is byte-identical to the one deployed in production.
- **Automated base image update proposals** reconcile pinned digests with the need for security patching.
- **Provenance metadata attached to images** links a running container back to its source commit, build and inputs, which is what makes incident investigation tractable.

**Common mistakes**

- **Deploying by mutable tag**, leaving what runs unspecified.
- **Rebuilding per environment**, so tested and deployed artefacts differ.
- **Unpinned dependencies or a floating base tag**, making builds time-dependent.
- **Secrets in build steps**, permanently recoverable from layer history.
- **Source copied before dependency installation**, wasting layer caching.
- **Build toolchain in the runtime image**, inflating size and attack surface.
- **Configuration baked in**, preventing promotion of a single artefact.

**The staff-level view**

Most deployment reproducibility problems come down to something being resolved at the wrong time.

- **Require digest references in deployment manifests.** A tag-based deployment leaves what runs unspecified and makes rollback a hope rather than a guarantee, and this is the single highest-value change in most pipelines.
- **Build once and promote.** Rebuilding per environment means staging validated an artefact production never ran, which invalidates the testing that justified the release.
- **Pin the base image by digest.** Teams pin application dependencies carefully and then leave the base floating, which reintroduces exactly the drift they eliminated everywhere else.
- **Keep configuration and secrets out of images.** Configuration inside the image forces a rebuild per environment; a secret in layer history is permanently recoverable and requires rotation rather than a rebuild.
- **Record provenance.** During an incident, being able to establish precisely what is running and what went into it turns a long investigation into a short one.

**Go deeper**

A container image packages an application and its dependencies into an artefact identified by the hash of its contents. That digest is immutable and verifiable, so the same digest deployed anywhere is byte-identical and a rollback to a previous digest restores exactly what ran before. Tags are mutable pointers and cannot provide that guarantee, which is why deployment manifests should reference digests even though humans read version tags.

The artefact must be built once and promoted rather than rebuilt per environment, because two builds from the same commit can differ — a patched base image, a different transitive dependency, a toolchain update — meaning staging validated something production never ran. Pinning application dependencies is standard practice; pinning the base image by digest is the step most often omitted, and it reopens exactly the drift channel everything else was closing.

Two build details matter operationally. Layer ordering from least to most volatile keeps dependency installation cached across commits, and multi-stage builds keep compilers and test tooling out of the runtime image, typically reducing it by an order of magnitude. Configuration and secrets belong outside the image: configuration inside forces a rebuild per environment, and a secret added during a build remains recoverable from layer history even after a later step deletes it.

Container images make a release a fixed, verifiable object rather than the result of a process that may produce something different next time.

**Content addressing is the whole guarantee.** A digest names exactly one set of bytes, so it cannot drift, can be verified on pull, and means the same thing in every environment and at every future point in time. This is what allows a rollback to be exact rather than approximate — the previous digest still exists and still denotes what previously ran. Tags are a separate and useful concept for human communication, but because they are reassignable pointers, a deployment recorded by tag has not actually specified what it deployed.

**Build once and promote, because rebuilding produces a different object.** Two builds from an identical commit at different times can differ through a patched base image, a newly published transitive dependency, or a toolchain update. Rebuilding per environment therefore means the artefact validated in staging is not the artefact serving production, which silently invalidates the testing used to justify the release. Promotion of a single digest closes that gap entirely, and it is the property that makes a pipeline's test results meaningful.

**Pinning must extend to the base image.** Teams routinely pin application dependencies with lockfiles and then reference a base image by a tag such as a major version or stable, which is republished regularly for security patching. The result is that identical source produces different images over time and the carefully constructed reproducibility has one open channel. Pinning the base by digest closes it, at the cost of making patch adoption a deliberate act — best reconciled by automating update proposals so patching remains routine without sacrificing the guarantee.

**Layer structure determines both build speed and image size.** Layers are ordered deltas rebuilt whenever they or anything preceding them changes, so placing dependency installation before source copying keeps the expensive layer cached across commits, while the reverse order reinstalls everything on every change and permits versions to drift between builds. Multi-stage builds address the other dimension: compiling in a stage with full toolchain and copying only the output into a minimal runtime base typically reduces the shipped image by an order of magnitude and removes compilers, package managers and test tooling from the production attack surface.

**Configuration and secrets must live outside the artefact.** Baking environment-specific configuration in means a separate image per environment, which eliminates promotion and returns you to rebuilding. Secrets are worse: layers are additive and the full history ships with the image, so a credential introduced during a build step and deleted in a later one remains recoverable by anyone able to pull the image. The deletion affects the running filesystem, not the artefact, and the only remedy once it has happened is rotating the credential, since the image may already have been pulled anywhere.

**Provenance turns an artefact into evidence.** Recording the source commit, dependency lockfile hash, base image digest and builder identity alongside the image answers, during an incident, exactly what is running and what went into it. Without it, establishing that basic fact can consume a significant share of the investigation. With it, a production failure is immediately attributable to code or environment rather than to an unresolved question about whether the build itself diverged — which is precisely the category of uncertainty that reproducibility exists to remove.

**Prove it — interview questions**

1. **[Basic] Why are container images immutable?**

   <details><summary>Model answer</summary>

   Because they are identified by the hash of their contents, so a digest names exactly one set of bytes and cannot be changed afterwards. That is what makes a deployment verifiable: the same digest deployed anywhere is byte-identical, and a rollback to a previous digest restores exactly what ran before. Tags are a separate concept — human-readable pointers that can be reassigned — which is why they are useful for communication and unsuitable as the reference a deployment system records.

   </details>

2. **[Basic] Why does layer ordering matter?**

   <details><summary>Model answer</summary>

   Because a layer is rebuilt whenever it or anything before it changes, and the cache is reused otherwise. If source code is copied before dependencies are installed, then every commit invalidates the dependency layer and reinstalls everything — which is slow and, worse, allows dependency versions to drift between builds. Ordering from least to most frequently changing, so dependency installation precedes copying source, keeps the expensive layer cached and stable across commits.

   </details>

3. **[Senior] Why build once and promote rather than rebuilding per environment?**

   <details><summary>Model answer</summary>

   Because rebuilding produces a different artefact. Two builds from the same commit at different times can pick up a patched base image, a different transitive dependency version, or a different toolchain patch — so staging validated something production never ran, which invalidates the testing that justified the release. Building once and promoting the same digest through environments means what passed integration tests is byte-identical to what serves traffic. It also makes rollback exact, since the previous digest still exists and still means the same thing.

   </details>

4. **[Senior] Why can't you remove a secret by deleting it in a later layer?**

   <details><summary>Model answer</summary>

   Because layers are additive deltas and the complete history ships with the image. A credential copied in during one build step and deleted in the next is absent from the final filesystem but fully recoverable by anyone who can pull the image and inspect its layers. The deletion hides it from the running container, not from the artefact. The correct handling is to supply secrets at runtime, or use build mechanisms that do not persist into layers — and once one has been baked in, the remedy is rotating the credential, because the image cannot be un-published from wherever it has already been pulled.

   </details>

5. **[Staff] Design a build and release pipeline for a compiled service.**

   <details><summary>Model answer</summary>

   A multi-stage build: the first stage carries the full toolchain to compile and run unit tests, the second copies only the resulting binary into a minimal runtime base — typically an order of magnitude smaller and without compilers or test tooling in the production attack surface. Every dependency pinned by lockfile, and critically the base image referenced by digest rather than by a tag like stable, because that tag is republished and is the drift channel teams most often leave open after carefully pinning everything else. The build runs once per commit and produces one digest, with a version tag applied alongside purely for human communication. Provenance recorded with the image: source commit, lockfile hash, base digest, builder identity — which is what makes it possible during an incident to establish precisely what is running. That digest is then promoted: staging deploys it, integration tests run against it, production deploys the same digest with no rebuild. Configuration and secrets come from the environment so a single artefact is environment-agnostic. Rollback is redeploying the previous digest, which is exact. And I would pair the pinned base with automated update proposals, so security patching remains routine without giving up reproducibility.

   </details>

6. **[Principal] What does reproducibility actually buy, given the effort?**

   <details><summary>Model answer</summary>

   It buys the ability to reason about production at all. Without it, the chain from a passing test to a running service has a gap in it — the artefact that passed and the artefact that runs are different objects related only by having been built from the same source, which means every environment-specific failure requires an investigation to determine whether the code or the build diverged. With it, that entire category of question disappears: the digest running in production is the digest that passed, so a production failure is a property of the code or the environment, never of the build. The second thing it buys is trustworthy rollback, which matters more than it sounds because rollback is the primary mitigation for most deployment incidents — and a rollback that might not restore the previous behaviour is not a mitigation, it is another change under time pressure. The effort is mostly front-loaded and mostly one-off: pin dependencies, pin the base by digest, build once, reference digests, keep configuration out. What makes it worth insisting on is that each of those is cheap individually, and omitting any one of them quietly reintroduces the whole problem — which is why the reproducibility argument tends to fail at its weakest link rather than degrade gracefully.

   </details>

---

### Orchestration control loops

*Declare the desired state and let a controller continuously reconcile reality toward it, so the system self-corrects rather than executing one-off instructions.*

**Flow:** `Desired state` → `Observed state` → `Diff` → `Corrective action` → `Continuous reconciliation`

> **The 30-second version**  
> Declare the desired state and run a loop that observes reality, computes the difference and corrects it continuously — level-triggered, idempotent, rate-limited, and bounded in how much it changes at once.

**The problem**

A deployment script starts five instances. Later one crashes, another is terminated when its host is reclaimed, and a third is left running from a previous release that the script never cleaned up. The script ran successfully and the system is now wrong, because a script describes a sequence of actions at one moment rather than a condition to be maintained.

The imperative approach also fails to recover. Nothing re-examines the system, so drift accumulates: manual fixes applied during an incident, resources created and never removed, configuration changed by hand and forgotten.

> **Declare the outcome and reconcile continuously, rather than issuing instructions**  
> A control loop observes actual state, compares it with the declared desired state, and takes corrective action — then repeats, forever. Failures become transient deviations that the loop corrects automatically, and manual drift is undone rather than accumulated. The shift is from describing how to reach a state to describing the state itself and letting a controller maintain it.

**Mental model**

A controller runs a loop: read the desired state, observe the actual state, compute the difference, act to reduce it, repeat. Nothing is assumed to have worked, and nothing is done only once.

1. **Desired state** — A declaration of what should be true, stored durably.
2. **Observed state** — What is actually true right now, read fresh each iteration.
3. **Diff** — The difference between the two — the work to be done.
4. **Action** — A step toward reducing the difference, not necessarily completing it.
5. **Repeat** — The loop runs continuously, so failures and drift are corrected without intervention.

> **A controller acting on stale observations fights itself**  
> If a controller decides from cached state rather than fresh observation, it may create resources that already exist or delete ones that were just created. The classic symptom is oscillation — the controller repeatedly creating and destroying the same thing — and it is why each iteration must observe reality rather than assume its previous action succeeded.

**How it works**

**The reconciliation loop**

```text
loop forever:
    desired  = read declared state
    observed = read ACTUAL state       <- fresh, always
    diff     = desired - observed
    if diff is empty:
        wait and continue
    take ONE step toward reducing diff
    (do not assume it worked)

WHY IT SELF-HEALS
  a pod crashes
    -> next iteration observes 4 instead of 5
    -> creates one
  someone manually deletes something
    -> next iteration restores it
  a host is reclaimed
    -> next iteration reschedules elsewhere
  -> no human, no alert, no script re-run

WHY IDEMPOTENCE IS REQUIRED
  the loop runs constantly and may act on a stale
  view
  -> "ensure 5 exist" is safe to evaluate repeatedly
  -> "create one more" is not

LEVEL-TRIGGERED, NOT EDGE-TRIGGERED
  edge-triggered: react to the EVENT of a change
    -> a missed event means permanent divergence
  level-triggered: react to the STATE
    -> a missed event is corrected on the next pass
  -> this is why controllers reconcile on a timer as
     well as on events
```

1. **Read fresh state every iteration** — Acting on cached observations causes duplicate creation and oscillation.
2. **Make every action idempotent** — The loop will retry, so operations must be safe to repeat.
3. **Reconcile on a timer as well as on events** — Events can be missed; a periodic pass makes the system self-correcting regardless.
4. **Take small steps rather than jumping to the target** — Replacing everything at once is a much larger failure if the new state is broken.
5. **Record status separately from desired state** — Conflating what should be true with what is true makes both unreliable.
6. **Bound the rate of corrective action** — An unconstrained controller responding to a widespread failure can amplify it.

**Level-triggered versus edge-triggered**

```text
EDGE-TRIGGERED (event-driven only)
  "a pod was deleted" -> create a replacement
  if the event is missed - controller restart, network
  blip, queue overflow
    -> the replacement is never created
    -> divergence is PERMANENT until something else
       notices

LEVEL-TRIGGERED (state-driven)
  "5 desired, 4 observed" -> create one
  the event does not matter; the STATE does
    -> a missed event is corrected on the next pass
    -> the system converges from ANY starting point

WHY THIS MATTERS ARCHITECTURALLY
  level-triggered systems have no concept of a lost
  update
  -> restart the controller with no memory: it
     reconciles correctly
  -> that is why controllers can be stateless and
     restartable

EVENTS AS AN OPTIMISATION
  events make reconciliation FASTER, not correct
  the timer makes it CORRECT
  -> a controller that only reacts to events is one
     missed message from permanent drift
```

> **Controllers can amplify failures if their actions are unbounded**  
> A controller observing that many instances are unhealthy may attempt to replace all of them simultaneously, which can overwhelm the scheduler, the registry, or the dependency that made them unhealthy in the first place. Rate limits, maximum unavailable constraints and backoff on repeated failure are what prevent a self-healing mechanism from turning a partial failure into a total one.

**Worked example**

A deployment controller maintaining a running service.

**Reconciliation in practice**

```text
DESIRED STATE  (declared)
  replicas: 5
  image: app@sha256:abc123
  resources: 1 CPU, 2 GB
  maxUnavailable: 1

CONTROLLER LOOP  (every few seconds, plus on events)
  observe: how many pods exist, at which image,
           in which state
  diff:    what differs from desired
  act:     one step toward convergence

SCENARIOS

pod crashes
  observed 4, desired 5 -> create one
  -> recovered in seconds, no human involved

node fails, taking 2 pods
  observed 3, desired 5 -> create two
  -> but rate-limited, so the scheduler is not
     overwhelmed

image updated to abc456
  desired image differs from observed
  -> replace pods one at a time (maxUnavailable: 1)
  -> NOT all at once: a broken image would then take
     the whole service down
  -> the constraint is what makes rollout safe

someone manually deletes a pod
  observed 4 -> create one
  -> manual drift undone automatically

new image crash-loops
  observed pods unhealthy after replacement
  -> rollout halts at maxUnavailable rather than
     continuing
  -> 4 of 5 still serving on the old image
  -> this is the single most valuable property of the
     constraint

STATUS IS SEPARATE
  desired state: what the operator declared
  status: what the controller observes
  -> never conflated, or neither can be trusted
```

| Metric | Value | Note |
|---|---|---|
| Loop | seconds | continuous |
| Crash | auto-replaced | no human |
| Rollout | maxUnavailable 1 | **broken image contained** |
| Drift | undone | manual changes reverted |

> **The rollout constraint is what makes automated deployment safe**  
> A controller replacing all instances simultaneously would deploy a broken image everywhere before anything noticed. Constraining how many can be unavailable at once means a bad release stops after the first replacement fails its health check, leaving the majority still serving the previous version. The self-healing property gets the attention, but this containment is what makes continuous deployment survivable.

**When to use it**

- **Container orchestration**, where the model originated and fits naturally.
- **Infrastructure as code**, where declared resources should be continuously reconciled.
- **Configuration management**, to undo drift rather than accumulate it.
- **Any long-lived system state** that should be maintained rather than merely established.
- **Self-healing requirements**, where recovery should not require human action.

**When to avoid it**

- **Do not use control loops for one-off operations**, such as a data migration that must run exactly once.
- **Do not reconcile state a human must approve**, where automatic correction is inappropriate.
- **Do not build controllers with non-idempotent actions**, which the retry behaviour will expose.
- **Do not rely on events alone**, since a missed event becomes permanent divergence.
- **Do not let controllers act without rate limits**, which can amplify a partial failure.

**Advantages**

- **Self-healing**, correcting failures without human intervention.
- **Drift is undone** rather than accumulating.
- **Converges from any starting state**, since the loop is level-triggered.
- **Controllers can be stateless and restartable**, because state lives in the declaration and in observation.
- **Declarative intent** is readable, reviewable and version-controlled.
- **Rollout constraints contain bad releases** automatically.

**Disadvantages**

- **Debugging is indirect**, since the controller acts rather than a person.
- **Fighting the controller is futile** — manual changes are reverted, which surprises people.
- **Unbounded action can amplify failures** without rate limits.
- **Not suited to one-off operations**, which need different machinery.
- **Convergence can be slow** for large changes taken in small steps.
- **Requires idempotent actions throughout**, which constrains implementation.

**Trade-offs**

**Reconciliation design trade-offs**

| Choice | Benefit | Cost |
|---|---|---|
| Level-triggered | Converges from any state; missed events harmless | Requires periodic full observation |
| Edge-triggered only | Fast; low overhead | A missed event is permanent divergence |
| Frequent reconciliation | Fast correction | Load on the API and observed systems |
| Small steps | Contains bad changes | Slow convergence for large changes |
| Aggressive replacement | Fast rollout | A broken release reaches everything |

The step size choice is the one with real consequences: replacing everything at once completes a rollout quickly and also deploys a broken image everywhere before health checks report anything, whereas a bounded unavailable count stops the rollout after the first failure with the majority still serving.

**How it fails**

**Control loop failures**

| Failure | Cause | Fix |
|---|---|---|
| Resources created repeatedly | Acting on stale observations | Read fresh state each iteration |
| Divergence persists after a blip | Event-driven only | Reconcile on a timer as well |
| Bad release deployed everywhere | No unavailability constraint | Bound how many can be replaced at once |
| Partial failure amplified | Controller replacing everything at once | Rate limits and backoff |
| Manual fix keeps reverting | Controller reconciling to the declared state | Change the declaration, not the running state |
| Duplicate side effects | Non-idempotent controller actions | Make every action safe to repeat |
| Cannot tell intent from reality | Status conflated with desired state | Store them separately |

**Limits**

> **Operational parameters**
>
> - **Reconciliation interval**: seconds to a minute, balancing correction speed against load on observed systems.
> - **Unavailability constraint**: typically one or a small fraction, which is what contains a bad rollout.
> - **Idempotence**: required throughout, since every action may be repeated.
> - **Rate limiting**: necessary, or a widespread failure triggers a widespread corrective storm.
> - **Events are an optimisation**; the timer is what provides correctness.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Declarative control loops | Long-lived state maintenance | Indirect debugging; needs idempotence |
| Imperative scripts | One-off operations | No self-healing; drift accumulates |
| Configuration management runs | Periodic convergence | Between runs, drift persists |
| Manual operation | Very small systems | Does not scale; error-prone |
| Event-driven automation | Fast reaction | Missed events cause permanent divergence |
| GitOps | Declared state in version control | Requires reconciliation from the repository |

GitOps applies the same loop with the desired state stored in version control, which adds review, history and auditability to the declaration while keeping the reconciliation model identical — a natural extension rather than a competing approach.

**In real systems**

- **Kubernetes controllers** are the canonical implementation, reconciling observed cluster state toward declared specifications continuously.
- **Level-triggered design** is what allows controllers to be restarted with no memory and still converge correctly.
- **Maximum-unavailable constraints on rollouts** are what stop a broken image from reaching every instance before health checks report.
- **GitOps tooling** reconciles a cluster against a version-controlled declaration, adding review and audit to the same loop.
- **Rate limiting and backoff in controllers** prevent a large-scale failure from producing a corrective storm that worsens it.

**Common mistakes**

- **Event-driven controllers with no periodic reconciliation**, leaving missed events as permanent drift.
- **Acting on cached state**, causing duplicate creation and oscillation.
- **Non-idempotent actions**, exposed by the loop's retry behaviour.
- **No unavailability constraint**, deploying a broken release everywhere.
- **Unbounded corrective action**, amplifying partial failures.
- **Manual fixes to running state**, which the controller reverts.
- **Status conflated with desired state**, making both unreliable.

**The staff-level view**

The control loop model is powerful and its failure modes are all about what happens when the loop acts too freely or on bad information.

- **Insist on level-triggered reconciliation.** An event-driven controller is one dropped message away from permanent divergence, and the timer is what makes correctness independent of message delivery.
- **Require rollout constraints.** The self-healing property gets the attention, but bounding how many instances can be replaced at once is what stops a broken release from reaching all of them before anything notices.
- **Rate-limit corrective action.** A controller responding to a widespread failure by replacing everything simultaneously can overwhelm the scheduler or the dependency that caused the failure, converting partial into total.
- **Teach that manual changes are reverted.** People fighting a controller during an incident is a recurring and avoidable source of confusion; the declaration is what must change.
- **Keep desired state and status separate.** Conflating intent with observation makes it impossible to tell whether something is broken or simply not yet converged.

**Go deeper**

A control loop reads a declared desired state, observes what is actually true, and acts to reduce the difference — repeatedly and forever. Failures become transient deviations corrected automatically, and manual drift is reverted rather than accumulated. This replaces the imperative model where a script establishes a state once and nothing subsequently re-examines it.

Level-triggered design is what makes it robust: reacting to state rather than to change events means a missed event is corrected on the next pass, so the system converges from any starting point and controllers can be stateless and restartable. Events remain useful as an optimisation for speed, but the periodic reconciliation is what provides correctness, and a controller relying on events alone is one dropped message from permanent divergence.

The failure modes come from acting too freely. A controller replacing every instance at once deploys a broken image everywhere before health checks report, so bounding how many may be unavailable is what makes automated rollout safe. Similarly, responding to a widespread failure by replacing everything simultaneously can overwhelm the scheduler or the dependency that caused it, which is why rate limits and backoff matter as much as the reconciliation itself.

Orchestration control loops maintain system state by continuously reconciling observed reality toward a declared intent, rather than by executing instructions that are correct only at the moment they run.

**Divergence is treated as normal.** An imperative model assumes that a successful action leaves the system correct, so every later deviation is an anomaly requiring detection and a specific response. A reconciliation loop assumes reality is always somewhat wrong and that correction is the ongoing job — so a crashed process, a reclaimed host, a manual deletion and an incomplete rollout all reduce to the same operation of observing a difference and acting on it. That collapse of many failure cases into one mechanism is the model's central benefit.

**Level-triggered beats edge-triggered for correctness.** Reacting to the event of a change means a missed event produces permanent divergence, and events are missed for ordinary reasons — controller restarts, network interruptions, queue overflow. Reacting to observed state means the event is irrelevant: five desired against four observed produces the same action regardless of whether any notification arrived. This is also what allows controllers to hold no durable memory and be restarted freely, since everything needed is in the declaration and in fresh observation.

**Fresh observation and idempotence are prerequisites.** A controller deciding from cached state may create what already exists or delete what was just created, producing oscillation that looks like a bug in the resource rather than in the loop. And because the loop retries continually and may act on an imperfect view, every action must be safe to repeat — expressing operations as ensure-this-is-true rather than do-this-once is what makes repeated evaluation harmless.

**Bounding the step is what makes rollout safe.** A controller free to replace every instance simultaneously will deploy a broken image everywhere before any health check has had time to report, converting a bad release into a total outage. Constraining how many instances may be unavailable at once means the rollout halts after the first replacement fails, with the majority still serving the previous version. The self-healing behaviour attracts the attention, but this containment is what makes continuous automated deployment survivable.

**Unbounded correction amplifies failure.** When many instances become unhealthy simultaneously — a node failure, a dependency outage — a controller responding at full speed can overwhelm the scheduler, the image registry, or the very dependency that caused the problem. Rate limiting and backoff on repeated failure are therefore not refinements but requirements, because the mechanism designed to recover from failure is otherwise capable of deepening it precisely when the system is least able to absorb additional load.

**The human consequences deserve explicit attention.** Debugging becomes indirect, since actions are taken by a controller rather than a person, and diagnosing behaviour means reading the loop's reasoning rather than a command history. More disruptively, people attempting to fix something by modifying running state find their change reverted within seconds, which during an incident is a recurring source of confusion and wasted time. Both are manageable once understood — the declaration is what must change — but they are genuine costs, and teams adopting the model benefit from being told about the second one before they encounter it under pressure.

**Prove it — interview questions**

1. **[Basic] What is a reconciliation loop?**

   <details><summary>Model answer</summary>

   A controller that continuously reads the declared desired state, observes actual state, computes the difference, and takes a step to reduce it — then repeats forever. Nothing is assumed to have worked and nothing is done only once, which is what makes the system self-healing: a crashed instance, a reclaimed host or a manual deletion all appear as a difference on the next iteration and are corrected without anyone doing anything.

   </details>

2. **[Basic] Why is declarative better than imperative here?**

   <details><summary>Model answer</summary>

   Because a script describes actions at a moment while a declaration describes a condition to be maintained. The script runs, succeeds, and then reality diverges — instances crash, hosts are reclaimed, someone changes something by hand — and nothing re-examines the system, so drift accumulates. A declaration plus a controller means the same divergences are simply differences to be reconciled, and the system converges continuously rather than being correct only immediately after a deployment.

   </details>

3. **[Senior] What is the difference between level-triggered and edge-triggered?**

   <details><summary>Model answer</summary>

   Edge-triggered reacts to the event of a change: a pod was deleted, so create a replacement. If that event is missed — a controller restart, a network issue, a full queue — the replacement is never created and the divergence is permanent until something else notices. Level-triggered reacts to the state: five desired, four observed, create one. The event is irrelevant, so a missed message is corrected on the next pass and the system converges from any starting point. This is also why controllers can be stateless and restartable, since they hold no memory that a restart would lose.

   </details>

4. **[Senior] Why do controllers need rate limits and rollout constraints?**

   <details><summary>Model answer</summary>

   Because the same mechanism that heals failures can amplify them. A controller observing that many instances are unhealthy may try to replace all of them at once, overwhelming the scheduler, the image registry, or the dependency that made them unhealthy to begin with — turning a partial failure into a total one. Separately, an unconstrained rollout replaces every instance with a new image before any health check has reported, so a broken release reaches everything. Bounding how many can be unavailable simultaneously means the rollout halts after the first failure with the majority still serving the previous version.

   </details>

5. **[Staff] Design the deployment behaviour you would want from an orchestrator.**

   <details><summary>Model answer</summary>

   Level-triggered reconciliation on a short timer, with events as an optimisation rather than the correctness mechanism — because a controller that only reacts to events is one dropped message from permanent drift, and the timer is what makes it converge from any state including after its own restart. Fresh observation every iteration rather than cached state, since acting on a stale view produces duplicate creation and oscillation. All actions idempotent, because the loop will retry. For rollouts, a strict constraint on how many instances may be unavailable at once: this is what I would consider the single most valuable setting, because it means a crash-looping new image halts the rollout after one replacement with the rest still serving the old version, instead of the controller cheerfully replacing everything before health checks report. Rate limiting on corrective action so a node failure taking many instances does not produce a replacement storm against an already-stressed scheduler. And desired state kept strictly separate from observed status, so it is always possible to distinguish something that is broken from something that has simply not converged yet.

   </details>

6. **[Principal] What is the conceptual shift that makes this model work?**

   <details><summary>Model answer</summary>

   Treating divergence as normal rather than exceptional. An imperative model implicitly assumes that once an action succeeds the state is correct, so every subsequent deviation is an anomaly requiring detection and a response. A control loop assumes reality is always somewhat wrong and that the system's job is continuous correction, which means failures stop being events that need handling and become differences that get reconciled — a crashed process, a reclaimed host, a manual change and a partial deployment all reduce to the same operation. That reframing is what produces the system's most useful properties: controllers can be stateless because nothing needs remembering, they can be restarted freely because they converge from any state, and missed messages are harmless because correctness comes from observation rather than from delivery. The cost is that debugging becomes indirect — actions are taken by a controller rather than a person — and that people who try to fix something by changing running state find their change reverted, which is a genuinely common source of confusion during incidents. Both of those are manageable with understanding, and neither outweighs the elimination of an entire category of failure handling.

   </details>

---

### Rolling deployments

*Replace instances gradually while keeping the service available, accepting that two versions run simultaneously and must therefore be compatible.*

**Flow:** `Old version serving` → `Batch replaced` → `Health gate` → `Progressive rollout` → `Fully updated`

> **The 30-second version**  
> Replace instances in small batches behind a health gate that reflects real serving ability, allow surge so capacity holds, and make every change compatible with the version running alongside it.

**The problem**

Deploying by stopping every instance and starting the new version means downtime proportional to startup time, and if the new version is broken the service is down until someone notices and reverts. For anything user-facing this is unacceptable, and for anything deployed frequently it is unacceptable several times a week.

The obvious fix — replace instances gradually — introduces a subtler problem: for the duration of the rollout, two versions are running at once, handling the same traffic, reading and writing the same database. Everything they share must work for both.

> **Gradual replacement buys availability and costs compatibility**  
> Rolling instances one batch at a time keeps the service up and limits the blast radius of a bad release to the instances replaced so far. The price is that old and new versions coexist, so any change to a database schema, a message format, or an API contract must be compatible in both directions for the duration — which is a design constraint on the change itself, not on the deployment mechanism.

**Mental model**

Instances are replaced in batches. Each batch must become healthy before the next begins, so a broken release stops the rollout rather than completing it. Throughout, both versions serve traffic.

1. **Batch size** — How many instances are replaced at once — the blast radius per step.
2. **Health gate** — The check that must pass before proceeding, which is what makes the rollout self-limiting.
3. **Surge** — Extra capacity allowed during the rollout, so replacement does not reduce serving capacity.
4. **Coexistence** — The window during which both versions run and must interoperate.
5. **Rollback** — Rolling back to the previous version, which is subject to the same constraints in reverse.

> **The health gate is the only thing standing between a bad release and everything**  
> If instances are marked healthy before they can actually serve — because readiness is not checked, or checks only whether the process started — then the rollout proceeds through every batch and replaces the entire fleet with a broken version. The gradual mechanism provides no protection at all without a health check that genuinely reflects the ability to serve.

**How it works**

**The rollout and what bounds it**

```text
PARAMETERS
  maxUnavailable  how much capacity may be lost
  maxSurge        how much extra capacity is allowed
  health gate     what must pass before proceeding

10 INSTANCES, maxUnavailable=1, maxSurge=1
  start 1 new  (11 running: 10 old + 1 new)
  new becomes READY
  stop 1 old   (10 running: 9 old + 1 new)
  repeat
  -> capacity never drops below 10
  -> at most 1 instance in transition

IF THE NEW VERSION IS BROKEN
  new instance never becomes ready
  -> rollout HALTS
  -> 9 old instances still serving
  -> impact: ~10% capacity in a failed state
  -> and an alert, rather than a full outage

maxUnavailable=0 REQUIRES SURGE
  you cannot replace without first adding
  -> needs spare capacity headroom
  -> the safest configuration, at the cost of
     temporarily running extra instances

SPEED VERSUS SAFETY
  batch of 1   slowest, smallest blast radius
  batch of 25% fast, but a quarter broken before the
               gate catches it
  -> the health gate delay is the real constraint:
     if a fault takes 60 s to manifest, a 10 s gate
     passes broken instances through
```

1. **Gate on readiness that reflects the ability to serve** — A gate that only checks process start lets the rollout replace everything with a broken version.
2. **Wait long enough for faults to manifest** — Crash loops appear in seconds; memory leaks and slow degradation do not, so a short gate passes them.
3. **Keep batches small for risky changes** — Batch size is blast radius, and the cost of a slower rollout is almost always lower than the cost of a wide one.
4. **Ensure both versions can coexist** — Schema, message format and API changes must work for old and new simultaneously, or the rollout itself breaks things.
5. **Drain connections before stopping instances** — Failing readiness first and waiting for the load balancer prevents errors on every deployment.
6. **Verify that rollback is possible** — A change that is forward-compatible only cannot be rolled back, which removes the primary mitigation.

**The coexistence constraint**

```text
DURING A ROLLING DEPLOY, BOTH VERSIONS RUN.
Anything shared must work for both.

DATABASE SCHEMA
  BROKEN: new version renames a column
    -> old instances query the old name -> errors
    -> and rollback is impossible: the new version
       needs the new name
  SAFE: add the new column, write both, migrate,
    then remove the old one in a LATER release
    -> each step compatible with the version before
       and after it

MESSAGE FORMATS
  BROKEN: new version publishes a format old consumers
    cannot parse
  SAFE: additive changes only; consumers ignore
    unknown fields

API CONTRACTS
  BROKEN: new version requires a field the old client
    does not send
  SAFE: new field optional until all callers send it

CACHED DATA
  BROKEN: versions write incompatible structures to a
    shared cache
  SAFE: version the cache key, so each writes its own

THE RULE
  every change must be compatible with the version
  immediately before it AND after it
  -> because during rollout and rollback, both run
```

> **A change that cannot be rolled back removes your main mitigation**  
> Rollback is the first response to most deployment incidents, and it only works if the previous version can still function against current state. A migration that drops a column, a message format that old consumers cannot read, or a cache structure the old code cannot parse all make rollback impossible — so the only path is forward, under time pressure, which is precisely the situation the rollback was meant to avoid.

**Worked example**

Rolling out a release that also changes the database schema.

**Deployment plan**

```text
SERVICE  20 instances behind a load balancer

ROLLOUT SETTINGS
  maxSurge: 2         (allow 22 briefly)
  maxUnavailable: 0   (never drop below 20)
  readiness gate: must serve successfully for 30 s
    -> long enough for startup faults and immediate
       crash loops to appear
  progress deadline: 10 min -> abort and alert

SCHEMA CHANGE, DONE SAFELY  (three releases)
  release 1  add the new column, nullable
             code writes BOTH old and new
             -> old instances unaffected
  release 2  backfill; code reads new, still writes
             both
             -> rollback to release 1 still works
  release 3  stop writing old; drop it later
             -> only once release 2 is everywhere and
                stable

WHY NOT ONE RELEASE
  renaming in a single deploy means old instances
  query a column that no longer exists
  -> errors during the entire rollout window
  -> and rollback is impossible

SHUTDOWN SEQUENCE PER INSTANCE
  fail readiness -> wait for the balancer -> stop
  accepting -> drain in-flight -> exit
  -> skipping the wait causes errors on every deploy

FAILURE BEHAVIOUR
  broken release: first new instance never passes the
  30 s gate -> rollout halts with 20 old instances
  still serving -> zero user impact, one alert
```

| Metric | Value | Note |
|---|---|---|
| Surge | +2, unavailable 0 | capacity maintained |
| Gate | 30 s serving | faults manifest |
| Schema | 3 releases | **rollback preserved** |
| Bad release | halts at 1 instance | no user impact |

> **Schema changes should span three releases, not one**  
> Adding a column, writing both old and new, backfilling, then switching reads, and only later removing the old column, means every intermediate state is compatible with the version before and after it. A single-release rename breaks old instances during the rollout and makes rollback impossible. The three-release discipline feels slow until the first time a release has to be reverted, at which point it is the only thing that makes reverting possible.

**When to use it**

- **Stateless services behind a load balancer**, which is the natural fit.
- **Frequent deployments**, where downtime per release would accumulate unacceptably.
- **When extra capacity is available**, allowing surge so capacity never drops.
- **Changes that are backward and forward compatible**, which is the precondition.
- **As the default deployment strategy**, with more elaborate approaches reserved for higher-risk changes.

**When to avoid it**

- **Do not roll out changes that cannot coexist**, such as an incompatible schema rename in one release.
- **Do not deploy with a health gate that does not reflect serving ability**, which removes all protection.
- **Do not use short gates for slow-manifesting faults**, which pass broken instances through.
- **Do not roll deploy stateful systems** without explicit handling of data and membership.
- **Do not skip connection draining**, which produces errors on every deployment.

**Advantages**

- **No downtime** when surge capacity is available.
- **Bounded blast radius**, since only a batch is affected at a time.
- **Self-limiting**, because a failing health gate halts the rollout.
- **No duplicate infrastructure required**, unlike parallel-environment strategies.
- **Gradual and observable**, so problems appear before the fleet is converted.
- **Straightforward rollback** when changes are compatible in both directions.

**Disadvantages**

- **Two versions run simultaneously**, constraining every shared contract.
- **Slow for large fleets** when batches are small.
- **Rollback is also gradual**, so recovery is not instant.
- **Requires spare capacity** for surge, or capacity dips during rollout.
- **Health gate quality determines safety entirely.**
- **Poor fit for stateful systems** without additional handling.

**Trade-offs**

**Rollout configuration trade-offs**

| Setting | Faster | Safer | Notes |
|---|---|---|---|
| Batch size 1 | No | Yes | Smallest blast radius |
| Batch size 25% | Yes | No | A quarter affected before the gate reacts |
| maxUnavailable 0 with surge | Neutral | Yes | Requires spare capacity |
| Short health gate | Yes | No | Slow faults pass through |
| Long health gate | No | Yes | Rollout takes longer |
| Progress deadline | — | Yes | Aborts a stuck rollout rather than hanging |

The health gate duration is the setting most often left too short. Crash loops appear in seconds, but memory leaks, connection pool exhaustion and slow degradation take minutes — so a gate that waits only for the process to respond will happily promote instances that fail later, converting a contained problem into a fleet-wide one.

**How it fails**

**Rolling deployment failures**

| Failure | Cause | Fix |
|---|---|---|
| Entire fleet replaced with a broken version | Health gate does not reflect serving ability | Readiness that exercises real functionality |
| Errors during every deployment | Instances stop accepting before draining | Fail readiness, wait, then close |
| Old instances erroring mid-rollout | Incompatible schema or contract change | Multi-release compatible migrations |
| Rollback impossible | Change is forward-compatible only | Ensure each release works with the one before it |
| Capacity dips during rollout | maxUnavailable greater than zero without surge | Allow surge; keep unavailable at zero |
| Slow fault reaches the whole fleet | Gate too short to observe it | Longer gate for changes with delayed symptoms |
| Rollout hangs indefinitely | No progress deadline | Abort and alert after a bounded time |

**Limits**

> **Configuration guidance**
>
> - **Batch size** is blast radius: one instance is safest, a large percentage is fastest and riskiest.
> - **Health gate**: long enough for the fault class you are worried about — seconds for crash loops, minutes for leaks.
> - **Surge capacity** is what allows zero unavailability during replacement.
> - **Progress deadline**: bound the rollout so a stuck deployment alerts rather than hanging.
> - **Coexistence window**: the whole rollout duration, during which both versions must interoperate.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Rolling deployment | Default for stateless services | Two versions coexist |
| Blue-green | Instant switch and rollback | Double infrastructure |
| Canary | Risk-managed exposure | Slower; needs good metrics |
| Recreate | Simple, tolerant of downtime | Downtime per release |
| Feature flags | Decoupling release from deploy | Flag complexity; still needs deployment |
| Shadow traffic | Validation without user exposure | No user feedback; duplicated load |

Rolling and canary deployments are complementary rather than exclusive: a canary phase exposes a small fraction of traffic to the new version for long enough to evaluate real metrics, after which a rolling deployment completes the rest — combining measured risk with efficient completion.

**In real systems**

- **Orchestrators implement rolling updates natively**, with surge and unavailability parameters and readiness gates controlling progression.
- **Expand-contract schema migrations across multiple releases** are standard practice because a single-release rename breaks coexisting versions and prevents rollback.
- **Readiness-based draining before shutdown** removes the errors that otherwise occur on every deployment.
- **Progress deadlines** abort stuck rollouts rather than leaving a deployment half-complete indefinitely.
- **Combining a canary phase with a rolling completion** is common where the first instances are evaluated against real metrics before the remainder proceed.

**Common mistakes**

- **Health gates that do not reflect serving ability**, letting a broken release replace everything.
- **Single-release schema renames**, breaking coexisting versions and preventing rollback.
- **Gates too short** for faults that take minutes to appear.
- **No connection draining**, producing errors on every deploy.
- **Batch sizes chosen for speed** without regard to blast radius.
- **No progress deadline**, leaving stuck rollouts half-complete.
- **Forward-only changes**, removing rollback as an option.

**The staff-level view**

The deployment mechanism is rarely the problem; the compatibility of the change being deployed usually is.

- **Review changes for coexistence, not just correctness.** Both versions run simultaneously, so a schema rename, a message format change or a newly required field breaks the rollout itself regardless of how good the code is.
- **Require that rollback works.** A forward-only change removes the primary mitigation for deployment incidents, which means the response to a bad release becomes fixing forward under time pressure.
- **Scrutinise the health gate.** It is the only thing preventing a bad release from reaching the entire fleet, and a gate that only checks process liveness provides no protection whatever.
- **Match gate duration to the fault class.** Crash loops appear immediately; leaks and pool exhaustion take minutes, and a short gate promotes instances that will fail later.
- **Check the drain sequence.** Failing readiness before closing connections eliminates the errors that otherwise accompany every single deployment, and the wrong order is extremely common.

**Go deeper**

Rolling deployment replaces instances in batches, gating each batch on the new instances becoming healthy, so the service remains available and a broken release halts the rollout rather than completing it. With surge capacity, a new instance starts before an old one stops, so serving capacity never dips.

The cost is coexistence: both versions run simultaneously and share the same database, caches, message topics and API contracts, so every change must work for both. A column rename, a newly required field or an incompatible message format breaks the older instances during the rollout window — and, more seriously, makes rollback impossible, removing the primary mitigation for deployment incidents. Schema changes therefore span multiple releases, each compatible with the versions on either side.

Safety depends entirely on the health gate. A gate that only confirms the process started will promote batch after batch of a broken release until the whole fleet is converted, providing no protection at all. Duration matters too, since crash loops appear in seconds while memory leaks and pool exhaustion take minutes. And per-instance shutdown must fail readiness before closing connections, or every deployment produces user-visible errors.

Rolling deployment trades the simplicity of replacing everything at once for continuous availability, and in doing so imposes a compatibility requirement on every change.

**Gradual replacement bounds blast radius and is self-limiting.** Replacing a batch, waiting for health, then proceeding means a broken release stops after the first batch rather than converting the fleet. Combined with surge capacity — starting a new instance before stopping an old one — the service never loses capacity during the rollout. Batch size is therefore a direct expression of how much risk each step carries, and the cost of a slower rollout is almost always smaller than the cost of a wider failure.

**The health gate is the entire safety mechanism.** Progression is conditional on new instances working, so a gate that merely confirms the process is running provides no protection whatsoever — it converts the rollout into a slower version of replacing everything. The gate must exercise real serving capability, and its duration must match the fault class of concern: crash loops manifest within seconds, while memory leaks, connection pool exhaustion and slow degradation take minutes and will be promoted straight through a short gate.

**Coexistence is a design constraint on the change, not the deployment.** During the rollout both versions handle the same traffic and share the same database, caches, topics and contracts. A renamed column breaks old instances that still query it; a newly required field breaks old callers; an altered message format breaks old consumers; incompatible structures written to a shared cache break both. These are all reasonable changes in isolation and all incompatible with gradual replacement, which is why the review question must be whether the change works alongside the version preceding it.

**Rollback compatibility matters more than forward compatibility.** Rollback is the first mitigation for most deployment incidents, and it only functions if the previous version can still operate against current state. A migration that drops a column, or a format the old code cannot read, leaves only the option of fixing forward under time pressure — which is exactly the situation rollback exists to prevent. The expand-contract discipline, spreading a schema change across several releases so each intermediate state works with its neighbours, exists to preserve that option and is worth its apparent slowness the first time it is needed.

**Per-instance shutdown sequencing removes a whole class of deployment errors.** An instance that stops accepting connections on receiving a termination signal is still being routed to until the load balancer's health check notices, so those requests fail. Failing readiness first, waiting longer than the detection interval, then closing and draining in-flight work means traffic has already moved away. The wrong ordering produces intermittent errors on every deployment that are easily attributed to something else, and it is common enough to be worth verifying directly.

**Failures are almost always the change or the gate, not the mechanism.** Rolling deployment is well understood and reliably implemented by orchestrators; what goes wrong is a change that cannot coexist with its predecessor, or a gate that cannot distinguish a working instance from a broken one. Both are review concerns rather than configuration concerns, which is why the highest-value intervention is asking two questions of every release — does this work with the previous version running alongside it, and can we roll back — rather than tuning batch sizes.

**Prove it — interview questions**

1. **[Basic] How does a rolling deployment work?**

   <details><summary>Model answer</summary>

   Instances are replaced in batches rather than all at once. Each new instance must pass a health check before the next batch begins, so the service stays available throughout and a broken release halts the rollout instead of completing it. With surge capacity allowed, a new instance is started before an old one is stopped, so serving capacity never dips below the target.

   </details>

2. **[Basic] What is the main constraint rolling deployment imposes?**

   <details><summary>Model answer</summary>

   That two versions run simultaneously for the duration of the rollout, handling the same traffic and sharing the same database, caches and message topics. Everything shared must therefore work for both — a renamed column, a message format old consumers cannot parse, or a newly required API field all break the older instances while the rollout is in progress. This is a constraint on the change being made, not on the deployment mechanism.

   </details>

3. **[Senior] Why does the health gate matter so much?**

   <details><summary>Model answer</summary>

   Because it is the only thing that stops a bad release from reaching every instance. The gradual mechanism provides protection only if progression is conditional on the new instances actually working — so a gate that merely confirms the process started will happily promote batch after batch of a broken version until the whole fleet is converted. A gate that exercises real serving capability halts the rollout after the first batch, leaving the remainder on the previous version with an alert rather than an outage. Duration matters too: crash loops appear in seconds, but memory leaks and pool exhaustion take minutes, so a short gate passes those through.

   </details>

4. **[Senior] How do you change a database schema safely under rolling deployment?**

   <details><summary>Model answer</summary>

   Across multiple releases, so every intermediate state is compatible with the versions on either side. First release adds the new column as nullable and writes both old and new, which old instances are unaffected by. Second backfills and switches reads to the new column while still writing both, so rollback to the first release still functions. A later release stops writing the old column, and the column is dropped later still. A single-release rename breaks old instances throughout the rollout window because they query a column that no longer exists, and it makes rollback impossible because the previous version cannot work against the new schema — which removes the primary mitigation exactly when it is needed.

   </details>

5. **[Staff] Design a rolling deployment for a twenty-instance service including a schema change.**

   <details><summary>Model answer</summary>

   Rollout configured with surge allowed and maximum unavailability at zero, so a new instance starts before an old one stops and capacity never drops. The readiness gate requires the instance to serve successfully for a meaningful period rather than merely respond — thirty seconds is a reasonable starting point, long enough for startup faults and immediate crash loops, and I would extend it for releases where the risk is slow degradation. A progress deadline aborts and alerts if the rollout stalls, rather than leaving it half-complete indefinitely. Per instance, shutdown fails readiness first, waits longer than the load balancer's detection interval, then stops accepting and drains — because the reverse order produces errors on every single deployment and is very commonly wrong. The schema change spans three releases: add the column nullable and write both, then backfill and switch reads while still writing both, then stop writing the old one and drop it later. That discipline feels laborious until the first time a release must be reverted, at which point it is the only thing that makes reverting possible. The net effect is that a broken release halts after one instance with nineteen still serving, which is an alert rather than an incident.

   </details>

6. **[Principal] Why do rolling deployments fail in practice, given how well understood they are?**

   <details><summary>Model answer</summary>

   Almost never because of the deployment mechanism, and almost always because of the change being deployed or the health gate evaluating it. The mechanism assumes two things that teams routinely do not verify: that both versions can coexist, and that the gate genuinely distinguishes a working instance from a broken one. Coexistence fails through changes that are individually reasonable — a column rename, a required field, a new message format — and the failure appears during the rollout window rather than after it, so it looks like a deployment problem when it is a compatibility one. The gate fails through being too shallow or too brief, at which point the gradual mechanism provides no protection at all and simply replaces the entire fleet with a broken version more slowly than a recreate would have. Both are review problems rather than tooling problems, which is why I would put the emphasis on change review — does this work with the previous version running alongside it, and can we roll back — rather than on deployment configuration. The related point worth making explicitly is that rollback is the primary mitigation for deployment incidents, so a change that cannot be rolled back has removed it, and that constraint should be surfaced at design time rather than discovered at three in the morning.

   </details>

---

### Blue-green deployment

*Run two complete environments and switch traffic between them, so cutover and rollback are both instantaneous — at the cost of double infrastructure and shared state.*

**Flow:** `Blue environment live` → `Green deployed` → `Verification` → `Traffic switch` → `Blue held for rollback`

> **The 30-second version**  
> Run the new version as a complete parallel environment, verify it with no user traffic, switch routing atomically, and keep the old environment so rollback is a routing change rather than a redeployment.

**The problem**

Rolling deployment is gradual in both directions: a bad release is detected partway through, and rolling back takes as long as rolling forward did. During that window some users are on the broken version and some are not, which complicates both diagnosis and communication.

There are also changes that cannot tolerate two versions running simultaneously at all — a substantial rewrite, a change to a shared cache structure, a protocol change between components — where the coexistence requirement is the obstacle rather than the deployment speed.

> **Two full environments make cutover and rollback the same instant operation**  
> If the new version runs as a complete parallel environment, it can be verified before receiving any traffic, switched to atomically, and switched back just as quickly if something is wrong. The mixed-version window disappears, and rollback stops being a deployment and becomes a routing change measured in seconds.

**Mental model**

Two identical environments exist. One serves production traffic; the other holds the new version. After verification, traffic is redirected to the new environment, and the old one is kept ready for an immediate switch back.

1. **Blue** — The environment currently serving production traffic.
2. **Green** — The environment holding the new version, verified before receiving traffic.
3. **Switch** — The routing change that moves traffic, which is the deployment.
4. **Retention** — Keeping the previous environment available so rollback is a switch rather than a redeploy.
5. **Shared state** — The database, caches and queues both environments use — which is where the real difficulty lies.

> **The environments are separate but the database usually is not**  
> Blue and green run separate application instances and almost always share the same database. So a schema change still has to work for both versions, and after a switch the new version has written data the old version may not understand — which means rollback can be blocked by data written in the minutes since cutover. The instant-rollback promise holds for code and not automatically for state.

**How it works**

**The cutover and its properties**

```text
BEFORE
  blue  (v1)  <- all production traffic
  green (v2)  <- deployed, no traffic

VERIFY GREEN WITHOUT USERS
  smoke tests against green directly
  synthetic transactions
  internal traffic or employee routing
  -> catches gross breakage before any user sees it
  -> this is the phase rolling deployment cannot offer

SWITCH
  routing changes: load balancer, DNS, or service mesh
  -> all traffic moves at once
  -> no mixed-version window for CODE

AFTER
  blue  (v1)  <- idle, retained
  green (v2)  <- all production traffic
  keep blue for a defined period
  -> rollback = switch back = seconds

SWITCH MECHANISM MATTERS
  load balancer target change   seconds, precise
  service mesh routing rule     seconds, precise
  DNS change                    minutes to hours,
                                governed by TTL and
                                client caching
  -> DNS is the wrong mechanism for anything needing
     fast rollback
  -> and DNS-based rollback is equally slow

IN-FLIGHT REQUESTS
  requests already being served by blue must complete
  -> drain rather than cut instantly
  -> the switch is atomic for NEW requests only
```

1. **Verify green before switching** — The ability to test the complete new environment under realistic conditions with no user exposure is the main advantage over rolling deployment.
2. **Switch at the load balancer or mesh, not DNS** — DNS propagation is governed by caches you do not control, which makes both cutover and rollback slow and uneven.
3. **Retain blue for a defined period** — Rollback is only instant while the previous environment still exists, so the retention window is the rollback window.
4. **Keep schema changes compatible anyway** — Both environments share a database, so a schema that only the new version understands blocks rollback.
5. **Consider data written after cutover** — Rollback restores the code, not the data, and new-version writes may be unreadable to the old one.
6. **Drain in-flight requests** — The switch is atomic for new requests; existing ones must be allowed to finish.

**What blue-green does and does not solve**

```text
SOLVES
  mixed-version window for application code
    -> no two versions serving simultaneously
  rollback speed
    -> a routing change, not a redeployment
  pre-production verification of the real environment
    -> test green exactly as it will serve

DOES NOT SOLVE
  shared database schema compatibility
    -> both environments use the same database
    -> a breaking schema change breaks blue too
  data written after cutover
    -> rollback returns to old code, not old data
    -> if v2 wrote rows v1 cannot read, rollback
       is not clean
  shared caches and queues
    -> a message published by v2 may be unreadable
       by v1 after rollback
  session state
    -> in-memory sessions in blue are lost on switch
       unless externalised

THE COST
  double infrastructure during deployment
  -> for a large fleet this is significant
  -> mitigated if environments are short-lived, but
     that reduces the retention window for rollback
```

> **Rollback restores code, not data**  
> After a switch, the new version begins writing to the shared database. If it writes in a format or structure the old version cannot handle, then switching back leaves the old code reading data it does not understand — so the rollback that was supposed to take seconds instead requires a data repair. The window between cutover and rollback is therefore a period in which incompatible writes accumulate, which is a strong argument for keeping writes backward-compatible even with blue-green in place.

**Worked example**

A blue-green cutover for a service with a shared database.

**Cutover plan**

```text
ENVIRONMENTS
  blue: v1.4, 20 instances, serving
  green: v1.5, 20 instances, deployed, no traffic
  shared: database, cache, message queues

PRE-SWITCH VERIFICATION  (the key advantage)
  smoke tests against green endpoints directly
  synthetic checkout transactions
  internal employee traffic routed to green for 1 hour
  -> gross breakage caught with zero user exposure

SCHEMA
  v1.5 requires a new column
  -> added in a PREVIOUS release, nullable, and v1.4
     tolerates it
  -> because blue and green share the database, an
     incompatible schema would break blue as well
  -> blue-green does NOT remove the expand-contract
     discipline

SWITCH
  load balancer target group changed blue -> green
  in-flight requests on blue allowed to drain
  elapsed: a few seconds

POST-SWITCH
  watch error rate, latency, business metrics
  blue retained for 24 hours
  -> rollback available for that window
  -> after which blue is decommissioned

ROLLBACK CONSIDERATION
  v1.5 writes a new field that v1.4 ignores safely
  -> rollback is clean
  if v1.5 had written a structure v1.4 could not parse,
  rollback would require data repair
  -> so write compatibility is still required

COST
  40 instances during the window instead of 20
```

| Metric | Value | Note |
|---|---|---|
| Switch | seconds | load balancer |
| Verification | before traffic | **unique advantage** |
| Retention | 24 h | rollback window |
| Cost | 2× instances | during deployment |

> **The retention window is the rollback window**  
> Rollback is instant only while the previous environment still exists. Decommissioning blue immediately after the switch to save infrastructure cost converts blue-green into an ordinary deployment with extra steps, because recovering then means redeploying the old version. The retention period should be chosen from how long problems typically take to surface — often longer than teams initially assume, since some faults only appear under a daily traffic peak or a scheduled job.

**When to use it**

- **Changes that cannot tolerate mixed versions**, such as protocol or shared-structure changes.
- **When instant rollback matters**, and a gradual reversal would be too slow.
- **Where pre-production verification of the real environment is valuable**, particularly for high-risk releases.
- **Services with sufficient spare capacity** to run two environments briefly.
- **Regulated changes requiring explicit verification** before user exposure.

**When to avoid it**

- **Do not use it when infrastructure cost is prohibitive**, since it doubles capacity during deployment.
- **Do not assume it removes schema compatibility requirements**, since the database is shared.
- **Do not use DNS as the switch mechanism** where fast rollback matters.
- **Do not decommission the old environment immediately**, which eliminates the rollback advantage.
- **Do not rely on it alone for risk management**, since all traffic moves at once — canary exposes gradually.

**Advantages**

- **Instant cutover and instant rollback**, both being routing changes.
- **No mixed-version window** for application code.
- **Full verification before exposure**, in the actual environment that will serve.
- **Simple mental model** — one environment live, one standby.
- **Clean separation** between deployment and release.

**Disadvantages**

- **Double infrastructure** during the deployment window.
- **Shared database still constrains schema changes.**
- **Rollback does not undo data** written after cutover.
- **All traffic switches at once**, so a subtle fault reaches every user immediately.
- **Session and in-memory state is lost** unless externalised.
- **Retention costs money**, and shortening it removes the rollback benefit.

**Trade-offs**

**Strategy comparison**

| Aspect | Blue-green | Rolling | Canary |
|---|---|---|---|
| Cutover speed | Instant | Gradual | Gradual |
| Rollback speed | Instant | Gradual | Instant for the canary |
| Mixed versions | None | Throughout rollout | Deliberate |
| Infrastructure cost | Double temporarily | Minimal surge | Small overhead |
| Exposure control | All at once | Batch by batch | Fine-grained |
| Pre-exposure verification | Full environment | Limited | Real traffic subset |

Blue-green and canary answer different concerns: blue-green optimises for reversal speed and eliminates version mixing, while canary optimises for limiting how many users encounter a fault. Where both matter, a canary phase within the green environment before the full switch combines them.

**How it fails**

**Blue-green failures**

| Failure | Cause | Fix |
|---|---|---|
| Rollback not actually instant | Old environment already decommissioned | Retain it for a defined window |
| Blue broken by the deployment | Incompatible schema change on the shared database | Expand-contract migrations regardless of strategy |
| Rollback leaves broken data | New version wrote structures the old cannot read | Keep writes backward-compatible |
| Switch takes hours | DNS used as the mechanism | Switch at the load balancer or mesh |
| Errors during cutover | In-flight requests cut off | Drain blue rather than cutting instantly |
| Users logged out after switch | Session state held in memory | Externalise session state |
| Subtle fault reached every user | All traffic switched at once | Canary phase before the full switch |

**Limits**

> **Practical parameters**
>
> - **Infrastructure**: double capacity for the duration of the deployment and retention window.
> - **Switch time**: seconds at a load balancer or mesh; minutes to hours via DNS.
> - **Retention**: hours to a day, chosen from how long faults typically take to surface.
> - **Schema**: still requires compatibility, because the database is shared.
> - **Data written post-cutover** is not reverted by a rollback.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Blue-green | Instant rollback; no version mixing | Double infrastructure |
| Rolling | Default; cost-efficient | Mixed versions; gradual rollback |
| Canary | Limiting exposure | Slower; needs good metrics |
| Feature flags | Decoupling release from deployment | Flag lifecycle management |
| Shadow traffic | Validation without exposure | No user feedback; duplicate load |
| Recreate | Simplicity where downtime is acceptable | Downtime |

Feature flags address a related but distinct concern: they separate deploying code from enabling behaviour, so a change can be shipped inertly and turned on independently. Combined with blue-green, the deployment and the release become two separately reversible decisions.

**In real systems**

- **Load balancer target group switching** is the standard cutover mechanism, because it is fast and precise in a way DNS is not.
- **Retaining the previous environment for a defined window** is what preserves the instant-rollback property that motivates the approach.
- **Expand-contract schema migrations remain necessary** under blue-green, because both environments share the same database.
- **Canary phases within the green environment** combine limited exposure with the fast reversal blue-green provides.
- **Externalised session state** is a precondition, since an in-memory session in the old environment does not survive the switch.

**Common mistakes**

- **Decommissioning the old environment immediately**, losing the rollback advantage.
- **Assuming shared schema compatibility is unnecessary**, breaking both environments.
- **Expecting rollback to undo data**, when it only restores code.
- **Switching via DNS**, making cutover and rollback slow and uneven.
- **Cutting traffic without draining**, causing errors on in-flight requests.
- **In-memory session state**, logging users out at the switch.
- **Switching all traffic at once** for changes whose faults are subtle.

**The staff-level view**

Blue-green is often adopted for the instant rollback and then undermined by the details that make rollback conditional.

- **Protect the retention window.** Decommissioning the old environment immediately to save cost converts blue-green into an ordinary deployment, since recovery then means redeploying rather than switching.
- **Do not let it excuse schema discipline.** Both environments share a database, so an incompatible migration breaks the supposedly safe environment too, and the expand-contract requirement is unchanged.
- **Be explicit that rollback restores code, not data.** Writes made after cutover persist, so a new version writing structures the old cannot read turns an instant rollback into a data repair.
- **Use a routing mechanism you control.** DNS-based switching is governed by caches outside your control, which makes both cutover and rollback slow and uneven — and rollback speed was the entire point.
- **Add a canary phase for subtle faults.** Switching all traffic at once means a fault that metrics cannot immediately detect reaches every user, which the gradual strategies would have limited.

**Go deeper**

Blue-green runs two complete environments, one serving production and one holding the new version. The new environment is verified with no user exposure — smoke tests, synthetic transactions, internal traffic — and then routing switches all traffic to it at once. Because the previous environment is retained, rollback is a routing change measured in seconds rather than a gradual redeployment, and there is no window in which two versions serve simultaneously.

The limits come from what is shared. Both environments use the same database, so schema changes must still be compatible with both versions — an incompatible migration breaks the standby environment that rollback depends on. And rollback restores code rather than data: writes made after cutover persist, so a new version writing structures the old cannot read converts an instant rollback into a data repair.

Two operational details determine whether the promise holds. The switch must happen at a load balancer or service mesh rather than through DNS, whose propagation is governed by caches outside your control in both directions. And the previous environment must be retained long enough for faults to surface — decommissioning it immediately to save infrastructure cost turns blue-green into an ordinary deployment with extra steps.

Blue-green deployment eliminates the mixed-version window and makes reversal a routing decision by maintaining two complete environments.

**Verification before exposure is the distinctive capability.** The new environment can be exercised exactly as it will serve — smoke tests, synthetic transactions, internal or employee traffic — with no user affected by anything found. Rolling deployment cannot offer this, because its first test is real users on real traffic. For high-risk changes this phase is frequently the main reason to accept the infrastructure cost.

**Cutover and rollback are the same operation.** Because both environments exist, moving traffic in either direction is a routing change taking seconds. This matters most for the reverse direction: rollback is the primary mitigation for deployment incidents, and a rolling rollback takes as long as the rollout did, during which some users remain on the broken version. The property holds only while the previous environment exists, which makes the retention window the rollback window — and the cost pressure to reclaim that capacity early is the most common way the benefit is lost.

**The database is shared, and that bounds what is actually isolated.** Blue and green separate application instances, not state. A migration that only the new version understands breaks the old environment, which is precisely the one being held for rollback, so expand-contract discipline across multiple releases remains necessary. Blue-green changes how code is swapped; it does not change what a single database can simultaneously support.

**Rollback restores code, not data.** From cutover onward the new version writes to shared storage, and switching back leaves the old code reading whatever accumulated. If those writes use structures the old version cannot parse, the seconds-long rollback becomes a data repair under time pressure — so backward-compatible writes are a requirement even here, and the practical rollback window is bounded by incompatible data accumulation as well as by environment retention. The same applies to messages published to shared queues and to any cache structures written during the window.

**Exposure control is worse, not better, than the gradual strategies.** All traffic moves at once, so a fault that metrics do not immediately surface — a subtle correctness problem, a slow leak, something that only manifests under a particular workload — reaches every user simultaneously. Rolling and canary deployments both limit this by construction. Where both fast reversal and limited exposure matter, running a canary phase within the green environment before the full switch combines them, and is usually the right composition for high-risk changes.

**The mechanism of the switch determines whether the promise is real.** A load balancer target change or a service mesh routing rule takes effect in seconds and is fully under your control. DNS is governed by resolver and client caches that honour time-to-live inconsistently, so both the cutover and — critically — the rollback take minutes to hours and affect users unevenly. Since reversal speed is the entire justification for the approach, using a mechanism that cannot deliver it undermines the strategy while retaining all of its costs.

**Prove it — interview questions**

1. **[Basic] How does blue-green deployment work?**

   <details><summary>Model answer</summary>

   Two complete environments exist: one serving production traffic and one holding the new version with no traffic. The new environment is verified — smoke tests, synthetic transactions, perhaps internal traffic — and then routing is changed so all production traffic moves to it at once. The previous environment is retained, so rolling back is simply switching routing back, which takes seconds rather than requiring a redeployment.

   </details>

2. **[Basic] What is the main advantage over rolling deployment?**

   <details><summary>Model answer</summary>

   Speed and cleanliness of reversal, plus the ability to verify before exposure. Rollback is a routing change rather than a gradual redeployment, and there is no window during which two versions serve traffic simultaneously — which matters for changes that cannot tolerate coexistence at all. The verification phase is also distinctive: the complete new environment can be exercised under realistic conditions with zero user exposure, which rolling deployment cannot offer.

   </details>

3. **[Senior] Why does blue-green not remove schema compatibility requirements?**

   <details><summary>Model answer</summary>

   Because the environments are separate at the application layer and almost always share the same database. A migration that only the new version understands breaks the old environment immediately, which is the one you were relying on for rollback. So expand-contract migrations across multiple releases remain necessary regardless of deployment strategy — blue-green changes how application instances are replaced, not what the database can simultaneously support.

   </details>

4. **[Senior] Why is rollback not as complete as it appears?**

   <details><summary>Model answer</summary>

   Because it restores code and not data. From the moment of cutover the new version writes to the shared database, so switching back leaves the old code reading whatever accumulated in the interim. If those writes are in a structure or format the old version cannot handle, the rollback that was supposed to take seconds instead requires a data repair under pressure. That makes backward-compatible writes a requirement even with blue-green in place, and it means the practical rollback window is bounded not only by environment retention but by how much incompatible data has been written.

   </details>

5. **[Staff] When would you choose blue-green over rolling or canary?**

   <details><summary>Model answer</summary>

   When the change cannot tolerate mixed versions, or when reversal speed is the dominant concern. A protocol change between components, a shared cache structure change, or a substantial rewrite are all cases where having two versions serving simultaneously is the actual obstacle, and blue-green removes it entirely. Reversal speed matters when the cost of even a few minutes of a bad release is high — a routing switch back is seconds where a rolling rollback takes as long as the rollout did. Against that I would weigh two things: the infrastructure cost of running double capacity for the deployment plus the retention window, and the fact that all traffic switches at once, so a subtle fault that metrics do not immediately surface reaches every user — which is exactly what canary would have limited. Where both concerns apply I would run a canary phase within green before the full switch, which gets limited exposure and fast reversal together. And I would make sure the retention window is long enough that problems have actually had a chance to surface, since some only appear at a daily peak or when a scheduled job runs.

   </details>

6. **[Principal] What does blue-green actually guarantee, and what do teams assume it guarantees?**

   <details><summary>Model answer</summary>

   It guarantees that application code can be swapped and unswapped quickly, and that no two versions of that code serve traffic simultaneously. What teams tend to assume is that this makes deployment safe generally, and several things quietly fall outside the guarantee. The database is shared, so schema compatibility is unchanged and a bad migration breaks the standby environment too — the very thing the strategy was protecting. Data written after cutover persists through a rollback, so the reversal restores behaviour and not state, and an incompatible write format turns the seconds-long rollback into a repair exercise. Messages published to shared queues by the new version may be unreadable after reverting. In-memory session state does not survive the switch. And because all traffic moves at once, exposure control is actually worse than rolling deployment rather than better — a subtle fault reaches everyone immediately. My practical position is that blue-green is an excellent answer to a specific question, which is how to make reversal fast and eliminate version coexistence, and that adopting it does not relieve any of the discipline around compatible schemas, compatible writes and externalised state. Where it goes wrong in practice is almost always one of those, plus the cost pressure to decommission the old environment early, which removes the property that justified the approach.

   </details>

---

### Canary releases

*Expose a small fraction of real traffic to the new version and decide from measured comparison whether to proceed, so a bad release affects few users.*

**Flow:** `Small traffic slice` → `Baseline comparison` → `Metric evaluation` → `Promote or abort` → `Full rollout`

> **The 30-second version**  
> Send a small slice of real traffic to the new version, compare it against the concurrent baseline on technical and business criteria, and promote or abort automatically.

**The problem**

A release passes every test and breaks in production, because production has traffic patterns, data shapes, concurrency levels and third-party behaviours that no test environment reproduces. The first genuine test of a release is real traffic, and the question is how many users experience it before anyone knows.

Deploying to everyone and watching dashboards means the entire user base is the experiment. Deploying gradually helps, but only if someone is actually comparing the new version's behaviour against the old rather than waiting for an alert loud enough to notice.

> **Route a small slice, compare against the baseline, decide from data**  
> A canary exposes the new version to a few per cent of real traffic while the remainder continues on the current version — which gives a controlled comparison under identical conditions. The decision to proceed becomes a measurement rather than a judgement, and a bad release is caught having affected a small, bounded fraction of users.

**Mental model**

Two versions run concurrently with a controlled traffic split. The canary's metrics are compared against the baseline's over the same window, and the split increases only if the comparison holds.

1. **Split** — The fraction of traffic sent to the new version — the bounded exposure.
2. **Baseline** — The current version serving the remainder, providing the comparison.
3. **Comparison window** — How long to observe before deciding, determined by how quickly faults manifest.
4. **Criteria** — The metrics and thresholds that constitute pass or fail.
5. **Progression** — Increasing the split in stages, or aborting and reverting.

> **A canary without automated comparison is just a slow rollout**  
> The value comes from evaluating the new version against the baseline on defined criteria, not from the traffic split itself. If nobody is comparing — or if the comparison is a human glancing at a dashboard — then a subtle regression proceeds through every stage to full rollout, and the exposure limitation achieved nothing except delaying the moment everybody was affected.

**How it works**

**Progression and the comparison that gates it**

```text
STAGES
  1%   observe 15 min
  5%   observe 30 min
  25%  observe 1 h
  50%  observe 1 h
  100%
-> each stage gated on the comparison passing

WHAT IS COMPARED  (canary vs baseline, same window)
  error rate           canary must not be worse
  latency p99          canary must not be worse
  business metrics     conversion, completion rate
  resource usage       memory growth, CPU per request
-> comparing against the CONCURRENT baseline, not
   against a historical figure, controls for traffic
   mix, time of day, and external events

WHY COMPARE RATHER THAN THRESHOLD
  absolute threshold: "error rate < 1%"
    -> fails if the baseline is also at 1.2% due to a
       dependency problem
    -> aborts a perfectly good release
  relative comparison: "canary not worse than baseline"
    -> unaffected by conditions common to both

STATISTICAL REALITY AT 1%
  1% of 10,000 req/s = 100 req/s
  in 15 min = 90,000 requests
  -> enough to detect a meaningful error-rate change
  1% of 50 req/s = 0.5 req/s
  in 15 min = ~450 requests
  -> NOT enough to detect anything subtle
  -> low-traffic services need longer windows or
     larger initial splits
```

1. **Compare against the concurrent baseline, not a threshold** — Relative comparison controls for traffic mix, time of day and external conditions affecting both versions.
2. **Automate the decision** — A human watching a dashboard misses subtle regressions and does not scale across frequent releases.
3. **Size the window for statistical significance** — At low traffic volumes a small split produces too few requests to detect anything, so the window or the split must grow.
4. **Include business metrics, not only technical ones** — A release can be technically healthy and reduce conversion, which error rates and latency will never reveal.
5. **Abort automatically on regression** — Waiting for a human to notice and act is the slowest part of the loop and the easiest to remove.
6. **Ensure the split is representative** — Routing by user or session avoids a canary that only sees one geography, one client version, or one traffic pattern.

**Traffic splitting and its pitfalls**

```text
SPLIT MECHANISMS
  random per request
    + statistically clean sample
    - one user may hit both versions, which breaks
      session continuity and confuses per-user metrics
  sticky by user or session
    + consistent experience; per-user metrics valid
    + the usual correct choice
    - a small split may be an unrepresentative cohort
  by attribute (region, client version, tier)
    + deliberate targeting
    - NOT a random sample, so comparison is biased

THE BIAS TRAP
  canary routed to one region only
  -> canary metrics reflect that region's traffic mix,
     latency profile and time of day
  -> baseline reflects everything else
  -> the comparison is between populations, not
     versions

SESSION AFFINITY MATTERS
  without it, a user may be served by v1 then v2
  within one flow
  -> broken multi-step journeys
  -> and per-user business metrics become meaningless

THE COLD-START CONFOUND
  a freshly started canary has cold caches, an unwarmed
  JIT, and new connection pools
  -> it will look slower for the first minutes
  -> comparing too early aborts good releases
  -> allow a warm-up period before evaluation begins
```

> **Cold start makes a healthy canary look broken**  
> A newly started instance has empty caches, unoptimised runtime code paths and fresh connection pools, so its latency is genuinely worse for the first minutes regardless of the change. Beginning the comparison immediately produces false failures and trains teams to override the automation — which then removes the protection. An explicit warm-up period before evaluation starts avoids both problems.

**Worked example**

An automated canary pipeline for a checkout service.

**Canary configuration**

```text
TRAFFIC  10,000 req/s, sticky routing by session

STAGES AND GATES
  stage 1   1%   warm-up 5 min, observe 15 min
  stage 2   5%   observe 30 min
  stage 3   25%  observe 1 h
  stage 4   50%  observe 1 h
  stage 5   100%

COMPARISON CRITERIA  (canary vs concurrent baseline)
  error rate         not more than 1.2x baseline
  p99 latency        not more than 1.15x baseline
  checkout success   not lower than baseline
  memory per pod     no upward trend beyond baseline
-> any breach: abort, revert traffic to baseline, alert

WHY A RATIO AND NOT AN ABSOLUTE THRESHOLD
  if the payment provider degrades, BOTH versions see
  higher errors
  -> an absolute threshold aborts a good release
  -> a ratio correctly shows the canary is no worse

WHY BUSINESS METRICS ARE INCLUDED
  a release once passed every technical gate and
  reduced checkout completion by 4% because a button
  moved below the fold
  -> error rate normal, latency normal, revenue down
  -> only a business metric catches this

LOW-TRAFFIC ENDPOINTS
  at 1% some endpoints see too few requests to judge
  -> those criteria are evaluated only from stage 3
  -> otherwise noise causes false aborts

RESULT
  a bad release affects ~1% of users for ~15 minutes
  instead of 100% until someone notices
```

| Metric | Value | Note |
|---|---|---|
| Initial exposure | 1% / 15 min | **bounded blast radius** |
| Criteria | ratio to baseline | not absolute |
| Business metric | checkout success | catches silent harm |
| Abort | automated | no human latency |

> **Comparing against the concurrent baseline removes the confounds**  
> Absolute thresholds fail in both directions: they abort good releases when a shared dependency degrades, and they pass bad ones when the threshold was set generously. Comparing the canary against the baseline running at the same moment, on the same traffic, controls for time of day, traffic mix, dependency health and external events — so the difference measured is attributable to the version rather than to the conditions.

**When to use it**

- **High-risk changes**, where limiting exposure is worth a slower rollout.
- **Services with sufficient traffic** for a small split to yield statistically meaningful data.
- **Where business metrics matter**, since technically healthy releases can still cause harm.
- **Frequent deployment**, where automated evaluation scales in a way human review does not.
- **Validating behaviour that only production reproduces**, such as real data shapes and third-party responses.

**When to avoid it**

- **Do not canary at traffic levels too low for significance**, where the sample proves nothing.
- **Do not canary without automated comparison**, which reduces it to a slow rollout.
- **Do not evaluate during warm-up**, which produces false failures.
- **Do not split by an attribute and call it a random sample**, which biases the comparison.
- **Do not use canaries for changes that cannot coexist** with the current version.

**Advantages**

- **Bounded exposure**, so a bad release affects a small fraction of users.
- **Real production conditions**, which no test environment reproduces.
- **Data-driven promotion**, replacing judgement with measurement.
- **Catches business-metric regressions** invisible to technical monitoring.
- **Automated abort** removes human reaction time from the loop.
- **Progressive confidence**, with exposure increasing as evidence accumulates.

**Disadvantages**

- **Slow**, since meaningful observation takes time at each stage.
- **Requires sufficient traffic** for statistical validity.
- **Two versions coexist**, with the usual compatibility requirements.
- **Comparison infrastructure is real work** to build and maintain.
- **Cold-start effects cause false failures** without a warm-up period.
- **Subtle or delayed faults** may not manifest within the observation window.

**Trade-offs**

**Canary configuration trade-offs**

| Choice | Benefit | Cost |
|---|---|---|
| Small initial split | Minimal exposure | Weak statistics at low traffic |
| Larger initial split | Faster significance | More users exposed to a fault |
| Long observation windows | Catches slow-manifesting faults | Very slow rollouts |
| Relative comparison | Robust to shared conditions | Requires a concurrent baseline |
| Absolute thresholds | Simple | False aborts and false passes |
| Sticky routing | Consistent experience; valid per-user metrics | Cohort may be unrepresentative |

The tension between exposure and statistical power is fundamental: a smaller split protects more users and detects less. At low traffic the resolution is usually longer windows rather than larger splits, since time costs the team while exposure costs users.

**How it fails**

**Canary failures**

| Failure | Cause | Fix |
|---|---|---|
| Regression promoted to full rollout | No automated comparison | Automate evaluation against defined criteria |
| Good releases repeatedly aborted | Evaluation during cold start | Warm-up period before comparison begins |
| Canary compared unfairly | Split by region or attribute | Random or session-sticky routing |
| Comparison proves nothing | Too few requests at the split | Longer window or larger initial split |
| Technically fine, business harm | Only technical metrics evaluated | Include conversion and completion metrics |
| Broken user journeys | Per-request splitting without stickiness | Session affinity |
| Good release aborted by a dependency issue | Absolute thresholds | Compare as a ratio to the baseline |

**Limits**

> **Sizing guidance**
>
> - **Initial split**: 1–5% where traffic supports it; larger at low volume.
> - **Statistical power**: a 1% split of a low-traffic service yields too few requests to detect subtle change.
> - **Warm-up**: several minutes before evaluation, to avoid cold-start false failures.
> - **Window**: long enough for the fault class of concern — minutes for crashes, longer for leaks.
> - **Comparison**: relative to the concurrent baseline, expressed as a ratio rather than an absolute limit.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Canary release | Limiting exposure with measurement | Slow; needs traffic and tooling |
| Rolling deployment | Default efficiency | Less measurement; broader exposure per step |
| Blue-green | Fast reversal; no version mixing | All traffic at once; double infrastructure |
| Feature flags | Per-user control; instant disable | Flag lifecycle; code complexity |
| Shadow traffic | Validation with zero exposure | No user feedback; duplicate load |
| A/B testing | Measuring product impact | Different purpose; longer horizons |

Shadow traffic is the complementary technique when exposure must be zero: the new version processes copies of real requests without its responses reaching users, which validates correctness and performance under real load but cannot measure user-facing outcomes.

**In real systems**

- **Automated canary analysis tools** compare canary and baseline metrics on defined criteria and promote or abort without human intervention.
- **Relative comparison against a concurrent baseline** is standard, because absolute thresholds abort good releases whenever a shared dependency degrades.
- **Business metrics in canary criteria** catch regressions that error rates and latency cannot, such as a layout change reducing completion.
- **Session-sticky traffic splitting** preserves multi-step user journeys and keeps per-user metrics meaningful.
- **Warm-up periods before evaluation** prevent cold caches and unoptimised runtime paths from producing false failures.

**Common mistakes**

- **Traffic splitting with no automated comparison**, which is just a slow rollout.
- **Absolute thresholds** rather than comparison against the baseline.
- **Evaluating during cold start**, causing false aborts.
- **Splits too small for the traffic volume**, so the gate passes on noise.
- **Only technical metrics**, missing business-level harm.
- **Per-request splitting without stickiness**, breaking user journeys.
- **Attribute-based splits treated as random samples**, biasing the comparison.

**The staff-level view**

Canary releases deliver value through the comparison, and teams frequently build the traffic splitting without the evaluation.

- **Automate the promotion decision.** A canary gated on someone watching a dashboard misses subtle regressions and does not scale, which reduces an expensive mechanism to a slow rollout.
- **Compare relatively, never absolutely.** Absolute thresholds abort good releases when a shared dependency degrades and pass bad ones when set generously; the concurrent baseline controls for everything affecting both.
- **Insist on at least one business metric.** Releases that are technically flawless and commercially harmful are common, and no amount of error-rate monitoring surfaces them.
- **Check the statistics before trusting the result.** A one per cent split of a low-traffic service produces a sample that cannot detect anything, so the gate is passing on noise.
- **Allow for warm-up.** Cold-start false failures train teams to override the automation, which removes the protection entirely and is worse than not having it.

**Go deeper**

A canary release routes a few per cent of production traffic to the new version while the rest continues on the current one, then compares the two before expanding. Exposure to a bad release is bounded to a small fraction of users for a short window, and promotion becomes a measurement rather than an act of confidence — which matters because production has traffic patterns, data shapes and third-party behaviours no test environment reproduces.

The comparison must be relative to the concurrent baseline rather than against absolute thresholds, because a shared dependency degrading affects both versions and would abort a perfectly good release under a fixed limit. It must also be automated: a canary gated on a human watching dashboards misses subtle regressions and reduces the whole mechanism to a slow rollout. And it should include at least one business metric, since a release can be technically flawless while reducing conversion.

Two practical constraints shape the configuration. Statistical power requires enough requests in the slice, so low-traffic services need longer windows rather than tiny splits. And a freshly started canary has cold caches and unoptimised runtime paths, so evaluating immediately produces false failures that train teams to override the automation — which removes the protection entirely.

Canary releases limit how many users encounter a fault and, more importantly, produce evidence about whether one exists.

**The comparison is the mechanism; the split is only its precondition.** Routing a fraction of traffic to a new version while the remainder serves the current one creates a controlled experiment under identical conditions — same time of day, same traffic mix, same dependency health. The promotion decision then rests on measured difference rather than on absence of alarms, which catches regressions too subtle to trip any threshold. Teams that build the traffic splitting without the automated evaluation obtain a slow rollout and none of the value.

**Relative comparison is robust where absolute thresholds are not.** A fixed error-rate limit aborts a good release whenever a shared dependency degrades, because both versions are affected and only one is being judged. Set generously enough to avoid that, it passes releases that are meaningfully worse than what they replaced. Expressing criteria as ratios against the concurrent baseline eliminates every confound common to both versions, which is most of them.

**Business metrics catch what technical monitoring cannot.** A release can have identical error rates, identical latency and healthy resource usage while reducing checkout completion through a change in how something is presented. No amount of infrastructure monitoring surfaces this, and by the time it appears in weekly reporting the release is everywhere and the causal link is uncertain. Including at least one outcome metric in the canary criteria is what makes the mechanism capable of detecting commercial harm as well as technical failure.

**Statistical power is a real constraint, not a formality.** A one per cent split of a high-volume service yields tens of thousands of requests in a short window, which supports confident comparison. The same split of a low-traffic service yields a few hundred, which cannot distinguish a genuine regression from noise — and the gate then passes on insufficient evidence, conferring false confidence that is arguably worse than having no gate. The remedy is longer windows rather than larger splits where possible, since time is paid by the team and exposure is paid by users.

**Cold start is the most common source of false failures.** A newly started instance has empty caches, unoptimised runtime code paths and fresh connection pools, so its latency is genuinely worse for several minutes regardless of the change being tested. Evaluating immediately aborts healthy releases, and repeated false aborts train teams to override the automation — at which point the protection is gone while its cost remains. An explicit warm-up period before evaluation begins avoids both outcomes.

**Routing choices bias the comparison if made carelessly.** Splitting per request gives a statistically clean sample but can serve one user by both versions within a single flow, breaking multi-step journeys and rendering per-user metrics meaningless. Session-sticky routing fixes that and is usually correct. Splitting by an attribute such as region or client version is a legitimate targeting decision but is not a random sample: the canary then reflects that population's traffic profile and latency characteristics, so the comparison measures the difference between populations rather than between versions — a subtle error that produces confident, wrong conclusions.

**Prove it — interview questions**

1. **[Basic] What is a canary release?**

   <details><summary>Model answer</summary>

   Routing a small fraction of real production traffic to the new version while the rest continues on the current one, then comparing their behaviour before deciding whether to proceed. The point is bounded exposure with genuine evidence: a bad release affects a few per cent of users for a short window rather than everyone until someone notices, and the decision to expand comes from measurement rather than from confidence.

   </details>

2. **[Basic] Why compare against the baseline rather than a fixed threshold?**

   <details><summary>Model answer</summary>

   Because a fixed threshold cannot distinguish a problem in the release from conditions affecting everything. If a payment provider degrades, both versions see elevated errors, and an absolute threshold aborts a perfectly good release; conversely a generous threshold passes a release that is meaningfully worse than what it replaced. Comparing the canary against the baseline running concurrently on the same traffic controls for time of day, traffic mix, dependency health and external events, so any difference measured is attributable to the version.

   </details>

3. **[Senior] Why does a canary need automated evaluation?**

   <details><summary>Model answer</summary>

   Because the traffic split is not the valuable part — the comparison is. Without automation, promotion depends on a human looking at dashboards, which misses subtle regressions, adds reaction time to every abort, and does not scale across frequent releases. In that state a canary is simply a slow rollout with extra steps: the regression proceeds through every stage to full exposure, and the limitation bought nothing except delaying when everyone was affected. Automated criteria with automatic abort remove human latency from the loop entirely.

   </details>

4. **[Senior] What statistical problem do low-traffic services have with canaries?**

   <details><summary>Model answer</summary>

   That a small split of small traffic produces too few requests to detect anything. One per cent of ten thousand requests per second gives ninety thousand requests in fifteen minutes, which is ample; one per cent of fifty requests per second gives a few hundred, which cannot distinguish a real regression from noise. The gate then passes on insufficient evidence, which is arguably worse than no gate because it confers false confidence. The remedies are longer observation windows or larger initial splits, and the trade is explicit — time costs the team while exposure costs users, which usually argues for longer windows.

   </details>

5. **[Staff] Design an automated canary pipeline for a checkout service.**

   <details><summary>Model answer</summary>

   Session-sticky routing so a user's multi-step journey is served consistently by one version, which also keeps per-user business metrics meaningful. Staged progression — one per cent, five, twenty-five, fifty, then full — with each stage gated on an automated comparison against the concurrent baseline. Criteria expressed as ratios rather than absolutes: error rate no more than a small multiple of baseline, p99 latency similarly, and crucially at least one business metric such as checkout completion, because a release can pass every technical gate while reducing conversion through something like a layout change, and no amount of error-rate monitoring surfaces that. A warm-up period before evaluation begins at each stage, since a freshly started instance has cold caches and unoptimised runtime paths and will look slower regardless of the change — false aborts train people to override the automation, which removes the protection. For low-traffic endpoints, defer their criteria to later stages where the sample is adequate rather than aborting on noise. Any breach reverts traffic to the baseline automatically and alerts. The outcome is that a bad release affects around one per cent of users for fifteen minutes instead of everyone until somebody notices.

   </details>

6. **[Principal] How do canary releases fit alongside other deployment strategies?**

   <details><summary>Model answer</summary>

   They answer a different question from the others, which is why they compose rather than compete. Rolling deployment answers how to replace instances without downtime; blue-green answers how to make reversal instant and eliminate version coexistence; canary answers how many users encounter a fault before you know about it, and — uniquely — provides evidence rather than just limiting damage. That evidential quality is the part worth emphasising, because it changes deployment from an act of confidence into a measurement, and it catches a category of failure the others cannot: the release that is technically perfect and commercially harmful. In practice I would treat canary as the strategy for high-risk changes and combine it with the others — a canary phase within a blue-green environment gives limited exposure and instant reversal together, and a canary stage preceding a rolling completion gives measured risk with efficient finishing. The honest constraints are that it needs traffic volume to be statistically meaningful, it needs real comparison infrastructure that is genuine engineering work, and it is slow — so using it for every change would make deployment frequency the bottleneck. Reserving it for changes where the exposure question actually matters is the sensible position.

   </details>

---

### Feature flags

*Separate deploying code from enabling behaviour, so release becomes a runtime decision that can be targeted, measured and reversed instantly.*

**Flow:** `Deployed code` → `Flag evaluation` → `Targeted enablement` → `Instant disable` → `Flag removal`

> **The 30-second version**  
> Gate behaviour behind runtime conditions so code deploys inert and releases separately — with local evaluation, deterministic per-user bucketing, safe defaults, and a removal date set the day the flag is created.

**The problem**

A deployment and a release are treated as the same event, so shipping code means exposing behaviour. That couples engineering cadence to product readiness, forces long-lived branches for anything not ready, and makes reversal a redeployment — which takes minutes at best and is a larger operation than the situation usually warrants.

It also means every change is all-or-nothing for every user. There is no way to enable something for internal staff first, for one customer, or for ten per cent of traffic, without building and deploying a separate variant.

> **Decouple deployment from release and both become easier**  
> If behaviour is gated by a flag evaluated at runtime, code can be deployed continuously in an inert state and enabled independently — for particular users, a percentage of traffic, or everyone. Reversal becomes a configuration change taking seconds rather than a deployment, and the release decision moves to whoever owns the product rather than whoever owns the pipeline.

**Mental model**

Code paths are guarded by conditions evaluated at request time against a configuration that can change without deploying. The flag decides who sees what, and the decision can be changed instantly.

1. **Flag** — A named condition evaluated at runtime to choose a code path.
2. **Targeting** — Rules determining who gets which value — user, segment, percentage, environment.
3. **Evaluation** — Where and how the decision is made, which must be fast and failure-tolerant.
4. **Kill switch** — The ability to disable instantly, which is the primary operational value.
5. **Removal** — Deleting the flag and the dead branch once the decision is permanent.

> **Flags that are never removed become permanent complexity**  
> Every flag doubles the number of code paths, and combinations multiply: ten active flags is theoretically over a thousand configurations, most of which nobody has tested and some of which are impossible. Flags introduced for a rollout and left in place accumulate until the codebase is a maze of conditionals whose purpose nobody remembers, and removing them later is risky because nobody knows what depends on them.

**How it works**

**Flag types and their lifetimes**

```text
RELEASE FLAGS          lifetime: days to weeks
  gate an in-progress feature
  -> REMOVE once fully rolled out
  -> these are the ones that become debt

OPERATIONAL FLAGS      lifetime: permanent
  kill switches, load-shedding toggles, degradation
  controls
  -> legitimately long-lived
  -> but should be few, named clearly, and owned

PERMISSION FLAGS       lifetime: permanent
  entitlements by plan or tier
  -> arguably not flags at all; this is authorisation
  -> belongs in the permission model, not the flag
     system

EXPERIMENT FLAGS       lifetime: the experiment
  A/B variants
  -> REMOVE when the experiment concludes

THE DEBT COMES FROM ONE CATEGORY
  release flags that outlive their rollout
  -> they were temporary by intent and permanent in
     practice
  -> a removal deadline set at creation is the only
     reliable control

COMBINATORIAL REALITY
  n boolean flags = 2^n possible states
  10 flags = 1,024 combinations
  -> you test a handful
  -> production will find the others
```

1. **Set a removal date when the flag is created** — Release flags are temporary by intent and permanent by default; a deadline at creation is the only control that works.
2. **Default to the safe value on evaluation failure** — If the flag service is unreachable, the code must still make a sensible decision rather than erroring.
3. **Evaluate locally from a cached ruleset** — A network call per flag per request adds latency and a dependency to every code path.
4. **Keep entitlements out of the flag system** — Plan-based access is authorisation and belongs in the permission model, where it can be audited.
5. **Log which value was served** — Debugging is impossible without knowing which path a given request took.
6. **Limit concurrent release flags** — Combinations multiply beyond what anyone can test or reason about.

**Evaluation: latency, failure and consistency**

```text
NAIVE: call the flag service per evaluation
  + always current
  - network latency on every flagged path
  - the flag service becomes a hard dependency of
    every request
  -> an outage there becomes an outage everywhere

CORRECT: local evaluation from a cached ruleset
  rules streamed or polled into the process
  evaluation is a local function call
  + microseconds, no network on the request path
  + flag service outage -> keep using the last
    known ruleset
  - changes take seconds to propagate

FAILURE DEFAULTS ARE PART OF THE DESIGN
  every flag needs a defined value for:
    the service being unreachable at startup
    a rule that fails to parse
    an unknown flag name
  -> the safe default is usually OFF for new
     behaviour and ON for a kill switch's protected
     path
  -> "throws an exception" is not a default

CONSISTENCY FOR A GIVEN USER
  percentage rollouts must be STICKY
  -> hash the user id, not a random number
  -> otherwise the same user flips between variants
     on every request
  -> which breaks multi-step flows and makes
     experiment data meaningless
```

> **Percentage rollouts must be deterministic per user**  
> Evaluating a random number per request means a user at a ten per cent rollout sees the new behaviour on roughly one request in ten — flipping between variants mid-session, breaking multi-step flows, and making any measurement meaningless. Hashing a stable identifier gives the same user the same answer every time while still producing the intended distribution across the population.

**Worked example**

Rolling out a redesigned checkout behind a flag.

**Flag-driven rollout**

```text
FLAG  checkout-redesign
  created with an owner and a removal date 6 weeks out
  default if evaluation fails: OFF (old checkout)

TARGETING PROGRESSION
  week 1  internal employees only
          -> real usage, zero customer exposure
  week 2  1% of users, sticky by hashed user id
          -> same user always sees the same version
  week 3  10%, compare conversion against the rest
  week 4  50%
  week 5  100%
  week 6  remove the flag and delete the old code path

WHY STICKY MATTERS HERE
  checkout is multi-step
  -> a user flipping between designs mid-flow would
     see an inconsistent interface and likely abandon
  -> and conversion comparison would be meaningless

KILL SWITCH VALUE
  a payment edge case discovered at 10%
  -> flag set to OFF
  -> every user on the old checkout within seconds
  -> no deployment, no rollback, no pipeline
  -> fix, then resume the progression

EVALUATION
  ruleset cached in-process, refreshed every few
  seconds
  evaluation is a local function call
  -> the flag service being down does not affect
     serving

REMOVAL  (the step that is usually skipped)
  at 100% and stable, delete the flag AND the old
  branch
  -> without this, the codebase accumulates
     conditionals nobody can safely remove later
```

| Metric | Value | Note |
|---|---|---|
| Decouple | deploy ≠ release | product decides |
| Sticky | hashed user id | consistent experience |
| Kill switch | seconds | **no deployment** |
| Removal | scheduled at creation | prevents debt |

> **The kill switch is usually worth more than the gradual rollout**  
> Gradual exposure is valuable, but the ability to disable a behaviour in seconds without a deployment is what changes incident response. A problem discovered at ten per cent rollout goes from a deployment pipeline run under pressure to a configuration change — which means the mitigation is faster than diagnosis, and the decision can be made by whoever notices rather than whoever can deploy.

**When to use it**

- **Decoupling deployment from release**, so code ships continuously and behaviour is enabled separately.
- **Gradual rollouts** with targeting by user, segment or percentage.
- **Kill switches** for risky functionality, giving instant disablement.
- **Trunk-based development**, where incomplete work ships inert rather than living on a branch.
- **Experiments**, where variants must be assigned and measured.

**When to avoid it**

- **Do not use flags for entitlements**, which belong in the authorisation model where they can be audited.
- **Do not leave release flags in place**, since they become permanent untested complexity.
- **Do not evaluate flags over the network per request**, which adds latency and a dependency to every path.
- **Do not use random percentage evaluation**, which flips users between variants mid-session.
- **Do not accumulate many concurrent flags**, whose combinations exceed what can be tested.

**Advantages**

- **Instant enable and disable** without deployment.
- **Targeted exposure** by user, segment or percentage.
- **Release decisions move to product owners** rather than the deployment pipeline.
- **Supports trunk-based development**, avoiding long-lived branches.
- **Enables experimentation** with controlled variant assignment.
- **Mitigation faster than diagnosis**, since disabling precedes understanding.

**Disadvantages**

- **Code complexity grows** with every conditional path.
- **Combinations are untestable** beyond a small number of flags.
- **Flags accumulate** unless removal is enforced.
- **Another runtime dependency**, with its own failure modes.
- **Dead code persists** until the flag is removed.
- **Debugging requires knowing which values were served.**

**Trade-offs**

**Flag design trade-offs**

| Choice | Benefit | Cost |
|---|---|---|
| Local cached evaluation | Fast; tolerates flag service outage | Changes propagate in seconds, not instantly |
| Remote evaluation per request | Always current | Latency; hard dependency on every path |
| Sticky percentage by hash | Consistent per user; valid measurement | Cohort fixed by identifier |
| Random percentage | Simple | Users flip between variants |
| Long-lived flags | Flexibility retained | Permanent complexity |
| Enforced removal dates | Bounded complexity | Requires process discipline |

The evaluation choice is the one with operational consequences: fetching flag values remotely on every request makes the flag service a hard dependency of everything, so its outage becomes a total outage — precisely the coupling that flags were introduced to reduce.

**How it fails**

**Feature flag failures**

| Failure | Cause | Fix |
|---|---|---|
| Codebase full of untestable conditionals | Flags never removed | Removal date set at creation; enforce it |
| Flag service outage causes an outage | Remote evaluation per request | Local evaluation from a cached ruleset |
| Users flip between variants | Random percentage evaluation | Hash a stable user identifier |
| Unexpected behaviour in a combination | Untested flag interaction | Limit concurrent flags; test key combinations |
| Cannot reproduce a reported bug | Flag values not logged | Log evaluated values per request |
| Entitlement bypassed | Access control implemented as a flag | Authorisation belongs in the permission model |
| Errors when flags cannot be evaluated | No defined failure default | Every flag has a safe default value |

**Limits**

> **Practical guidance**
>
> - **Combinations**: n boolean flags give 2^n states; beyond a handful, most are untested.
> - **Removal**: release flags should have a deadline set at creation, typically weeks.
> - **Evaluation**: local and in-process, microseconds, with a cached ruleset refreshed periodically.
> - **Stickiness**: hash a stable identifier so a user's value is consistent.
> - **Defaults**: every flag needs a defined value for evaluation failure — usually off for new behaviour.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Feature flags | Decoupling release from deployment | Code complexity; flag debt |
| Branch-based development | Isolating incomplete work | Merge conflicts; long-lived divergence |
| Canary deployment | Risk-managed rollout | No per-user targeting; deployment-level |
| Blue-green | Fast environment switch | All-or-nothing; infrastructure cost |
| Configuration files | Simple environment differences | Requires restart or redeploy |
| Separate builds per variant | Complete isolation | Combinatorial build and deployment burden |

Canary deployment and feature flags overlap but differ in granularity and reversal cost: a canary controls which instances run new code, while a flag controls which users see new behaviour on the same instances — and a flag is disabled in seconds without touching the deployment at all.

**In real systems**

- **Trunk-based development with flags** lets incomplete work ship inert rather than diverging on long-lived branches.
- **Local evaluation from a streamed ruleset** keeps flag checks fast and prevents the flag service from becoming a hard dependency on the request path.
- **Deterministic bucketing by hashed identifier** gives consistent per-user assignment while producing the intended population distribution.
- **Kill switches around risky functionality** make mitigation a configuration change, which is faster than any deployment-based reversal.
- **Enforced flag removal deadlines** are the standard control against accumulating permanent conditional complexity.

**Common mistakes**

- **Flags never removed**, accumulating untestable conditional paths.
- **Remote evaluation per request**, making the flag service a universal dependency.
- **Random percentage assignment**, flipping users between variants.
- **No failure default**, so evaluation problems become errors.
- **Entitlements implemented as flags**, bypassing the authorisation model.
- **Flag values not logged**, making bug reports irreproducible.
- **Many concurrent flags**, whose combinations nobody has tested.

**The staff-level view**

Feature flags are adopted for the rollout control and owned for the complexity, and the balance depends almost entirely on removal discipline.

- **Require an owner and a removal date at creation.** Release flags are temporary by intent and permanent by default, and retrospective cleanup fails because nobody can establish what still depends on a flag.
- **Keep evaluation local.** Fetching flag values over the network on every request makes the flag service a hard dependency of every code path, so its outage is a total outage — the opposite of what flags are meant to provide.
- **Insist on deterministic bucketing.** Random per-request evaluation flips users between variants mid-flow, which breaks journeys and invalidates any measurement taken.
- **Separate entitlements from flags.** Plan-based access is authorisation, needs auditing, and does not belong in a system designed for rapid toggling by whoever is on call.
- **Value the kill switch explicitly.** The ability to disable in seconds without a deployment changes incident response more than gradual rollout does, because mitigation stops depending on diagnosis or on pipeline access.

**Go deeper**

Feature flags separate deploying code from enabling behaviour, so code ships continuously in an inert state and is turned on independently — for internal staff, a percentage of users, or everyone. Reversal becomes a configuration change taking seconds rather than a redeployment, and the release decision moves to whoever owns the product. The kill switch capability is usually worth more than the gradual rollout, because mitigation stops depending on a pipeline run under pressure.

Three implementation details matter. Evaluation should be local from a cached ruleset, because a network call per flag per request adds latency to every path and makes the flag service a hard dependency of the entire application. Percentage rollouts must hash a stable user identifier rather than using a random number, or users flip between variants mid-session and any measurement is meaningless. And every flag needs a defined value for evaluation failure, since an exception is not a default.

The long-term cost is complexity. Each flag doubles the state space and combinations quickly exceed what anyone tests, while release flags that were temporary by intent become permanent by default — and retrospective removal is risky because nobody can establish what still depends on them. An owner and a removal date recorded at creation is the only control that reliably works, along with keeping entitlements in the authorisation model where they belong.

Feature flags move the release decision from deployment time to runtime, which changes who can make it, how quickly it can be reversed, and how precisely it can be targeted.

**Decoupling deployment from release removes a coupling nobody chose.** Treating them as one event ties engineering cadence to product readiness, forces incomplete work onto long-lived branches, and makes reversal a redeployment. With behaviour gated at runtime, code ships continuously in an inert state, incomplete work merges to trunk safely, and enabling becomes a decision made by product owners rather than by whoever has pipeline access.

**The kill switch is the highest-value property.** Gradual exposure matters, but the ability to disable a behaviour in seconds without deploying anything transforms incident response: mitigation precedes diagnosis rather than waiting on it, and the action can be taken by whoever notices the problem. A fault found at partial rollout goes from an urgent pipeline run to a configuration change, which is both faster and much less likely to introduce a second problem.

**Evaluation must be local, or flags become a liability.** Calling a flag service per evaluation places network latency on every gated path and makes that service a hard dependency of the whole application — so its outage is a full outage, which is exactly the coupling flags were supposed to reduce. Streaming or polling the ruleset into the process reduces evaluation to a local function call and means a flag service failure simply leaves the last known configuration in effect. Every flag also needs a defined value for evaluation failure, because throwing an exception when a flag cannot be resolved converts a configuration problem into an application error.

**Bucketing must be deterministic per user.** A random draw per request means a user at a ten per cent rollout experiences the new behaviour intermittently, flipping between variants within a single session. Multi-step flows break, the interface appears inconsistent, and any comparison between cohorts is invalid because membership is not stable. Hashing a persistent identifier produces the intended population distribution while giving each individual a consistent experience.

**Complexity accumulates multiplicatively.** Each boolean flag doubles the number of possible configurations, so a codebase with a dozen live flags has thousands of theoretical states of which only a handful are ever exercised. Production consequently encounters combinations nobody anticipated, and reasoning about behaviour requires knowing runtime flag state that is not visible in the code. This argues for limiting how many release flags are live simultaneously, and for logging which values were served so that a reported problem can be reproduced at all.

**Removal has no natural forcing function, which is why it must be scheduled.** A release flag completes its rollout, works correctly, and breaks nothing — so cleanup competes with new work indefinitely and loses. By the time anyone attempts it, establishing what still depends on the flag and whether the old branch is genuinely dead has become its own investigation, so removal is deferred again and the conditional becomes permanent. Recording an owner and a removal date at creation, and treating the flag and its dead branch as deleted together, is the only approach that reliably prevents this. It helps to be strict about categories too: operational kill switches are legitimately permanent and should be few and clearly named, entitlements are authorisation and belong in a model that can be audited, and everything else is temporary and should be written with its deletion already scheduled.

**Prove it — interview questions**

1. **[Basic] What do feature flags actually decouple?**

   <details><summary>Model answer</summary>

   Deployment from release. Code can be shipped continuously in an inert state and the behaviour enabled separately — for internal staff, a percentage of users, or everyone — which means engineering cadence no longer depends on product readiness. It also moves the release decision to whoever owns the product rather than whoever can run the deployment pipeline, and makes reversal a configuration change of seconds rather than a redeployment.

   </details>

2. **[Basic] Why must percentage rollouts be deterministic?**

   <details><summary>Model answer</summary>

   Because evaluating a random number per request means the same user sees the new behaviour on roughly one request in ten at a ten per cent rollout, flipping between variants mid-session. That breaks multi-step flows, produces a visibly inconsistent interface, and makes any comparison between the groups meaningless since nobody is reliably in either. Hashing a stable user identifier gives each user the same answer every time while still producing the intended distribution across the population.

   </details>

3. **[Senior] Why is flag removal the critical discipline?**

   <details><summary>Model answer</summary>

   Because every flag adds a conditional path and the combinations multiply — ten boolean flags is over a thousand possible states, of which a handful get tested. Release flags are created as temporary and, without a forcing function, become permanent: the rollout finishes, attention moves on, and the conditional stays. Later removal is genuinely risky because nobody can establish what still depends on the flag or whether the old branch is truly dead. Setting an owner and a removal date at creation is the only control that reliably works, because it makes removal part of the original work rather than a separate future task nobody is assigned.

   </details>

4. **[Senior] Why should flag evaluation be local?**

   <details><summary>Model answer</summary>

   Because fetching a value over the network on every evaluation puts latency on every flagged code path and makes the flag service a hard dependency of everything — so its outage becomes an outage of your entire application, which is precisely the kind of coupling flags were introduced to reduce. Streaming or polling the ruleset into the process and evaluating locally makes each check a function call taking microseconds, and means a flag service outage simply leaves the last known ruleset in effect. The cost is that changes take seconds to propagate rather than being instantaneous, which is almost always an acceptable trade.

   </details>

5. **[Staff] Design the rollout of a redesigned checkout using flags.**

   <details><summary>Model answer</summary>

   One flag with a named owner and a removal date set at creation — six weeks out, say — and a defined default of off if evaluation fails, so an unreachable flag service means the old checkout rather than an error. Targeting progresses from internal employees, which gives real usage with zero customer exposure, through one, ten and fifty per cent to full rollout. Bucketing is deterministic by hashed user identifier, which matters particularly here because checkout is multi-step: a user flipping between designs mid-flow would see an inconsistent interface, probably abandon, and corrupt the conversion comparison that justifies the rollout. Evaluation is local from a cached ruleset refreshed every few seconds, so the flag service is never on the request path. The operational value I would emphasise is the kill switch: if a payment edge case surfaces at ten per cent, setting the flag off puts every user back on the old checkout within seconds, with no deployment and no pipeline — mitigation arrives before diagnosis, which is the right order. And the final step, which is the one teams skip, is deleting both the flag and the old code branch once the new version is stable at full rollout.

   </details>

6. **[Principal] What is the long-term cost of feature flags, and how do you manage it?**

   <details><summary>Model answer</summary>

   The cost is that they convert a deployment-time decision into a permanent runtime branch, and branches accumulate. Each flag doubles the state space, so a codebase with a dozen live flags has thousands of theoretical configurations of which a handful have ever been exercised — which means production regularly finds combinations nobody anticipated, and reasoning about behaviour requires knowing the flag state, which is not in the code. The compounding problem is that removal has no natural forcing function: the rollout completes, the flag works, nothing breaks, and the cleanup task competes with new work indefinitely. By the time someone attempts it, nobody can confidently say what depends on the flag or whether the old branch is dead, so removal is itself risky and gets deferred again. The only thing I have seen work is treating removal as part of the original change: an owner and a date recorded at creation, tracked the way any other commitment is, with the expectation that the flag and its dead branch are deleted together. It also helps to be strict about categories — operational kill switches are legitimately permanent and should be few and clearly named, entitlements belong in the authorisation model where they can be audited, and everything else is temporary and should be treated as such from the moment it is written.

   </details>

---

### Expand-contract schema migrations

*Change a schema through additive steps that keep old and new code working simultaneously, then remove the old shape only once nothing uses it.*

**Flow:** `Expand: add new shape` → `Dual write` → `Migrate and switch reads` → `Verify` → `Contract: remove old`

> **The 30-second version**  
> Add the new shape, write both, backfill, switch reads, stop writing the old — and only then, much later, drop it, so every intermediate state works with the code on either side.

**The problem**

A column needs renaming. Done in a single deployment, the migration runs and every instance still on the previous version immediately starts querying a column that no longer exists — so the service errors throughout the rollout window. Worse, the previous version can no longer function at all, which means rollback is impossible.

That second consequence is the serious one. Rollback is the primary mitigation for deployment incidents, so a change that removes it leaves only the option of fixing forward under time pressure, which is exactly the situation rollback exists to prevent.

> **Every intermediate state must work with the code on either side of it**  
> Because old and new versions run simultaneously during any rolling deployment, and because rollback means running the previous version against the current schema, each migration step has to be compatible both backwards and forwards. Splitting a change into additive expansion, a transition, and a later contraction is what makes every intermediate state safe.

**Mental model**

Rather than transforming the schema in place, add the new shape alongside the old, move data and readers across gradually, and remove the old shape only after nothing references it — typically several releases later.

1. **Expand** — Add the new column, table or field. Purely additive, so nothing breaks.
2. **Dual write** — Write both shapes, so either version's readers find current data.
3. **Migrate** — Backfill historical rows into the new shape.
4. **Switch reads** — Read from the new shape while still writing both, preserving rollback.
5. **Contract** — Stop writing the old shape and remove it, once nothing depends on it.

> **Contraction is the step that must not be rushed**  
> Removing the old column is irreversible in a way the other steps are not: once dropped, no previous version can function and the data may be gone. It should happen only when the code writing it has been fully deployed and stable for long enough that rollback to any version still in play is unnecessary — which is usually longer than it feels, because a release from two weeks ago may still be the rollback target.

**How it works**

**The five steps and what each preserves**

```text
GOAL  rename  user_name  ->  full_name

STEP 1  EXPAND  (release 1)
  ALTER TABLE add full_name NULL
  code writes BOTH user_name and full_name
  code reads user_name
  -> old instances unaffected: they see an extra
     nullable column they ignore
  -> rollback safe

STEP 2  BACKFILL  (no release)
  UPDATE ... SET full_name = user_name
  in batches, throttled
  -> does not block; does not change behaviour

STEP 3  SWITCH READS  (release 2)
  code reads full_name
  code still writes BOTH
  -> rollback to release 1 works, because release 1
     reads user_name, which is still being written
  -> THIS is why dual write continues here

STEP 4  STOP DUAL WRITE  (release 3)
  code writes only full_name
  -> rollback to release 2 works: it reads full_name
  -> rollback to release 1 does NOT, so release 2
     must be stable everywhere first

STEP 5  CONTRACT  (later, separate change)
  ALTER TABLE drop user_name
  -> irreversible
  -> only when no deployable version reads or writes
     it

THREE RELEASES AND A DROP, FOR A RENAME.
The alternative is an outage during rollout and no
rollback.
```

1. **Make every step additive or removal-only** — A step that both adds and removes cannot be compatible in both directions.
2. **Keep dual writes until reads have moved and stabilised** — Dual writing is what allows rollback to a version that reads the old shape.
3. **Backfill in throttled batches** — A single large update locks rows, blocks writers and can exhaust replication capacity.
4. **Verify before contracting** — Confirm the new shape is complete and correct, since contraction destroys the fallback.
5. **Separate the contraction into its own change** — Bundling it with other work makes it harder to reason about and to defer safely.
6. **Consider constraints and indexes separately** — Adding a NOT NULL constraint or a unique index has its own locking and compatibility implications.

**Where migrations cause outages**

```text
LOCKING
  ALTER TABLE on a large table can hold a lock for
  minutes
  -> every query on that table blocks
  -> connection pool fills; the service is down
  SAFE: add nullable columns (usually cheap),
    create indexes concurrently where supported,
    avoid rewriting table data during a deploy

NOT NULL WITH A DEFAULT
  on some systems this rewrites the whole table
  -> add nullable, backfill, then add the constraint
     as a separate validated step

BACKFILL LOAD
  UPDATE over millions of rows in one statement
  -> long transaction, replication lag, lock
     escalation
  SAFE: batches of a few thousand with pauses,
    monitored against replication lag

THE ROLLBACK TRAP
  migration applied, code deployed, problem found
  -> code rolls back in minutes
  -> the schema does NOT
  -> so the previous code must work against the
     NEW schema
  -> which is exactly what expand-contract
     guarantees and what a one-step rename destroys

ORDERING RULE
  schema changes go FIRST and must be compatible with
  the currently running code
  -> never deploy code that requires a schema change
     that has not yet been applied
```

> **Schema changes do not roll back with the code**  
> A deployment rollback restores the application in minutes; the migration that ran alongside it does not revert. This asymmetry is the core reason for expand-contract: the previous version of the code must be able to operate against the schema as it now stands. Any migration that breaks that property has silently removed rollback from the deployment it accompanies.

**Worked example**

Splitting a single name column into first and last names across a live service.

**Migration plan**

```text
STARTING POINT  users.full_name, 50 million rows

RELEASE 1  EXPAND
  add first_name NULL, last_name NULL
  code writes full_name AND the split pair
  code reads full_name
  -> old instances ignore the new columns
  -> deploy is safe; rollback is safe

BACKFILL  (background, days)
  batches of 5,000 rows, pause between batches
  monitored against replication lag
  -> if lag rises, slow down
  -> 50 million rows at this rate takes time, and
     that is fine because nothing waits on it

VERIFY
  count rows where first_name IS NULL
  spot-check parsing correctness on a sample
  -> the split is lossy for some names, so this step
     finds the cases needing manual handling
  -> contraction must not happen before this is
     resolved

RELEASE 2  SWITCH READS
  code reads first_name and last_name
  still writes all three
  -> rollback to release 1 works: it reads full_name,
     which is still written

RELEASE 3  STOP DUAL WRITE
  writes only first_name and last_name
  -> only after release 2 is stable everywhere

LATER  CONTRACT
  drop full_name
  -> weeks later, once no deployable version needs it
  -> and after confirming no reporting query, export
     or downstream consumer reads it

DOWNSTREAM CONSUMERS ARE THE USUAL SURPRISE
  analytics jobs, exports and reports also read this
  column and are not in the deployment pipeline
```

| Metric | Value | Note |
|---|---|---|
| Releases | 3 + a later drop | for one change |
| Backfill | 5,000-row batches | watched against lag |
| Rollback | preserved throughout | **the point** |
| Contract | weeks later | irreversible |

> **Downstream consumers are the step teams forget**  
> Application instances are covered by the deployment pipeline, but analytics jobs, scheduled exports, reporting queries and other services frequently read the same tables and are not deployed alongside. Contracting a schema without auditing those breaks things that will not be noticed until a report runs at the end of the month, by which time the connection to the migration is far from obvious.

**When to use it**

- **Any schema change on a system deployed without downtime**, which is the normal case.
- **Rolling or canary deployments**, where old and new code necessarily coexist.
- **Where rollback must remain available**, which is essentially always.
- **Large tables**, where in-place transformation would lock for an unacceptable period.
- **Shared databases with multiple consumers**, where readers extend beyond one application.

**When to avoid it**

- **Do not combine expansion and contraction in one release**, which breaks compatibility in one direction.
- **Do not contract while any deployable version still needs the old shape.**
- **Do not backfill in a single statement**, which locks and lags.
- **Do not deploy code requiring a schema change before that change is applied.**
- **Do not assume application code is the only reader**, since analytics and exports frequently are too.

**Advantages**

- **No downtime**, since every step is compatible with running code.
- **Rollback preserved** at every stage until contraction.
- **Safe under rolling deployment**, where both versions coexist by design.
- **Backfill decoupled from deployment**, so it can be slow and throttled.
- **Verification before destruction**, since the old shape remains until confirmed unnecessary.

**Disadvantages**

- **Several releases for one logical change**, which feels slow.
- **Dual writes add complexity** and a window for inconsistency between the shapes.
- **Storage duplicated** while both shapes exist.
- **Contraction is frequently forgotten**, leaving dead columns indefinitely.
- **Requires coordination** across teams when consumers are external to the service.

**Trade-offs**

**Migration approach trade-offs**

| Approach | Downtime | Rollback | Effort |
|---|---|---|---|
| Single-step rename | Errors during rollout | Impossible | Minimal |
| Expand-contract | None | Preserved until contraction | Three releases |
| Maintenance window | Planned downtime | Restore from backup | Low, if downtime is acceptable |
| Shadow table with switchover | None | Possible | High complexity |
| Never contract | None | Preserved | Permanent schema clutter |

The last row is a real and common outcome rather than a strategy: teams complete the expansion, get the benefit, and never perform the contraction — leaving columns that nothing reads and nobody dares remove. Scheduling the contraction as a tracked follow-up at the time of expansion is what prevents it.

**How it fails**

**Migration failures**

| Failure | Cause | Fix |
|---|---|---|
| Errors throughout the rollout | Single-step incompatible change | Expand-contract across releases |
| Rollback impossible after a bad release | Schema no longer supports the previous code | Keep every step compatible in both directions |
| Service down during migration | Long lock on a large table | Additive changes; concurrent index creation; avoid rewrites |
| Replication lag spike | Backfill in one large statement | Throttled batches monitored against lag |
| Data missing after contraction | Backfill incomplete or incorrect | Verify completeness before dropping anything |
| Reports break weeks later | Downstream consumers not audited | Enumerate all readers before contracting |
| Schema full of unused columns | Contraction never performed | Schedule contraction as a tracked follow-up |

**Limits**

> **Practical guidance**
>
> - **Releases**: typically three plus a later contraction for a single logical change.
> - **Backfill batches**: thousands of rows with pauses, monitored against replication lag.
> - **Contraction timing**: after the dual-write removal is stable everywhere, often weeks later.
> - **Locking**: adding a nullable column is usually cheap; adding constraints or rewriting data is not.
> - **Consumers**: application instances plus analytics, exports and any other service reading the table.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Expand-contract | Zero-downtime schema evolution | Multiple releases; dual-write complexity |
| Maintenance window | Systems tolerating planned downtime | Downtime; risky if it overruns |
| Schema versioning in the application | Rapidly evolving shapes | Application complexity |
| Schemaless storage | Highly variable data | Validation moves into application code |
| Shadow table and switchover | Large restructuring | Substantial complexity and synchronisation |
| Event-sourced rebuild | Complete reshaping | Requires an event log; expensive |

A maintenance window remains a legitimate choice for systems where planned downtime is acceptable, and it is dramatically simpler. The risk is that windows overrun and that the rollback plan becomes restoring from backup, which is a much larger operation than redeploying a previous version.

**In real systems**

- **Expand-contract is the standard pattern** for zero-downtime schema evolution, precisely because rolling deployments guarantee version coexistence.
- **Throttled batch backfills monitored against replication lag** avoid the long transactions and lock escalation that single-statement updates cause.
- **Concurrent index creation** allows adding indexes on large tables without blocking writes.
- **Dual writing maintained until reads have stabilised** is what preserves rollback to the version that still reads the old shape.
- **Contraction tracked as a scheduled follow-up** is how teams avoid accumulating columns nothing reads and nobody will remove.

**Common mistakes**

- **Single-release renames**, breaking coexisting versions and eliminating rollback.
- **Dropping dual writes before reads have stabilised**, breaking rollback to the previous release.
- **Backfilling in one statement**, locking rows and causing replication lag.
- **Contracting too early**, while a deployable version still needs the old shape.
- **Ignoring downstream consumers**, breaking reports and exports silently.
- **Deploying code before its migration**, causing errors until the migration runs.
- **Never contracting**, leaving a schema cluttered with unused columns.

**The staff-level view**

Schema changes are where deployment safety is most often quietly lost, because the incompatibility is invisible until the rollout is under way.

- **Review every migration for rollback compatibility.** The code reverts in minutes and the schema does not, so the previous version must work against the new schema — and a migration that breaks this has silently removed rollback from that deployment.
- **Require expansion and contraction to be separate changes.** Combining them produces a step that is incompatible in one direction, which is the entire failure the pattern exists to avoid.
- **Audit downstream readers before contracting.** Analytics jobs, exports and other services read the same tables and are outside the deployment pipeline, so they break silently and are discovered weeks later.
- **Insist on throttled backfills.** A single large update is the most common way a migration takes a service down, through locking and replication lag rather than through anything schema-related.
- **Track the contraction.** Left unscheduled it never happens, and the resulting accumulation of unused columns eventually becomes a schema nobody is willing to change.

**Go deeper**

A schema change deployed in one step breaks the versions running alongside it during a rolling deployment, and — more seriously — makes rollback impossible, because the previous code cannot operate against the new schema. Expand-contract splits the change so every intermediate state is compatible in both directions: add the new shape, write both, backfill, switch reads, stop writing the old, and drop it only much later.

The ordering detail that matters most is continuing to dual-write after reads have switched. At that point the deployed code reads the new shape while the previous release still reads the old one, so maintaining both writes is what keeps rollback viable. Dual writing stops only once the read-switch release is stable everywhere and is no longer a rollback target.

Two practical hazards sit outside the schema itself. Backfills run as single large statements lock rows, block writers and generate replication lag — batched and throttled updates avoid this, and since the backfill is decoupled from deployment it can take days. And contraction requires auditing every reader, because analytics jobs, exports and other services read the same tables without being in the deployment pipeline, so they break silently and surface much later.

Expand-contract makes schema evolution safe by ensuring that every intermediate state is compatible with the application versions that might be running against it.

**The asymmetry between code and schema rollback is the whole motivation.** A deployment rollback restores the application in minutes; the migration that accompanied it does not revert. So the previous version of the code must be able to operate against the schema as it now stands, or rollback has been silently removed from that deployment. Since rollback is the primary mitigation for most deployment incidents, a migration that breaks this property converts an ordinary bad release into an incident requiring a forward fix under time pressure.

**Coexistence makes the requirement bidirectional.** During any rolling deployment, old and new application versions serve traffic simultaneously against the same database, so the schema must support both at once — not merely the new one. Combined with the rollback requirement, this means each step must work with the release before it and the release after it, which is precisely what confining every step to pure addition or pure removal achieves.

**Dual writing is the mechanism that preserves rollback across the read switch.** After reads move to the new shape, the currently deployed code uses it while the previous release still uses the old one. Continuing to write both means a rollback finds current data in the place it expects. Stopping dual writes prematurely is the subtle error here, because everything appears to work — the deployed version is reading the correct column — right up until someone needs to revert.

**Most migration outages are caused by locking and load, not by incompatibility.** Altering a large table can hold a lock long enough to fill the connection pool and take the service down; adding a constraint may rewrite the entire table on some systems; a backfill issued as one statement holds a long transaction, escalates locks and generates replication lag that breaks read replicas. Additive nullable columns, concurrent index creation, and backfills issued in modest batches with pauses and lag monitoring avoid all of these — and since backfill is decoupled from deployment, taking days is perfectly acceptable.

**Contraction is irreversible and therefore deserves the most caution.** Dropping the old column means no earlier version can function and the data may be unrecoverable, so it should happen only when no deployable version reads or writes it — which is generally weeks after the dual-write removal, because a release from a fortnight ago may still be a plausible rollback target. Verification of the new shape's completeness and correctness must precede it, particularly where the transformation is lossy and some rows need manual attention.

**Downstream consumers are the reliably forgotten dimension.** Application instances are governed by the deployment pipeline, but analytics jobs, scheduled exports, reporting queries and other services frequently read the same tables and are not. Contracting without enumerating them breaks things that produce no immediate error and surface at the end of a reporting period, by which point the connection to a migration performed weeks earlier is far from obvious. The related organisational failure is never contracting at all — the expansion delivers the benefit, attention moves on, and the schema accumulates columns nothing reads and nobody is willing to remove. Scheduling the contraction as a tracked item at the time of expansion is what prevents both.

**Prove it — interview questions**

1. **[Basic] What is expand-contract?**

   <details><summary>Model answer</summary>

   Splitting a schema change into additive and subtractive phases separated by several releases. First expand — add the new column while the old one remains, and write both. Then migrate data and switch reads to the new shape while still writing both. Then stop writing the old shape. Only much later contract, by dropping it. Each intermediate state works with the code on either side of it, which is what allows the change to happen without downtime and without losing rollback.

   </details>

2. **[Basic] Why can't you just rename the column?**

   <details><summary>Model answer</summary>

   Because during a rolling deployment both versions run simultaneously, so instances still on the previous release immediately begin querying a column that no longer exists — errors for the entire rollout window. And more seriously, the previous version can no longer function against the new schema at all, which means rollback is impossible. Since rollback is the primary mitigation for deployment incidents, a single-step rename removes the safety net at exactly the moment it might be needed.

   </details>

3. **[Senior] Why continue dual writing after reads have switched?**

   <details><summary>Model answer</summary>

   To preserve rollback to the release that still reads the old shape. After the read switch, the deployed code reads the new column but the previous release reads the old one — so if you stop writing the old column at the same time, rolling back gives you code reading a column that has stopped receiving current data. Continuing to write both means the previous release still finds correct values. Dual writing can stop only once the read-switch release is stable everywhere and no longer a rollback target.

   </details>

4. **[Senior] Why are backfills dangerous?**

   <details><summary>Model answer</summary>

   Because they are the part of a migration most likely to take the service down, and not for schema reasons. A single statement updating millions of rows holds a long transaction, escalates locks, blocks writers on the same table, and generates enough replication traffic to push replicas into significant lag — which can break read paths that depend on them. Batched updates of a few thousand rows with pauses, monitored against replication lag and slowed when it rises, avoid all of this. The backfill is also decoupled from deployment, so it can take days without anything waiting on it.

   </details>

5. **[Staff] Walk through splitting a name column into first and last names on a fifty-million-row table.**

   <details><summary>Model answer</summary>

   Release one expands: add both new columns as nullable and have the code write all three, while still reading the original. Old instances simply ignore two extra nullable columns, so the deploy and its rollback are both safe. Then a background backfill in batches of a few thousand with pauses, watched against replication lag and throttled if it rises — fifty million rows will take days and nothing is waiting on it. Then verification, which matters particularly here because splitting names is lossy for some inputs, so this step finds the cases needing manual handling before anything irreversible happens. Release two switches reads to the new columns while continuing to write all three, which keeps rollback to release one working because it reads the original column that is still being maintained. Release three stops writing the original, but only once release two is stable everywhere and is no longer a rollback target. The contraction — dropping the original column — comes weeks later as its own change, and before it I would audit every reader, because analytics jobs, scheduled exports and other services read this table and are not in the deployment pipeline, so they break silently and surface at month end when the connection to the migration is no longer obvious.

   </details>

6. **[Principal] Why is this discipline so often skipped, and what is the real cost?**

   <details><summary>Model answer</summary>

   It is skipped because the fast path works most of the time. A single-release rename deploys successfully in a quiet period, the rollout is quick enough that the window of incompatibility passes unnoticed, and nothing needs rolling back — so the shortcut is validated by experience, repeatedly, until the release that does need reverting. At that point the cost is not a slightly worse incident but a qualitatively different one: the ordinary mitigation is unavailable, the only path is forward under time pressure, and the team is writing a fix and a migration during an outage rather than pressing a button. The second, quieter cost is that this is invisible in review unless someone is specifically looking for it — the migration looks correct, the code looks correct, and the incompatibility exists only in the combination of old code with new schema, which no test exercises. So the intervention I would insist on is a specific review question rather than general awareness: for every migration, can the currently deployed version run against this schema. That single check catches nearly all of it, and it reframes the discipline from bureaucracy into what it actually is, which is preserving the ability to undo a deployment.

   </details>

---

### Autoscaling control and hysteresis

*Adjust capacity from a signal that reflects real load, with asymmetric thresholds and cooldowns so the system converges rather than oscillating.*

**Flow:** `Load signal` → `Scaling decision` → `Cooldown` → `Stabilised capacity` → `Damped response`

> **The 30-second version**  
> Scale on a signal reflecting the real constraint, with separate up and down thresholds, asymmetric speeds and cooldowns, explicit bounds, and scheduled scaling for predictable ramps — keeping headroom for spikes scaling cannot catch.

**The problem**

A service scales on CPU with a threshold of seventy per cent. Load rises, instances are added, average CPU falls below the scale-down threshold, instances are removed, CPU rises again — and the system spends its time adding and removing capacity rather than settling. Each cycle costs startup time, cold caches and connection churn.

The opposite failure is equally common: a scaler tuned conservatively enough not to oscillate reacts so slowly that a traffic surge is over before capacity arrives, which means the autoscaling provided no benefit during the only period it mattered.

> **Autoscaling is a control loop, and control loops need damping**  
> Scaling on a signal without hysteresis produces oscillation, because the action taken changes the signal that triggered it. Asymmetric thresholds, cooldown periods and step limits are what convert a reactive rule into a system that converges — and the asymmetry should favour scaling up quickly and down slowly, because under-capacity harms users while over-capacity only costs money.

**Mental model**

A controller observes a load signal, compares it against thresholds, and changes the instance count — then waits before acting again, because the effect of its action takes time to appear.

1. **Signal** — The metric driving decisions, which must reflect actual load on the constraining resource.
2. **Thresholds** — Separate levels for scaling up and down, with a gap between them.
3. **Cooldown** — A waiting period after acting, so the effect is observed before deciding again.
4. **Step size** — How much capacity changes per decision — larger for emergencies, smaller for adjustment.
5. **Bounds** — Minimum and maximum instance counts, which protect against both failure modes.

> **Scaling on CPU when the constraint is elsewhere produces a scaler that never helps**  
> Services are frequently limited by connection pools, downstream capacity, lock contention or memory rather than CPU. A scaler watching CPU on such a service adds no instances while latency climbs, because CPU never reaches the threshold — and if it does eventually scale, adding instances may worsen the actual constraint by increasing pressure on the shared dependency.

**How it works**

**Hysteresis: why a single threshold oscillates**

```text
SINGLE THRESHOLD AT 70%
  load rises -> CPU 75% -> add instance
  -> CPU drops to 65% -> below 70% -> remove instance
  -> CPU back to 75% -> add instance
  -> oscillation, indefinitely
  each cycle costs: startup time, cold cache,
  connection churn, and often a brief latency spike

ASYMMETRIC THRESHOLDS  (hysteresis)
  scale UP   when above 70%
  scale DOWN when below 40%
  -> between 40% and 70% nothing happens
  -> the gap absorbs the effect of the scaler's own
     action
  -> the wider the gap, the more stable and the less
     efficient

COOLDOWN PERIODS
  after scaling up:   wait ~3 min before deciding again
  after scaling down: wait ~10 min
  -> new instances need time to warm and take load
  -> deciding before the effect appears means deciding
     on stale information

ASYMMETRY IN BOTH DIRECTIONS
  up:   fast, aggressive, large steps
        -> under-capacity harms users
  down: slow, cautious, one instance at a time
        -> over-capacity only costs money
  -> the costs are not symmetric, so the response
     should not be either
```

1. **Scale on a signal that reflects the real constraint** — Request concurrency or queue depth usually beats CPU, because most services saturate somewhere other than the processor.
2. **Use separate up and down thresholds** — A single threshold guarantees oscillation, because scaling changes the signal that triggered it.
3. **Apply cooldowns, longer for scale-down** — Acting before the previous action's effect is visible means deciding on stale information.
4. **Scale up fast and down slowly** — The two errors have different costs, so the responses should differ.
5. **Set minimum and maximum bounds** — The minimum protects against scaling into unavailability; the maximum protects against runaway cost and against overwhelming dependencies.
6. **Pre-scale for predictable events** — Reactive scaling always lags; known peaks should be anticipated rather than discovered.

**Why reactive scaling is always late**

```text
TIMELINE OF A SCALE-UP
  t+0     load increases
  t+60    metric window reflects it
  t+60    scaler decides, requests an instance
  t+90    instance scheduled and starting
  t+150   application started
  t+180   caches warm, connections established,
          actually useful
  -> ~3 MINUTES from load increase to added capacity

CONSEQUENCE
  a 2-minute traffic spike is over before capacity
  arrives
  -> autoscaling did nothing except add cost afterwards
  -> surges must be absorbed by HEADROOM, not by
     scaling

SO AUTOSCALING IS FOR:
  sustained load changes over tens of minutes
  daily and weekly patterns
  gradual growth
NOT FOR:
  sudden spikes
  -> those need spare capacity and load shedding

SCHEDULED SCALING BEATS REACTIVE FOR KNOWN PATTERNS
  traffic reliably rises at 09:00
  -> scale up at 08:45
  -> capacity is warm when it is needed
  -> reactive scaling would still be catching up at
     09:05

THE DEPENDENCY TRAP
  scaling the application up increases load on the
  database
  -> if the database is the constraint, more instances
     make it worse
  -> the maximum bound exists partly to prevent this
```

> **Scaling up can overwhelm the thing that was already struggling**  
> Adding instances increases the number of connections to the database, the request rate to downstream services, and the load on shared dependencies. If the bottleneck is one of those rather than the application tier, scaling amplifies the problem — and a scaler responding to rising latency by adding capacity can drive a struggling dependency into complete failure. A maximum bound is partly a cost control and partly protection against exactly this.

**Worked example**

Autoscaling configuration for an API service with a daily traffic pattern.

**Configuration and behaviour**

```text
SIGNAL  requests in flight per instance
  not CPU: this service is I/O bound and saturates on
  the database connection pool at ~45% CPU
  -> concurrency directly reflects the real constraint

THRESHOLDS  (hysteresis)
  scale up   above 80 concurrent per instance
  scale down below 40 concurrent per instance
  -> a wide dead band, deliberately

STEPS AND COOLDOWNS
  up:   add 20% of current capacity, minimum 2
        cooldown 3 min
  down: remove 1 instance
        cooldown 10 min
  -> up is fast and chunky; down is slow and single

BOUNDS
  minimum 6   (survives losing an availability zone)
  maximum 40  (beyond this the database pool saturates,
               so more instances would make things
               worse, not better)

SCHEDULED SCALING
  08:45 raise the minimum to 20
  18:30 return the minimum to 6
  -> reactive scaling cannot react fast enough to the
     morning ramp
  -> the schedule handles the predictable part;
     reactive handles the variance

HEADROOM FOR SPIKES
  target ~60% utilisation at steady state
  -> absorbs a sudden surge for the ~3 minutes
     scaling takes
  -> combined with load shedding beyond that

RESULT
  stable capacity, no oscillation, predictable cost,
  and surges absorbed rather than chased
```

| Metric | Value | Note |
|---|---|---|
| Signal | concurrency | **not CPU** |
| Dead band | 40–80 | no oscillation |
| Up / down | fast / slow | asymmetric cost |
| Spikes | headroom | scaling is too slow |

> **Scheduled scaling handles the predictable part; reactive handles the rest**  
> Most traffic patterns are substantially predictable — daily ramps, weekly cycles, known campaigns — and reactive scaling always lags because the loop takes minutes. Raising the floor ahead of a known ramp means capacity is warm when the load arrives, leaving the reactive scaler to handle only the variance around the pattern. This combination is consistently better than tuning a reactive scaler to be aggressive enough to chase a predictable curve.

**When to use it**

- **Workloads with sustained variation** over tens of minutes or longer.
- **Predictable daily or weekly patterns**, best handled with scheduled scaling.
- **Cost optimisation**, where running peak capacity continuously is wasteful.
- **Stateless services**, where instances can be added and removed freely.
- **Queue-driven workers**, where queue depth is a direct and honest signal.

**When to avoid it**

- **Do not rely on autoscaling for sudden spikes**, which are over before capacity arrives.
- **Do not scale on CPU** unless CPU is genuinely the constraint.
- **Do not use a single threshold**, which guarantees oscillation.
- **Do not scale a tier whose bottleneck is downstream**, which amplifies the real problem.
- **Do not autoscale stateful services** without explicit handling of data and membership.

**Advantages**

- **Capacity tracks demand**, reducing cost during quiet periods.
- **Handles sustained growth** without manual intervention.
- **Hysteresis produces stable convergence** rather than oscillation.
- **Asymmetric response** matches the asymmetric cost of the two errors.
- **Scheduled scaling anticipates** patterns that reactive scaling cannot catch.
- **Bounds protect** against both unavailability and runaway cost.

**Disadvantages**

- **Always lags**, since the control loop takes minutes end to end.
- **Oscillates when tuned badly**, with real cost per cycle.
- **Can amplify downstream problems** by increasing pressure on a struggling dependency.
- **Cold instances underperform** immediately after starting.
- **Signal choice is subtle** and frequently wrong.
- **Unsuitable for spikes**, which require headroom instead.

**Trade-offs**

**Tuning trade-offs**

| Setting | Aggressive | Conservative | Reasonable |
|---|---|---|---|
| Threshold gap | Efficient; oscillates | Stable; wasteful | Wide enough to absorb one step |
| Scale-up speed | Handles ramps; may overshoot | Under-capacity during growth | Fast, proportional steps |
| Scale-down speed | Saves cost; risks flapping | Costs money | Slow, single steps |
| Cooldown | Responsive; acts on stale data | Stable; slow | Longer for scale-down |
| Minimum bound | Cheap; fragile | Costly; resilient | Survives losing a failure domain |

The asymmetry between scale-up and scale-down settings is the most important tuning principle, and it follows directly from the fact that insufficient capacity harms users while excess capacity merely costs money — so the two directions should never be configured symmetrically.

**How it fails**

**Autoscaling failures**

| Failure | Cause | Fix |
|---|---|---|
| Constant scaling in and out | Single threshold; no hysteresis | Separate up and down thresholds with a gap |
| Capacity arrives after the spike | Reactive scaling latency | Headroom and load shedding for spikes |
| Scaler never triggers while latency climbs | Scaling on CPU when the constraint is elsewhere | Scale on concurrency or queue depth |
| Database overwhelmed as instances scale | Scaling amplified pressure on the real bottleneck | Maximum bound; address the actual constraint |
| Scaled to zero and cannot recover | Minimum bound too low | Minimum sized to survive a failure domain loss |
| Cost spiralled | No maximum bound during an incident | Maximum bound with alerting |
| New instances degrade latency | Cold caches and connections taking load immediately | Gradual traffic ramp for new instances |

**Limits**

> **Tuning reference**
>
> - **End-to-end scale-up latency**: typically 2–5 minutes from load increase to useful capacity.
> - **Threshold gap**: wide enough that one scaling step does not cross back over the other threshold.
> - **Cooldowns**: minutes for scale-up, substantially longer for scale-down.
> - **Steady-state utilisation**: around 60% leaves headroom to absorb surges while scaling catches up.
> - **Minimum bound**: sized to survive losing an availability zone, not merely to serve average load.

**Alternatives**

| Approach | Best for | Trade |
|---|---|---|
| Reactive autoscaling | Sustained load variation | Always lags; needs careful tuning |
| Scheduled scaling | Predictable patterns | Does not handle unexpected load |
| Predictive scaling | Learnable patterns | Wrong when behaviour changes |
| Fixed over-provisioning | Spiky, unpredictable load | Expensive; still bounded |
| Load shedding | Surges beyond capacity | Refuses some users |
| Serverless | Highly variable workloads | Cold starts; platform constraints |

Autoscaling and load shedding are complements addressing different timescales: shedding protects the system in the seconds before capacity can change, while scaling adjusts over minutes. A system relying on scaling alone will collapse during the interval before instances arrive, which is precisely when a surge does its damage.

**In real systems**

- **Concurrency and queue-depth signals** have largely replaced CPU for scaling decisions, because most services saturate on something other than the processor.
- **Asymmetric scale-up and scale-down configuration** is standard, reflecting that under-capacity harms users while over-capacity only costs money.
- **Scheduled scaling ahead of known ramps** compensates for reactive scaling's inherent lag on predictable patterns.
- **Maximum instance bounds** protect downstream dependencies from being overwhelmed as the application tier grows.
- **Gradual traffic ramping for new instances** avoids cold caches and unestablished connections degrading latency on arrival.

**Common mistakes**

- **Scaling on CPU** when the constraint is a connection pool or dependency.
- **A single threshold**, guaranteeing oscillation.
- **Symmetric up and down behaviour**, ignoring the asymmetric cost.
- **Expecting autoscaling to handle spikes**, which it cannot.
- **No maximum bound**, allowing runaway cost and downstream overload.
- **Minimum bound too low** to survive a failure domain loss.
- **New instances taking full traffic immediately**, degrading latency while cold.

**The staff-level view**

Autoscaling is usually configured once from defaults and then blamed for behaviour those defaults guarantee.

- **Check what the service actually saturates on.** CPU-based scaling on an I/O-bound service never triggers while latency climbs, which is the most common reason a scaler appears to do nothing useful.
- **Require asymmetric configuration.** Symmetric thresholds and cooldowns guarantee oscillation and misprice the two failure modes, since insufficient capacity harms users and excess capacity merely costs money.
- **Be explicit that scaling cannot handle spikes.** The loop takes minutes end to end, so surges must be absorbed by headroom and load shedding — expecting otherwise leads to tuning a scaler ever more aggressively for a problem it cannot solve.
- **Use scheduled scaling for predictable patterns.** Most traffic is substantially forecastable, and raising the floor ahead of a known ramp beats any reactive tuning.
- **Set a maximum bound deliberately.** It is both a cost control and protection against the application tier scaling up into a downstream dependency that was the real constraint all along.

**Go deeper**

Autoscaling is a control loop, and like any control loop it needs damping. A single threshold guarantees oscillation because adding capacity changes the metric that triggered the addition, which then triggers a removal. Separate scale-up and scale-down thresholds with a gap between them absorb the effect of the scaler's own action, and cooldown periods prevent decisions being made before the previous action's effect is visible.

The signal matters more than the tuning. Most services saturate on connection pools, downstream capacity or lock contention rather than CPU, so a CPU-based scaler sits idle while latency climbs — and if it does fire, adding instances increases pressure on the dependency that was the actual constraint. Concurrency and queue depth reflect work waiting rather than processor use, and are usually the better choice.

Two structural limits should be stated explicitly. Scaling cannot handle spikes, because the loop takes minutes from load increase to warm capacity, so surges must be absorbed by headroom and load shedding. And predictable patterns are better served by scheduled scaling that raises the floor before a known ramp, leaving the reactive scaler to handle only the variance around it.

Autoscaling adjusts capacity to demand, and its difficulties are those of any feedback control system acting on a signal its own actions change.

**Hysteresis is what makes it converge.** With a single threshold, the scaler's action moves the metric across that threshold in the opposite direction, triggering the inverse action indefinitely. Each oscillation costs instance startup time, cold caches, connection churn and usually a brief latency disturbance. Separate thresholds with a dead band between them, sized so that one scaling step cannot cross from one to the other, eliminate this — and cooldown periods ensure decisions are made on observed effects rather than on stale data from before the last action took hold.

**The asymmetry of costs should be reflected in the configuration.** Insufficient capacity degrades service or fails requests; excess capacity costs money. Those are not equivalent, so scaling up should be fast, proportional and on a short cooldown, while scaling down is slow, incremental and patient. Symmetric configuration misprices the failure modes and is also more prone to flapping, since aggressive scale-down keeps returning the system to the edge of insufficiency.

**Signal choice is more consequential than threshold tuning.** Services commonly saturate on connection pools, downstream dependencies, lock contention or memory rather than CPU, so a CPU-driven scaler can remain inactive while latency climbs steadily — and the default configuration in most platforms is exactly that. Concurrency of in-flight requests and queue depth measure work waiting rather than processor consumption, which tracks the real constraint for the majority of service workloads.

**Scaling can amplify a downstream problem.** Adding application instances increases connections to the database and request rate to dependencies. When the bottleneck lies there, a scaler observing rising latency responds by adding precisely the load causing it, which can drive a struggling dependency into complete failure. A maximum bound therefore serves two purposes — limiting cost, and preventing the application tier from scaling into a constraint it cannot relieve — and the underlying lesson is that autoscaling cannot fix a bottleneck outside the tier it controls.

**Reactive scaling is structurally late.** From a load increase to genuinely useful capacity typically takes several minutes: the metric window must reflect the change, the decision must be made, an instance scheduled and started, the application initialised, and then caches warmed and connections established before it carries its share. A short surge is over before any of this completes. Spikes must therefore be absorbed by headroom in the existing fleet and by load shedding beyond it, and attempts to tune a scaler aggressively enough to catch bursts produce oscillation while still failing to help.

**Predictable load is better anticipated than chased.** Most traffic has substantial daily and weekly structure, and scheduled scaling that raises the capacity floor before a known ramp delivers warm capacity exactly when it is needed — which no reactive configuration can achieve against a rising curve. Leaving the reactive scaler to handle only the variance around a scheduled baseline is consistently more effective than trying to make it track the whole pattern, and it also permits more conservative reactive tuning, which reduces oscillation further.

**Prove it — interview questions**

1. **[Basic] Why does autoscaling need hysteresis?**

   <details><summary>Model answer</summary>

   Because the scaler's own action changes the signal it reacts to. With a single threshold, adding an instance drops the metric below that threshold, which triggers a scale-down, which pushes it back above, and the system oscillates indefinitely — each cycle costing startup time, cold caches and connection churn. Separate thresholds for scaling up and down, with a dead band between them, absorb the effect of the scaler's own action so the system converges instead.

   </details>

2. **[Basic] Why should scale-up and scale-down behave differently?**

   <details><summary>Model answer</summary>

   Because the two errors cost different things. Insufficient capacity means degraded service or failed requests, which harms users; excess capacity means paying for instances you do not need, which costs money. Those are not symmetric, so the response should not be either: scale up quickly, in proportional steps, on a short cooldown, and scale down slowly, one instance at a time, on a much longer cooldown. Erring toward too much capacity is the cheaper mistake.

   </details>

3. **[Senior] Why is CPU often the wrong scaling signal?**

   <details><summary>Model answer</summary>

   Because most services do not saturate on CPU. A request-handling service is typically constrained by its connection pool, a downstream dependency, lock contention or memory, so it can be completely saturated at forty per cent CPU with latency climbing while the scaler sits well below its threshold and does nothing. Worse, if it eventually does scale, adding instances increases connections and request rate against the dependency that was actually the bottleneck, making the situation worse. Concurrency or queue depth reflect the real constraint far better because they measure work waiting rather than processor use.

   </details>

4. **[Senior] Why can't autoscaling handle traffic spikes?**

   <details><summary>Model answer</summary>

   Because the loop takes minutes end to end. The metric window has to reflect the change, the scaler decides, an instance is scheduled and started, the application initialises, and then caches warm and connections establish before it is genuinely useful — commonly three minutes or more. A two-minute surge is therefore over before the capacity arrives, so the scaling achieved nothing except adding cost afterwards. Spikes have to be absorbed by headroom in the existing fleet and by load shedding beyond that; autoscaling is for sustained changes over tens of minutes.

   </details>

5. **[Staff] Configure autoscaling for an API service with a daily traffic pattern.**

   <details><summary>Model answer</summary>

   First, the signal: I would scale on in-flight request concurrency rather than CPU, because a typical I/O-bound service saturates on its database connection pool well before CPU becomes interesting, and a CPU-based scaler on such a service simply never fires while latency climbs. Then hysteresis with a deliberately wide dead band — scale up above a high concurrency mark, scale down below a much lower one — so a single scaling step cannot push the metric back across the other threshold. Asymmetric behaviour in both step size and cooldown: up by a proportion of current capacity with a short cooldown, down by one instance with a much longer one, because the costs of the two mistakes differ. Bounds set deliberately, with a minimum that survives losing an availability zone rather than merely serving average load, and a maximum chosen from where the database connection pool saturates — beyond that, more application instances make things worse rather than better, so the bound protects the dependency as much as the budget. For the daily pattern I would use scheduled scaling to raise the floor before the morning ramp, because reactive scaling is structurally incapable of catching a predictable curve and will still be catching up well after the load arrives. And I would target around sixty per cent steady-state utilisation so there is headroom to absorb a surge during the minutes scaling takes, with load shedding beyond that.

   </details>

6. **[Principal] What is the most common misconception about autoscaling?**

   <details><summary>Model answer</summary>

   That it is a substitute for capacity planning and for load shedding, when it is neither. It cannot respond to spikes — the loop is minutes long and a surge is typically shorter — so a system relying on it for burst protection will fail during exactly the events it was expected to handle, and the usual response is to tune the scaler more aggressively, which produces oscillation without solving anything. It also cannot fix a bottleneck that is not in the tier being scaled, and in that case actively makes things worse by increasing pressure on the constrained dependency, which is a genuinely dangerous failure mode because latency rising causes the scaler to add exactly the load that is causing the latency. What autoscaling does well is track sustained variation over tens of minutes, which is real and valuable for cost. The framing I would push is that a system needs three separate things: headroom for seconds, shedding for the period beyond headroom, and scaling for sustained change — and that treating any one of them as covering the others is how capacity problems become outages. The related discipline is verifying what the service actually saturates on before configuring anything, because the default of CPU is wrong for most services and produces a scaler that looks configured and does nothing.

   </details>

---
