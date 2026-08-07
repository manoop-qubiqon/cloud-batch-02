# Kubernetes Real-World Errors — Storage, Nodes & Cluster

---

## STORAGE

### 1. PVC stuck in `Pending`

**Common causes:**
- No StorageClass, or the named StorageClass doesn't exist
- No PV available that matches the request (size/accessMode)
- `volumeBindingMode: WaitForFirstConsumer` — PVC binds only once a Pod using it is scheduled

**How to diagnose:**
```bash
kubectl get pvc
kubectl describe pvc <name>
kubectl get storageclass
kubectl get pv
```

**How to resolve:**
- Set a valid `storageClassName` or a default StorageClass.
- Match `accessModes` (e.g., `ReadWriteOnce`) and requested size to an available PV.
- With `WaitForFirstConsumer`, create the consuming Pod — binding follows scheduling.

---

### 2. `Multi-Attach error` — volume already attached to another node

**Symptom:** Pod won't start; event says the RWO volume is attached elsewhere.

**Cause:** A `ReadWriteOnce` volume can only attach to one node. Usually happens when a Pod is rescheduled to a new node before the old attachment is released (common during node failure).

**How to resolve:**
- Wait for the old Pod/attachment to terminate fully.
- For stuck cases, delete the terminating Pod: `kubectl delete pod <name> --grace-period=0 --force`.
- Use `ReadWriteMany` storage (e.g., NFS, CephFS) if multiple Pods truly need shared access.

---

### 3. Pod can't write to mounted volume (permission denied)

**Cause:** Volume ownership doesn't match the container's user.

**How to resolve:**
- Set `securityContext.fsGroup` so the volume is group-owned correctly:
  ```yaml
  securityContext:
    fsGroup: 2000
  ```
- Match `runAsUser`/`runAsGroup` to what the app expects.

---

## NODES

### 4. Node `NotReady`

**Common causes:**
- kubelet crashed or stopped
- Network plugin (CNI) failure
- Disk/memory/PID pressure
- Node lost connectivity to the control plane

**How to diagnose:**
```bash
kubectl get nodes
kubectl describe node <node-name>    # check Conditions & Events
# On the node:
systemctl status kubelet
journalctl -u kubelet -f
```

**How to resolve:**
- Restart kubelet: `systemctl restart kubelet`.
- Free up disk/memory if under pressure.
- Verify the CNI plugin pods are healthy.
- Check kubelet certs haven't expired.

---

### 5. `Node has disk pressure` / evicted Pods

**Symptom:** Pods evicted with reason `Evicted`, message about disk/memory pressure.

**How to resolve:**
```bash
kubectl describe node <node> | grep -A5 Conditions
# Clean up:
crictl rmi --prune          # remove unused images
# Investigate disk usage on the node
df -h
```
- Increase node disk, prune images/logs, and set eviction thresholds sensibly.
- Add resource requests so the scheduler spreads load.

---

### 6. Taints preventing scheduling

**Symptom:** Pods `Pending` with "node(s) had taint that the pod didn't tolerate."

**How to resolve:**
```bash
kubectl describe node <node> | grep Taints
# Remove a taint:
kubectl taint nodes <node> key=value:NoSchedule-
# Or add a toleration to the Pod spec.
```

---

## CLUSTER / CONTROL PLANE

### 7. `kubectl` — `Unable to connect to the server`

**Common causes:**
- Wrong/expired kubeconfig context
- API server down
- Expired certificates

**How to resolve:**
```bash
kubectl config current-context
kubectl config get-contexts
kubectl cluster-info
# Check control plane pods (if accessible):
kubectl get pods -n kube-system
```
- Point to the correct context, refresh credentials, or renew certs (`kubeadm certs renew all`).

### 8. `error: You must be logged in to the server (Unauthorized)` / `Forbidden`

**Cause:** RBAC — the user/ServiceAccount lacks permission, or credentials are invalid.

**How to resolve:**
```bash
kubectl auth can-i <verb> <resource> --as=<user>
kubectl describe rolebinding,clusterrolebinding -A | grep <subject>
```
- Bind the appropriate Role/ClusterRole to the subject.
- Refresh expired tokens/certs for `Unauthorized`.

---

### 9. etcd / control plane pressure

**Symptoms:** Slow API responses, timeouts, leader elections flapping.

**How to resolve:**
- Check etcd health and disk latency (etcd is very sensitive to slow disks — use SSDs).
- Monitor etcd DB size; compact and defrag if bloated.
- Ensure control plane nodes aren't resource-starved.
