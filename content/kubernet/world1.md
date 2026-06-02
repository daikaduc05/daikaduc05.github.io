# Kubernetes Quest: Five Levels of Broken Clusters — What I Learned Debugging Real Scenarios

> This post covers my journey through the first five challenges of K8sQuest, a hands-on Kubernetes learning game. Each level throws a broken cluster at you and asks you to fix it — no hand-holding, just `kubectl` and your wits.

---

## Level 1: Fix the Crashing Pod

### The Problem

My pod was stuck in **CrashLoopBackOff**. Not a fun place to be, especially if it's 3 AM and your phone is ringing.

The culprit? A `command` field in the pod spec pointing to `nginxzz` — a command that doesn't exist inside the nginx container image.

```
Error: failed to create containerd task: exec: "nginxzz": executable file not found in $PATH
```

Kubernetes didn't fail to schedule the pod, didn't fail to pull the image — it just faithfully tried to run a command that doesn't exist. Every. Single. Time.

### What Kubernetes Was Doing Under the Hood

1. **Scheduler** assigned the pod to a node ✅
2. **Kubelet** pulled the nginx image ✅
3. **Container runtime** tried to start with command `["nginxzz"]` ❌
4. **Container crashed** immediately
5. **Kubelet** retried with exponential backoff → **CrashLoopBackOff**

The key insight here: Kubernetes is not smart. It does exactly what you tell it. If you tell it to run a nonexistent command, it will keep trying to run that nonexistent command until the heat death of the universe (or until you fix it).

### How to Debug It

```bash
# Step 1: What state is the pod in?
kubectl get pod <name> -n <namespace>

# Step 2: WHY is it in that state?
kubectl describe pod <name> -n <namespace>   # Events section is your friend

# Step 3: What did the app say before it died?
kubectl logs <name> -n <namespace>
kubectl logs <name> -n <namespace> --previous   # last crash's logs
```

The `Events` section from `kubectl describe` is where the story lives. If you read nothing else, read that.

### The Fix

Pods are **immutable** after creation — you can't edit most fields on a live pod. So the workflow is:

```bash
kubectl delete pod <name> -n <namespace>
kubectl apply -f fixed-pod.yaml
```

Remove the bad `command` field (or correct it), reapply, done.

### Key Mental Model: Pods

| Concept | What It Means |
|---|---|
| Pod immutability | Most fields can't change after creation — delete and recreate |
| `command` field | Overrides the image's default entrypoint — use carefully |
| CrashLoopBackOff | Container keeps failing — check Events + logs to find out why |
| Container image | Defines *what can run* — your command must exist inside it |

### Real-World War Story: The $50K Startup Script

A developer added this to a deployment:

```yaml
command: ["/app/startup.sh"]
```

The script existed — but wasn't marked executable (`chmod +x` was never run). Every pod crashed instantly. The API was down for 12 minutes. $50K in revenue, gone.

**Lesson:** Always test container commands locally before shipping them:

```bash
docker run --rm -it <image> <your-command>
```

If it doesn't work in Docker, it won't work in Kubernetes.

### Interview Questions This Level Prepares You For

**Q: A pod is in CrashLoopBackOff. How do you debug it?**

1. `kubectl describe pod <name>` — read the Events
2. `kubectl logs <name> --previous` — read the last crash output
3. Look for: command not found, missing files, bad config, crashing app
4. Fix → delete pod → reapply

**Q: Can you edit a running pod?**

Only a handful of fields: `spec.containers[*].image`, `spec.activeDeadlineSeconds`, `spec.tolerations` (additions only). Everything else requires delete + recreate — which is exactly why we use **Deployments** in production.

---

## Level 2: Fix the Deployment

### The Problem

This time the cluster had a Deployment — but nothing was running. No pods. The deployment reported `0/0 ready` and considered itself perfectly healthy.

The cause: `replicas: 0`.

This is valid Kubernetes configuration. It's not a bug. Kubernetes did exactly what it was asked to do: run zero instances of the application. Kubernetes was thriving. The service was not.

### What Kubernetes Was Doing Under the Hood

1. **Deployment Controller** read the spec: `replicas: 0`
2. **ReplicaSet** was created with desired count = 0
3. **No pods were scheduled** — intentional, by design
4. **Deployment status:** `0/0 ready` ✅ — from Kubernetes' perspective, everything was fine

The hierarchy to keep in mind:

```
Deployment → ReplicaSet → Pods → Containers
```

The Deployment owns the desired state. The ReplicaSet enforces it. The pods are the actual running workloads.

### How to Fix It

Three ways, depending on how urgent things are:

```bash
# 1. Imperative — fastest, for emergencies
kubectl scale deployment web --replicas=3 -n k8squest

# 2. Interactive — for one-off changes
kubectl edit deployment web -n k8squest
# (find replicas: 0, change to replicas: 3, save)

# 3. Declarative — the right way for production
# Edit the YAML, set replicas: 3, then:
kubectl apply -f deployment.yaml
```

Unlike pods, **Deployments are mutable**. You can edit them live and changes roll out gracefully.

### Key Mental Model: Deployments

| Concept | What It Means |
|---|---|
| `replicas: 0` | Valid config — means "run nothing" (maintenance mode or mistake) |
| `replicas: 1` | One pod, no redundancy — if it dies, downtime |
| `replicas: 3` | Three pods — survives one or two node failures |
| `readyReplicas` | Pods that passed health checks — the number that actually matters |
| Rolling update | Scale new ReplicaSet up, old one down — zero-downtime deploys |

### Useful Commands for Deployments

```bash
# Status overview
kubectl get deployment <name> -n <namespace>
kubectl get deployment <name> -n <namespace> -o wide

# Detailed events
kubectl describe deployment <name> -n <namespace>

# Watch a rollout
kubectl rollout status deployment/<name> -n <namespace>

# Scale
kubectl scale deployment <name> --replicas=N -n <namespace>

# See ReplicaSets managed by the deployment
kubectl get rs -n <namespace>

# See pods owned by the deployment
kubectl get pods -l app=<label> -n <namespace>
```

### Real-World War Story: The Black Friday Disaster

An e-commerce company's DevOps engineer was testing autoscaling behavior. He left this in the code:

```yaml
spec:
  replicas: 0  # TODO: test HPA from 0
```

It got merged. CI/CD deployed it to production at 11:45 PM on Thanksgiving night. When Black Friday traffic hit at midnight — the checkout service had zero pods.

- 15 minutes of downtime
- Estimated $2.3M in lost sales
- Made tech headlines

**The fix (1 minute):**

```bash
kubectl scale deployment checkout --replicas=10 -n production
```

**The permanent fixes:**
1. Git pre-commit hook: block `replicas: 0` in production configs
2. Admission webhook: reject the config before it even applies
3. Alert: fire when any production deployment drops below 1 replica

**Lesson:** Always have a minimum replica count for critical services. Monitor desired state, not just actual state.

### Interview Questions This Level Prepares You For

**Q: What's the difference between a Pod and a Deployment?**

- **Pod** = single container instance, ephemeral, can't self-heal
- **Deployment** = manages N pods, maintains desired count, handles rolling updates, restarts crashed pods

In production, you almost never create raw pods. Deployments manage pod lifecycle for you.

**Q: How do you scale an application in Kubernetes?**

```bash
# Manual horizontal scaling
kubectl scale deployment <name> --replicas=5

# Automatic scaling based on CPU/memory
kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=70
```

**Q: A deployment shows 0/3 ready. What could be wrong?**

1. Image pull error — wrong image name or tag
2. Pods crashing — bad config, missing env vars, app bug
3. Health checks failing — readiness probe misconfigured
4. Resource constraints — not enough CPU/memory on the cluster

Debug path:
```bash
kubectl get pods                           # see pod states
kubectl describe deployment <name>         # see deployment events
kubectl logs <pod-name>                    # see what the app says
```

---

## Level 3: ImagePullBackOff Mystery

### The Problem

The pod was there. The node was fine. The image name looked reasonable. But nothing started — status stuck at **ImagePullBackOff**.

The culprit: `nginx:nonexistent-tag-xyz-123`. That tag doesn't exist on Docker Hub. Kubernetes can't pull what isn't there, so the kubelet keeps trying, backs off, tries again. Forever.

```
Failed to pull image "nginx:nonexistent-tag-xyz-123": rpc error: code = NotFound
```

### What Kubernetes Was Doing Under the Hood

```
kubectl apply → Scheduler assigns node → Kubelet pulls image ❌ → ImagePullBackOff
```

The pod lifecycle has distinct phases. Kubernetes got past scheduling just fine — it was the image pull that blocked everything:

1. **Pending** → scheduled to a node ✅
2. **ContainerCreating** → kubelet attempting to pull the image ❌
3. **ImagePullBackOff** → pull failed, retrying with exponential backoff

Unlike CrashLoopBackOff (where the image pulls fine but the app crashes), here Kubernetes can't even get to starting the container.

### How to Debug It

```bash
# Check pod status
kubectl get pod <name> -n <namespace>

# This is where the answer lives:
kubectl describe pod <name> -n <namespace>
# Look at Events — you'll see "Failed to pull image" with the exact reason

# Confirm what image it's actually trying
kubectl get pod <name> -n <namespace> -o yaml | grep image:
```

The Events section in `kubectl describe` tells you exactly what went wrong: wrong tag, wrong repo, authentication failure, or network issue.

### The Fix

```bash
# Correct the image tag in your YAML:
# image: nginx:nonexistent-tag-xyz-123  ← broken
# image: nginx:latest                   ← fixed

kubectl delete pod <name> -n <namespace>
kubectl apply -f fixed-pod.yaml
```

Three categories of image pull failures, each with a different fix:

| Failure | Cause | Fix |
|---|---|---|
| `not found` | Wrong tag or image name | Correct the reference |
| `unauthorized` | Private registry, no credentials | Add `imagePullSecrets` |
| `connection refused` | Registry unreachable | Check network / registry URL |

### Real-World War Story: The $800K Launch Day

A team pushed a deployment referencing `v2.3.1` — but CI/CD had built and pushed `v2.3.0`. All pods went into ImagePullBackOff during a product launch. No monitoring alert fired (nobody had set one up for this state), so they spent two hours debugging application code before someone checked `kubectl describe pod`.

**Fix took 30 seconds once they found it.**

**Lesson:** Always verify an image exists before deploying — `docker pull <image>` locally, or check your registry UI. And set up alerts for ImagePullBackOff.

### Interview Questions This Level Prepares You For

**Q: What's the difference between ImagePullBackOff and CrashLoopBackOff?**

- **ImagePullBackOff** — the container never started; Kubernetes can't even get the image
- **CrashLoopBackOff** — the image pulled fine; the container starts and immediately exits

Same debug tool (`kubectl describe`), different section to read: image events vs. exit code / logs.

**Q: How do you pull from a private registry?**

Create a Secret with registry credentials, then reference it:

```yaml
spec:
  imagePullSecrets:
  - name: my-registry-secret
```

---

## Level 4: Pending Pod Problem

### The Problem

The pod was created. No errors. But it sat in **Pending** — forever. No node was picked. Nothing was running.

The cause: the resource requests were unreasonably large. When Kubernetes couldn't find a node with enough headroom, it simply didn't schedule the pod. No retries with backoff here — just permanent Pending until you fix the spec or add capacity.

```yaml
resources:
  requests:
    memory: "1"   # missing unit — ambiguous/wrong
    cpu: "1"      # 1 full core — may not fit
```

The solution: use realistic requests with proper units.

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "100m"
  limits:
    memory: "128Mi"
    cpu: "200m"
```

### What Kubernetes Was Doing Under the Hood

The **scheduler** runs a two-phase process for every unscheduled pod:

1. **Filtering** — eliminate nodes that don't meet requirements (resources, taints, node selectors)
2. **Scoring** — rank the remaining nodes by best fit
3. **Binding** — assign the pod to the winning node

When resource requests are too high, no node survives the filter. The pod sits in Pending indefinitely, and `kubectl describe pod` tells you exactly why:

```
Events:
  Warning  FailedScheduling  0/3 nodes are available: 3 Insufficient cpu
```

### Resource Requests vs. Limits

This is a fundamental Kubernetes concept worth getting right:

| Field | What It Means | Used For |
|---|---|---|
| `requests.cpu` | Guaranteed minimum CPU | Scheduling decisions |
| `requests.memory` | Guaranteed minimum memory | Scheduling decisions |
| `limits.cpu` | CPU cap (throttled if exceeded) | Runtime enforcement |
| `limits.memory` | Memory cap (OOMKilled if exceeded) | Runtime enforcement |

**CPU units:**
- `1` = 1 full CPU core
- `100m` = 100 millicores = 0.1 CPU
- `500m` = half a core

**Memory units:**
- `Mi` = mebibytes (what you almost always want)
- `Gi` = gibibytes
- `M` / `G` = megabytes / gigabytes (different, and easy to confuse)

Start small (`100m` CPU, `64Mi` memory), observe actual usage with `kubectl top pod`, then adjust. Don't guess.

### How to Debug It

```bash
# Pod is Pending — why?
kubectl describe pod <name> -n <namespace>
# Read Events → "Insufficient cpu" / "Insufficient memory"

# Check what nodes actually have available
kubectl describe nodes
# Look for "Allocatable" vs "Requests" sections

# Check actual live usage (requires metrics-server)
kubectl top pod -n <namespace>
kubectl top nodes

# See all events sorted by time
kubectl get events --sort-by='.lastTimestamp' -n <namespace>
```

### Real-World War Story: The Missed Deadline

A developer copy-pasted a "production-grade" blog post config that set `cpu: 2` and `memory: 4Gi` for every microservice. Their dev cluster had 2-CPU nodes. They deployed 10 services — only the first couple scheduled. The rest stayed Pending.

For six hours, ops thought it was a node health issue and debugged infrastructure. Nobody checked `kubectl describe pod` events until hour five. Changing requests to `100m` / `128Mi` fixed everything instantly — but the enterprise demo was already missed.

**Lesson:** Pending = scheduling failure. Always read Events first.

### Key Mental Model: Scheduling

| Pod State | What It Means | First Command to Run |
|---|---|---|
| `Pending` | No node found yet | `kubectl describe pod` → Events |
| `ContainerCreating` | Node found, pulling image | `kubectl describe pod` → Events |
| `Running` | Container started | `kubectl logs` to check app health |
| `CrashLoopBackOff` | Container keeps failing | `kubectl logs --previous` |
| `ImagePullBackOff` | Image can't be pulled | `kubectl describe pod` → image Events |

---

## Level 5: Lost Connection — Labels & Selectors

### The Problem

The pod was running. The Service existed. But no traffic was reaching the app. The Service had **zero endpoints** — it couldn't find its pods.

The broken YAML:

```yaml
# Pod labels:
labels:
  app: backend
  tier: api

# Service selector:
selector:
  app: frontend   # ← wrong! should be 'backend'
  tier: api
```

One word off. Total disconnect.

### What Kubernetes Was Doing Under the Hood

Services don't reference pods by name. They use **label selectors** to dynamically discover which pods should receive traffic:

1. Service is created with a selector (`app: frontend, tier: api`)
2. Service controller watches all pods
3. Finds pods where ALL selector labels match
4. Creates an **Endpoints** object with matching pod IPs
5. Routes traffic to those IPs

When no pods match the selector, the Endpoints object is empty — and traffic has nowhere to go.

```
Service selector: {app: frontend, tier: api}
                        ↓
Pod: {app: backend, tier: api}  ← ❌ app doesn't match
                        ↓
Endpoints: <none>
                        ↓
Traffic: connection refused
```

### How to Debug It

The key command is `kubectl get endpoints`. An empty endpoints list is the smoking gun:

```bash
# First: is the Service even finding any pods?
kubectl get endpoints <service-name> -n <namespace>
# If ENDPOINTS shows <none> or empty — label mismatch

# What does the Service think it's looking for?
kubectl describe service <service-name> -n <namespace>
# Look at "Selector:" field

# What labels do the pods actually have?
kubectl get pods --show-labels -n <namespace>

# Do these match? Try querying directly:
kubectl get pods -l app=backend -n <namespace>
kubectl get pods -l app=frontend -n <namespace>
# One of these will return nothing — that's your mismatch
```

### The Fix

Two valid approaches — fix whichever side owns the "truth":

```bash
# Option A: Fix the Service selector to match pod labels
kubectl edit service <name> -n <namespace>
# Change: app: frontend → app: backend

# Option B: Relabel the pod (use --overwrite for existing labels)
kubectl label pod <name> app=frontend --overwrite -n <namespace>
```

In practice, fix the Service selector — pod labels are usually set by Deployments and reflect what the app actually is.

### Key Mental Model: How Labels Connect Everything

Labels are Kubernetes' universal wiring system. They're not just for Services:

| Object | Uses Labels For |
|---|---|
| Service | Selecting pods to route traffic to |
| Deployment | Tracking which ReplicaSet belongs to it |
| ReplicaSet | Selecting pods it manages |
| NetworkPolicy | Filtering which pods a policy applies to |
| HPA | Targeting which deployment to scale |

This is why label hygiene matters. A stray rename breaks everything downstream.

### Real-World War Story: The $125K Fintech Outage

A DevOps engineer updated a payment service deployment and accidentally changed the pod label from `app: payment-processor` to `app: payment-service`. The Service selector still expected `payment-processor`. Result: instant disconnect. Zero endpoints. All payment traffic dropped.

The Service stayed "up" in every health check — it existed, it responded — but it had no backends. Three hours of debugging "network issues" before someone ran `kubectl get endpoints payment-service` and saw the empty list. Then `kubectl get pods --selector=app=payment-processor` returned nothing. Found in 30 seconds once they looked at endpoints.

**Lesson:** When traffic mysteriously stops, check endpoints before anything else. Monitor endpoint count: alert if any production service drops to 0.

### Interview Questions This Level Prepares You For

**Q: A Service exists but pods aren't receiving traffic. How do you debug?**

1. `kubectl get endpoints <service>` — if empty, it's a label mismatch
2. `kubectl describe service <service>` — check the `Selector:` field
3. `kubectl get pods --show-labels` — compare against the selector
4. Fix whichever side is wrong, confirm endpoints populate

**Q: What happens if a pod has extra labels that aren't in the Service selector?**

Nothing — that's fine. The selector is a minimum-match query. A pod with `{app: backend, tier: api, env: prod}` matches a selector for `{app: backend, tier: api}` just fine.

---

## Putting It Together: The Mental Stack

After five levels, here's the mental model I've built:

```
Deployment (desired state: 3 replicas of nginx:latest)
    └── ReplicaSet (ensures 3 pods exist, identified by labels)
            ├── Pod 1 (nginx container running)   ─┐
            ├── Pod 2 (nginx container running)    ├── selected by Service via labels
            └── Pod 3 (nginx container running)   ─┘
                    ↑
              needs: correct image tag + enough node resources
```

**Pods** are the atoms — fundamental, ephemeral, immutable once created.
**Deployments** are the molecules — they compose pods into something resilient, scalable, and manageable.
**Labels** are the wiring — everything connects to everything else through them.

The five failure modes you now own:

| Level | Failure | Symptom | First Debug Command |
|---|---|---|---|
| 1 | Bad command | CrashLoopBackOff | `kubectl logs --previous` |
| 2 | `replicas: 0` | 0/0 ready, no pods | `kubectl get deployment` |
| 3 | Bad image tag | ImagePullBackOff | `kubectl describe pod` → Events |
| 4 | Excessive resource requests | Pending forever | `kubectl describe pod` → Events |
| 5 | Label mismatch | Pod running, no traffic | `kubectl get endpoints` |

In production: you almost never touch pods directly. You talk to deployments, labels connect the pieces, and `kubectl describe` + `kubectl get endpoints` tell you what's broken.

---

## Commands Reference Cheat Sheet

```bash
# --- Pods ---
kubectl get pod <name> -n <ns>
kubectl describe pod <name> -n <ns>              # Events: always read this first
kubectl logs <name> -n <ns>
kubectl logs <name> -n <ns> --previous           # logs from last crash
kubectl delete pod <name> -n <ns>
kubectl apply -f pod.yaml

# --- Image debugging ---
kubectl get pod <name> -n <ns> -o yaml | grep image:   # what image is it using?

# --- Resource debugging ---
kubectl describe nodes                           # node capacity and allocatable
kubectl top pod -n <ns>                          # live CPU/memory usage
kubectl top nodes
kubectl get events --sort-by='.lastTimestamp' -n <ns>

# --- Deployments ---
kubectl get deployment <name> -n <ns>
kubectl describe deployment <name> -n <ns>
kubectl scale deployment <name> --replicas=N -n <ns>
kubectl edit deployment <name> -n <ns>
kubectl rollout status deployment/<name> -n <ns>
kubectl get rs -n <ns>                           # see ReplicaSets

# --- Labels & Services ---
kubectl get pods --show-labels -n <ns>
kubectl get pods -l app=<value> -n <ns>          # filter by label
kubectl label pod <name> app=<value> --overwrite -n <ns>
kubectl get service <name> -n <ns>
kubectl describe service <name> -n <ns>          # check Selector field
kubectl get endpoints <name> -n <ns>             # empty = no matching pods
```

---

## What's Next

Level 6 introduces container port mismatches — a Service targeting the right pods but the wrong port number. The label and endpoint debugging skills from Level 5 carry directly into that one.

Stay tuned.
