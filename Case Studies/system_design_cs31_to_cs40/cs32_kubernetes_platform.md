# Case Study 32 — Kubernetes Platform Engineering

> **How to use this file**
> Close it → run the 5-minute flow from memory → come back and compare.
> Pay most attention to: Resource management, HPA/VPA/KEDA, Pod disruption budgets.

---

## Business Context

Running Kubernetes is not hard. Running Kubernetes well for a payment platform — with predictable performance, cost efficiency, zero-downtime deployments, and proper resource isolation between teams — requires platform engineering. The difference between "we have K8s" and "we have a reliable K8s platform" is the layer of guardrails, automation, and observability added on top.

**Business goals driving architecture:**
- Teams deploy independently without affecting each other
- Payment pods never get evicted during peak traffic
- Resources right-sized (not over-provisioned by 5× "to be safe")
- Zero-downtime rolling deployments
- Cost visibility: which team/service is spending how much

---

## Recognition Framework

### The K8s Production Checklist

```
Most teams starting with K8s miss these:
  1. No resource requests/limits → scheduler cannot place pods efficiently; noisy neighbour
  2. No PodDisruptionBudgets → node drain evicts all pods simultaneously → outage
  3. No readiness probes → traffic sent before service is ready → errors
  4. No Horizontal Pod Autoscaling → fixed pod count; OOM during peak
  5. Requests >> Limits → request 0.1 CPU but actually use 1 CPU → other pods starved
  6. No namespace resource quotas → one team deploys 100× and starves others
  7. Single AZ node pool → AZ failure = all pods gone
  8. No pod anti-affinity → all replicas on same node → node failure = service down
  9. No preStop hook → pod killed while still processing requests
  10. Latest tag in image → unpredictable deployments
```

---

## Problem Statement

Production Kubernetes platform for a payment microservices platform. Proper resource management, autoscaling, zero-downtime deployments, multi-team isolation, and cost visibility.

---

## Core Configuration Patterns

### Resource Requests, Limits, and QoS Classes
```yaml
# Payment-service deployment: critical tier; Guaranteed QoS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: payments
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0        # never reduce capacity during rollout
      maxSurge: 1              # add 1 pod during rollout
  template:
    spec:
      containers:
      - name: payment-service
        image: payment-service:1.2.3    # NEVER use :latest
        resources:
          requests:
            cpu: "500m"        # 0.5 CPU cores reserved for scheduling
            memory: "512Mi"
          limits:
            cpu: "1000m"       # max 1 CPU before throttling
            memory: "512Mi"    # same as request → Guaranteed QoS (never evicted under memory pressure)
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          failureThreshold: 3  # remove from LB after 3 failures
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          periodSeconds: 10
          failureThreshold: 3  # restart pod after 3 consecutive failures
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sleep", "15"]  # wait 15s before SIGTERM
              # Gives time for LB to stop sending traffic before pod stops
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: payment-service
            topologyKey: kubernetes.io/hostname  # never 2 replicas on same node
```

### Namespace Resource Quotas (Team Isolation)
```yaml
# payments namespace: high priority; generous quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: payments
spec:
  hard:
    requests.cpu: "20"        # team can request up to 20 CPU cores
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    count/pods: "100"
    count/services: "20"
    persistentvolumeclaims: "10"
---
# analytics namespace: lower priority; smaller quota
apiVersion: v1
kind: LimitRange
metadata:
  name: analytics-defaults
  namespace: analytics
spec:
  limits:
  - default:
      cpu: "200m"        # default limit if not specified (prevents unlimited CPU grabs)
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

### Horizontal Pod Autoscaler (HPA) + KEDA
```yaml
# HPA: scale on CPU (built-in Kubernetes)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
  namespace: payments
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3            # never below 3 (multi-AZ coverage)
  maxReplicas: 50           # cap to prevent runaway scaling
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # scale up when avg CPU > 70%
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # scale up quickly (30s window)
      policies:
      - type: Pods
        value: 5
        periodSeconds: 60              # add up to 5 pods per minute
    scaleDown:
      stabilizationWindowSeconds: 300  # scale down slowly (5 min stability before scaling down)
---
# KEDA: scale on Kafka consumer lag (event-driven autoscaling)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: payment-event-consumer-scaler
spec:
  scaleTargetRef:
    name: payment-event-consumer
  minReplicaCount: 2
  maxReplicaCount: 30
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: payment-processors
      topic: payment.events
      lagThreshold: "1000"        # scale out when lag > 1000 messages per partition
```

### Pod Disruption Budget (Zero-Downtime Maintenance)
```yaml
# Ensures at least 2 payment-service pods are always available
# Even during: node drain, K8s upgrade, AZ migration
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-service-pdb
  namespace: payments
spec:
  selector:
    matchLabels:
      app: payment-service
  minAvailable: 2     # always keep at least 2 pods running
  # OR: maxUnavailable: 1 (at most 1 pod can be disrupted at a time)

# Without PDB: kubectl drain node → all 3 payment-service pods evicted simultaneously → outage
# With PDB: kubectl drain node → drains 1 pod at a time → always 2 pods serving
```

### Node Pools for Workload Classes
```yaml
# Different node types for different workload requirements:

# Payment-critical: high-memory, reserved instances (predictable performance)
payment-nodepool:
  machineType: c5.2xlarge
  spotInstances: false       # never use spot for payment processing (can be terminated)
  minNodes: 6
  maxNodes: 20
  taints:
  - key: "workload-class"
    value: "payment-critical"
    effect: NoSchedule        # only pods with matching toleration can run here

# Analytics: can use spot instances (cost-optimised; can tolerate termination)
analytics-nodepool:
  machineType: m5.4xlarge
  spotInstances: true         # 70% cost saving; analytics can retry
  minNodes: 2
  maxNodes: 50
  taints:
  - key: "workload-class"
    value: "analytics"
    effect: NoSchedule

# Payment pods use toleration to land on payment-critical nodes:
tolerations:
- key: "workload-class"
  value: "payment-critical"
  effect: NoSchedule
# Analytics pods get cheaper nodes; payment pods get dedicated high-performance nodes
```

---

## Detailed Components

### Zero-Downtime Rolling Deployment
```
Current: payment-service v1.2.3 (3 pods)
Deploying: payment-service v1.2.4

Sequence (maxUnavailable=0, maxSurge=1):
  Step 1: Create 1 new v1.2.4 pod (total: 3 v1.2.3 + 1 v1.2.4 = 4 pods)
  Step 2: Wait for v1.2.4 pod to pass readiness probe
  Step 3: Add v1.2.4 pod to Service endpoints → traffic flows to it
  Step 4: Terminate 1 v1.2.3 pod (preStop hook: sleep 15s → allow in-flight requests to drain)
  Step 5: Repeat until all v1.2.3 pods replaced
  
  At no point are there < 3 pods serving → zero capacity reduction
  If v1.2.4 pod fails readiness: rollout pauses; v1.2.3 pods keep serving

Automatic rollback:
  kubectl rollout undo deployment/payment-service
  Or: ArgoCD/Flux detects new pod crash rate > threshold → auto-rollback
```

### Vertical Pod Autoscaler (VPA) for Right-Sizing
```yaml
# VPA in "Off" mode: recommends right-size without changing pods
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: payment-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  updatePolicy:
    updateMode: "Off"         # Only recommend; don't auto-apply
  resourcePolicy:
    containerPolicies:
    - containerName: payment-service
      minAllowed:
        cpu: 200m
        memory: 256Mi
      maxAllowed:
        cpu: 4000m
        memory: 8Gi

# VPA output (recommendation):
#   container: payment-service
#   recommendation:
#     lowerBound: cpu=390m, memory=420Mi
#     target: cpu=500m, memory=512Mi
#     upperBound: cpu=1200m, memory=1Gi

# Use: review recommendations weekly; update requests manually based on VPA advice
# Don't use VPA in Auto mode for payment-critical services (pod restarts on resize)
```

---

## Observability
```
Key K8s operational metrics:
  kube_pod_status_phase                    (running/pending/failed; alert on stuck pending)
  kube_deployment_status_replicas_unavailable (alert if > 0 for payment services)
  kube_node_status_condition               (node health; alert on NotReady)
  kube_pod_container_resource_requests     (actual requested vs quota)
  container_cpu_usage_seconds_total        (vs limit → throttling detection)
  container_memory_usage_bytes             (vs limit → OOM risk)
  hpa_status_current_replicas             (autoscaling health)
  keda_scaledobject_scale_source_errors    (KEDA trigger health)

Cost metrics:
  cost_per_namespace_per_day               (team-level cost visibility)
  spot_instance_savings_vs_on_demand       (cost efficiency)
  cpu_request_utilization_ratio            (right-sizing: if 20% then over-provisioned)
  OOM kill rate                            (if high: memory limit too low)
```

---

## Chaos Testing
```
Experiment 1: Node Drain (K8s Upgrade Simulation)
  Inject:    kubectl drain node-1 (force node to be evacuated)
  Expected:  PDB enforced: pods evicted one at a time (not all at once)
  Expected:  payment-service: always ≥ 2 pods available
  Expected:  No payment failures during drain
  Red flag:  Drain evicts all pods simultaneously → payment outage

Experiment 2: HPA Under Traffic Spike
  Inject:    Load test: 10× normal traffic for 5 minutes
  Expected:  HPA detects CPU > 70%; triggers scale-out
  Expected:  5 pods added per minute (scaleUp policy); max 50 pods
  Expected:  Payment latency stays within SLO during scale-out (warmup lag)
  Red flag:  HPA doesn't trigger until service is already OOM (metrics lag)

Experiment 3: Spot Instance Termination (Analytics)
  Inject:    Terminate spot instance running analytics pods
  Expected:  Analytics pod rescheduled to new spot instance
  Expected:  Analytics job retries from checkpoint
  Expected:  Payment-critical pods: unaffected (on non-spot nodes)
  Red flag:  Payment pods scheduled on spot instances (toleration bug)
```

---

## Architecture Review Checklist
```
□ QoS classes and eviction order:
  BestEffort (no requests/limits) → evicted first (never for payment services)
  Burstable (limits > requests) → evicted second (analytics, batch)
  Guaranteed (limits == requests) → evicted last (payment, fraud, ledger services)
  Rule: payment-critical services MUST be Guaranteed QoS

□ Pod anti-affinity:
  Payment service: requiredDuringScheduling + topologyKey: hostname
  Ensures pods spread across nodes → single node failure loses max 1 pod

□ Multi-AZ spread:
  Node pool across 3 AZs + topologySpreadConstraints
  maxSkew: 1 → pods evenly spread across AZs

□ SCC LENS:
  STATE: K8s etcd (desired state), Kubelet (actual pod state per node)
  COORDINATION: Control plane reconciliation (continuously converges actual to desired state)
  CONCENTRATION: API server is the single write point → HA etcd mandatory; API server rate limits
```

---

*Next: Case Study 33 — CI/CD Pipeline and GitOps*
