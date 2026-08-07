# Kubernetes Real-World Errors — Pod & Container Level

A practical troubleshooting guide for the most common Pod and container errors you'll hit in production, why they happen, and how to fix them.

---

## 1. `CrashLoopBackOff`

**What it means:** The container starts, crashes, and Kubernetes keeps restarting it with an increasing back-off delay (10s, 20s, 40s… up to 5 min).

**Common causes:**
- Application bug / unhandled exception on startup
- Missing config, env var, or secret the app needs
- Failing liveness probe killing the container repeatedly
- Wrong command/entrypoint in the image
- Dependency not ready (DB, cache) and app exits instead of retrying

**How to diagnose:**
```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous     # logs from the crashed instance
kubectl get events --sort-by='.lastTimestamp'
```

**How to resolve:**
- Read `--previous` logs — the stack trace usually tells you exactly what's wrong.
- If it's a missing env/secret, patch the Deployment or fix the ConfigMap/Secret.
- If a liveness probe is too aggressive, increase `initialDelaySeconds` / `failureThreshold`.
- Add an init container or retry logic so the app waits for dependencies instead of exiting.
- Test the image locally: `docker run <image>` to confirm the entrypoint works.

---

## 2. `ImagePullBackOff` / `ErrImagePull`

**What it means:** The kubelet cannot pull the container image.

**Common causes:**
- Wrong image name or tag (typo, tag doesn't exist)
- Private registry without valid `imagePullSecrets`
- Registry rate limiting (e.g., Docker Hub anonymous pull limits)
- Network/firewall blocking the registry

**How to diagnose:**
```bash
kubectl describe pod <pod-name>   # look at Events for the exact pull error
```

**How to resolve:**
- Verify the image exists: `docker pull <image>:<tag>`.
- For private registries, create and attach a pull secret:
  ```bash
  kubectl create secret docker-registry regcred \
    --docker-server=<registry> \
    --docker-username=<user> \
    --docker-password=<pass>
  ```
  Then reference it under `spec.imagePullSecrets`.
- Pin explicit tags (avoid `latest`), and consider a pull-through cache or authenticated pulls to dodge rate limits.

---

## 3. `Pending` (Pod won't schedule)

**What it means:** The scheduler can't place the Pod on any node.

**Common causes:**
- Insufficient CPU/memory on all nodes
- Node selectors / affinity / taints not satisfied
- PVC waiting to bind (no matching StorageClass or PV)
- No nodes available (autoscaler scaling up)

**How to diagnose:**
```bash
kubectl describe pod <pod-name>       # Events show "0/3 nodes available: ..."
kubectl get nodes -o wide
kubectl top nodes
```

**How to resolve:**
- If resource-bound: lower requests, add nodes, or enable Cluster Autoscaler.
- If taint-bound: add the matching `toleration` or remove the taint.
- If affinity/selector-bound: fix the label expressions or label the nodes.
- If PVC-bound: check `kubectl get pvc` and ensure a StorageClass/PV can satisfy it.

---

## 4. `OOMKilled` (Exit Code 137)

**What it means:** The container exceeded its memory limit and the kernel OOM-killer terminated it.

**How to diagnose:**
```bash
kubectl describe pod <pod-name>   # State: Terminated, Reason: OOMKilled
kubectl top pod <pod-name>
```

**How to resolve:**
- Raise `resources.limits.memory` if the workload legitimately needs more.
- Fix memory leaks in the app (profile heap usage).
- Set requests/limits realistically — profile actual usage under load first.
- For JVM/Node apps, set the runtime heap to respect the container limit (e.g., `-XX:MaxRAMPercentage`).

---

## 5. `CreateContainerConfigError`

**What it means:** The container can't be created because a referenced ConfigMap or Secret is missing or a key doesn't exist.

**How to diagnose:**
```bash
kubectl describe pod <pod-name>   # Events name the missing ConfigMap/Secret
```

**How to resolve:**
- Create the missing ConfigMap/Secret, or fix the `key` referenced in `env`/`volumeMounts`.
- Confirm names and namespaces match — references are namespace-scoped.

---

## 6. `RunContainerError` / `CreateContainerError`

**What it means:** Container fails at creation/run time — often a bad command, missing mount path, or permission issue.

**How to resolve:**
- Check the `command`/`args` are valid for the image.
- Verify volume mount paths exist and are writable.
- Check `securityContext` (e.g., `runAsNonRoot` conflicting with an image that needs root).

---

## 7. `Init:CrashLoopBackOff` / stuck on `Init:0/1`

**What it means:** An init container is failing or blocking, so the main containers never start.

**How to diagnose:**
```bash
kubectl logs <pod-name> -c <init-container-name>
kubectl describe pod <pod-name>
```

**How to resolve:**
- Fix whatever the init container waits for (DB migration, service readiness).
- Ensure the init container's own image/command is correct.

---

## Quick Reference — Exit Codes

| Exit Code | Meaning |
|-----------|---------|
| 0   | Success / clean exit |
| 1   | General application error |
| 125 | Container failed to run (Docker daemon error) |
| 126 | Command not executable |
| 127 | Command not found |
| 137 | SIGKILL — usually OOMKilled |
| 139 | SIGSEGV — segmentation fault |
| 143 | SIGTERM — graceful termination |
