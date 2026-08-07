# Kubernetes Real-World Errors — Networking, Services & Ingress

Networking issues are the hardest to debug because failures are often silent. Here's how to approach the common ones.

---

## 1. Service has no endpoints / traffic not reaching Pods

**Symptom:** `curl <service>` times out or connection refused, even though Pods are Running.

**Root cause (90% of the time):** The Service `selector` doesn't match the Pod `labels`.

**How to diagnose:**
```bash
kubectl get svc <svc-name>
kubectl get endpoints <svc-name>      # empty ENDPOINTS = selector mismatch
kubectl get pods --show-labels
kubectl describe svc <svc-name>
```

**How to resolve:**
- Make the Service `spec.selector` exactly match the Pod labels.
- Confirm `targetPort` matches the container's `containerPort`.
- Verify the Pods are actually `Ready` (a not-ready Pod is excluded from endpoints).

---

## 2. `503 Service Unavailable` from Ingress

**Common causes:**
- Backend Service has no healthy endpoints
- Ingress `backend serviceName`/`port` is wrong
- Ingress controller not running or misconfigured
- Readiness probe failing, so no Pods are in rotation

**How to diagnose:**
```bash
kubectl get ingress
kubectl describe ingress <name>
kubectl get pods -n <ingress-controller-namespace>
kubectl logs -n <ns> <ingress-controller-pod>
```

**How to resolve:**
- Confirm the referenced Service exists and has endpoints.
- Match the Ingress port to the Service port.
- Fix readiness probes so Pods become Ready.

---

## 3. DNS resolution failing inside the cluster

**Symptom:** App can't resolve `my-service` or `my-service.namespace.svc.cluster.local`.

**How to diagnose:**
```bash
kubectl run -it --rm debug --image=busybox:1.28 --restart=Never -- nslookup kubernetes.default
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**How to resolve:**
- Ensure CoreDNS pods are Running and healthy.
- Use the fully qualified name for cross-namespace calls: `svc.<namespace>.svc.cluster.local`.
- Check the Pod's `/etc/resolv.conf` and any custom `dnsPolicy`.
- Look for NetworkPolicies blocking traffic to `kube-dns` on port 53.

---

## 4. NetworkPolicy blocking traffic unexpectedly

**Symptom:** Pods that previously communicated now can't, after a policy was applied.

**Key gotcha:** Once *any* NetworkPolicy selects a Pod, that Pod becomes **default-deny** for the direction(s) specified. You must explicitly allow needed traffic.

**How to diagnose:**
```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <ns>
```

**How to resolve:**
- Add explicit `ingress`/`egress` rules for legitimate traffic (including DNS egress to kube-system on port 53).
- Test iteratively — start permissive, then tighten.

---

## 5. `Connection refused` between Pods

**Common causes:**
- App not listening on `0.0.0.0` (only on `127.0.0.1`, unreachable from other Pods)
- Wrong port
- App still starting up

**How to resolve:**
- Bind the app to `0.0.0.0`, not `localhost`.
- Verify the port with `kubectl exec <pod> -- netstat -tlnp` (or `ss -tlnp`).

---

## 6. LoadBalancer stuck in `<pending>` EXTERNAL-IP

**Common causes:**
- Cluster has no cloud controller / not on a supported cloud
- Cloud quota exhausted
- Missing cloud provider permissions

**How to resolve:**
- On bare-metal, install MetalLB (or use NodePort/Ingress instead).
- On cloud, check quotas and IAM permissions for the load balancer service.
- Inspect events: `kubectl describe svc <name>`.

---

## Debugging Toolkit

```bash
# Spin up a throwaway debug pod with network tools
kubectl run -it --rm netshoot --image=nicolaka/netshoot --restart=Never -- bash

# From inside, test connectivity
curl -v http://<service>.<namespace>.svc.cluster.local
nslookup <service>
dig <service>.<namespace>.svc.cluster.local
```
