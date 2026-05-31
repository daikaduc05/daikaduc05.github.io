---
title: "Kubernetes Quest: From Crashing Pods to Broken Deployments — What I Learned Debugging Real Scenarios"
date: 2026-05-31 00:00:00 +0700
categories: [Kubernetes, DevOps]
tags: [kubernetes, kubectl, debugging, k8squest, pods, deployments]
description: My journey through the first two challenges of K8sQuest — a hands-on Kubernetes learning game where each level throws a broken cluster at you and asks you to fix it.
---

> This post covers my journey through the first two challenges of K8sQuest, a hands-on Kubernetes learning game. Each level throws a broken cluster at you and asks you to fix it — no hand-holding, just `kubectl` and your wits.

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

## Putting It Together: The Mental Stack

After these two levels, here's the mental model I've built:

```
Deployment (desired state: 3 replicas of nginx:latest)
    └── ReplicaSet (ensures 3 pods exist)
            ├── Pod 1 (nginx container running)
            ├── Pod 2 (nginx container running)
            └── Pod 3 (nginx container running)
```

**Pods** are the atoms — fundamental, ephemeral, immutable once created.  
**Deployments** are the molecules — they compose pods into something resilient, scalable, and manageable.

In production: you almost never touch pods directly. You talk to deployments, and deployments handle the rest.

---

## Commands Reference Cheat Sheet

```bash
# --- Pods ---
kubectl get pod <name> -n <ns>
kubectl describe pod <name> -n <ns>        # Events: always read this first
kubectl logs <name> -n <ns>
kubectl logs <name> -n <ns> --previous     # logs from last crash
kubectl delete pod <name> -n <ns>
kubectl apply -f pod.yaml

# --- Deployments ---
kubectl get deployment <name> -n <ns>
kubectl describe deployment <name> -n <ns>
kubectl scale deployment <name> --replicas=N -n <ns>
kubectl edit deployment <name> -n <ns>
kubectl rollout status deployment/<name> -n <ns>
kubectl get rs -n <ns>                     # see ReplicaSets
kubectl get pods -l app=<label> -n <ns>   # see managed pods
```

---

## What's Next

Level 3 brings more complex deployment scenarios — health checks, readiness probes, and rolling updates. The foundation we've built here (understanding the pod lifecycle and the Deployment → ReplicaSet → Pod hierarchy) is exactly what makes those concepts click.

Stay tuned.
