# k8s-homelab

A hands-on Kubernetes lab I built to get proper muscle memory with k8s — deployments, debugging, rollbacks, node inspection, the works. Runs entirely free using k3d (k8s in Docker) so no cloud spend needed.

---

## What I wanted to get comfortable with

Knowing kubectl commands is one thing. Actually breaking a deployment, watching ImagePullBackOff appear, digging into describe output and rolling back cleanly is another. This lab is about the second thing.

---

## Stack

- k3d v5.8.3
- k3s v1.31.5
- Terraform v1.15.3
- kubectl
- Docker

---

## Cluster setup

1 control plane, 2 worker nodes spun up with k3d:

```bash
k3d cluster create homelab --agents 2
kubectl get nodes
```

---

## What the lab covers

**Deploying an app**

nginx deployment with 3 replicas, resource limits, and both liveness and readiness probes configured. The distinction matters — readiness failing pulls a pod from the load balancer without restarting it, liveness failing triggers a restart. In a high-throughput environment like a trading platform that difference is significant.

**Breaking it deliberately**

Applying a deployment with a non-existent image tag to produce an ImagePullBackOff. The point is to practice the diagnostic flow from scratch rather than just reading about it.

```bash
kubectl get pods -n homelab -w
kubectl describe pod <pod-name> -n homelab
```

The Events section in describe output is where kubelet tells you exactly what went wrong.

**Rolling back**

```bash
kubectl rollout undo deployment/nginx -n homelab
kubectl rollout history deployment/nginx -n homelab
```

Rollback first, investigate after. Annotating deployments with a change cause makes the history readable during an incident rather than just a list of revision numbers.

**Node level debugging**

Getting inside a node to check what's actually running:

```bash
docker exec -it k3d-homelab-server-0 sh
ps aux | grep k3s
```

In a real cluster this would be SSH. Checking node conditions via kubectl describe node covers the things that explain why pods aren't scheduling — DiskPressure, MemoryPressure, NetworkUnavailable.

**Scaling**

```bash
kubectl scale deployment nginx --replicas=6 -n homelab
kubectl get pods -n homelab -o wide
```

Pods spread across nodes automatically. In production you'd also configure terminationGracePeriodSeconds to avoid dropping in-flight requests during scale down.

---

## Commands I use most

```bash
# what's running and where
kubectl get pods -n homelab -o wide

# why isn't this pod starting
kubectl describe pod <name> -n homelab

# what did it log before it crashed
kubectl logs <name> -n homelab --previous

# is the rollout actually progressing
kubectl rollout status deployment/nginx -n homelab

# roll back
kubectl rollout undo deployment/nginx -n homelab

# get inside a pod
kubectl exec -it <name> -n homelab -- sh

# node health
kubectl describe node <node-name>
```

---

## Teardown

```bash
k3d cluster delete homelab
```
