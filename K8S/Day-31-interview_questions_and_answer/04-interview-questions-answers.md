# Kubernetes Interview — Real-World Questions & Answers

Scenario-driven questions that come up in DevOps/SRE/K8s interviews, with answers that show reasoning, not just definitions.

---

### Q1. A Pod is stuck in `CrashLoopBackOff`. Walk me through your debugging.

**A:** First, `kubectl describe pod` to read the Events and container state, then `kubectl logs <pod> --previous` to see logs from the crashed instance — that usually reveals the actual error. I check whether it's an app bug, a missing config/secret, or a too-aggressive liveness probe restarting a healthy app. If dependencies aren't ready, I'd add an init container or retry logic. I confirm the fix by testing the image locally before redeploying.

---

### Q2. Everything says "Running," but the Service returns nothing. Why?

**A:** Most often the Service selector doesn't match the Pod labels, so the Service has zero endpoints. I run `kubectl get endpoints <svc>` — if it's empty, that confirms it. I also verify `targetPort` matches the container port and that Pods are actually `Ready`, since not-ready Pods are excluded from endpoints.

---

### Q3. What's the difference between a liveness and a readiness probe? Why does it matter?

**A:** Liveness decides whether to **restart** a container — it detects a hung/deadlocked app. Readiness decides whether to **send traffic** — it gates the Pod's inclusion in Service endpoints. Misusing them is a classic bug: if you put startup-dependency checks in a liveness probe, Kubernetes keeps killing a perfectly healthy app that's just waiting on a dependency. Slow-starting apps also need a `startupProbe` so the liveness probe doesn't fire prematurely.

---

### Q4. A Pod got `OOMKilled`. What do you do?

**A:** Exit code 137 means it exceeded its memory limit. I check `kubectl describe pod` for the OOMKilled reason and `kubectl top pod` for usage. Then I decide: is the limit too low for a legitimate workload (raise it), or is there a memory leak (profile and fix)? For JVM/Node apps I make sure the runtime respects the container limit — e.g., set max heap as a percentage of the container memory rather than letting it assume host memory.

---

### Q5. How do you do a zero-downtime deployment in Kubernetes?

**A:** Use a RollingUpdate strategy with sensible `maxSurge`/`maxUnavailable`, and — critically — define **readiness probes** so traffic only shifts to Pods that are actually ready. Add a `preStop` hook plus `terminationGracePeriodSeconds` so in-flight requests drain before shutdown. A PodDisruptionBudget protects availability during voluntary disruptions like node drains. For higher safety, blue-green or canary (via a service mesh or Argo Rollouts) lets you shift traffic gradually and roll back fast.

---

### Q6. Explain `requests` vs `limits`. What happens if you set limits but no requests?

**A:** `requests` are what the scheduler reserves and uses to place Pods; `limits` are the hard ceiling the runtime enforces. If you set only limits, Kubernetes defaults requests to equal the limits. CPU over the limit gets **throttled**; memory over the limit gets the container **OOMKilled**. Setting requests too high wastes capacity; too low risks eviction under pressure. Getting these right is core to bin-packing and stability.

---

### Q7. A node goes `NotReady`. What happens to its Pods, and what do you check?

**A:** After the node is unreachable past the toleration window (default ~5 min), the Pods are marked for eviction and rescheduled elsewhere (for controller-managed Pods). I'd `kubectl describe node` for Conditions (disk/memory/PID pressure), then on the node check `systemctl status kubelet` and `journalctl -u kubelet`. Common culprits are kubelet down, CNI failure, resource pressure, or expired certs.

---

### Q8. How does a request travel from the internet to a container?

**A:** External traffic hits a LoadBalancer or Ingress controller → the Ingress rules route by host/path to a Service → the Service (via kube-proxy/iptables/IPVS or eBPF) load-balances across the Pod endpoints → lands on a container port. DNS (CoreDNS) resolves service names to ClusterIPs along the way. Understanding this chain is how you localize a failure to the right layer.

---

### Q9. What is a StatefulSet and when do you need one over a Deployment?

**A:** A StatefulSet gives Pods **stable, ordered identities** — persistent network IDs (`pod-0`, `pod-1`), stable per-Pod storage via `volumeClaimTemplates`, and ordered rollout/scaling. You need it for stateful systems like databases, Kafka, or Zookeeper where identity and storage must survive rescheduling. Deployments are for stateless, interchangeable replicas.

---

### Q10. Your PVC is stuck `Pending`. Diagnose it.

**A:** I check `kubectl describe pvc` for the reason. Usually it's: no matching StorageClass, no PV satisfying the size/accessMode, or `volumeBindingMode: WaitForFirstConsumer` — in which case the PVC intentionally stays Pending until a Pod that uses it is scheduled. I confirm the StorageClass exists and the requested accessMode/size can actually be provisioned.

---

### Q11. How do you troubleshoot DNS not resolving inside the cluster?

**A:** I launch a debug pod and run `nslookup kubernetes.default`. If that fails, I check CoreDNS pods in kube-system are Running and read their logs. I confirm the Pod's `/etc/resolv.conf` and `dnsPolicy`, verify cross-namespace calls use the FQDN, and check no NetworkPolicy is blocking egress to kube-dns on port 53 — that last one bites people after they lock down egress.

---

### Q12. What's the difference between `ClusterIP`, `NodePort`, and `LoadBalancer`?

**A:** `ClusterIP` (default) exposes the Service only inside the cluster. `NodePort` opens a static port on every node so it's reachable externally via `nodeIP:port`. `LoadBalancer` provisions an external cloud load balancer that fronts the Service. In practice, production external access usually goes through an Ingress controller (fronted by one LoadBalancer) rather than a LoadBalancer per service.

---

### Q13. How do ConfigMaps and Secrets differ, and how do you handle secret security?

**A:** Both inject configuration, but Secrets are meant for sensitive data and are base64-encoded (not encrypted by default). To actually secure them: enable **encryption at rest** for etcd, restrict access via RBAC, avoid committing them to Git, and prefer an external manager (Vault, cloud secret managers, External Secrets Operator, or Sealed Secrets for GitOps). Base64 is encoding, not security — that distinction is a common interview trap.

---

### Q14. A rollout went bad in production. How do you roll back quickly?

**A:**
```bash
kubectl rollout undo deployment/<name>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
```
Kubernetes keeps ReplicaSet revision history, so `rollout undo` reverts to the prior good ReplicaSet. I'd verify readiness afterward. Longer term, canary/blue-green with automated rollback (Argo Rollouts, Flagger) catches bad releases before full exposure.

---

### Q15. What are taints and tolerations vs node affinity?

**A:** Taints/tolerations are **repelling** — a taint keeps Pods off a node unless they tolerate it (e.g., dedicated GPU nodes, control plane). Node affinity is **attracting** — it draws Pods toward nodes with matching labels. They solve opposite sides: taints say "don't schedule here unless allowed," affinity says "prefer/require scheduling here." They're often combined for dedicated node pools.

---

### Q16. How would you limit the blast radius of a compromised Pod?

**A:** Layer defenses: RBAC with least privilege for ServiceAccounts, NetworkPolicies to restrict Pod-to-Pod traffic (default-deny then allow), `securityContext` with `runAsNonRoot`, dropped Linux capabilities, read-only root filesystem, Pod Security Admission (baseline/restricted), resource limits to prevent noisy-neighbor abuse, and image scanning + signed images in the pipeline. Defense in depth, not one control.

---

### Q17. Pods are `Pending` and won't schedule. What are the possibilities?

**A:** In order of likelihood: insufficient CPU/memory across nodes, unsatisfied node affinity/selector, taints without matching tolerations, a PVC that can't bind, or no available nodes (autoscaler still scaling). `kubectl describe pod` spells out the exact reason under Events (e.g., "0/3 nodes available: insufficient memory").

---

### Q18. Explain the difference between `kubectl apply`, `create`, and `replace`.

**A:** `create` is imperative and fails if the object exists. `replace` fully overwrites an existing object (and needs the whole spec). `apply` is declarative — it does a three-way merge against the last-applied config, creating or updating as needed, which is why it's the standard for GitOps and repeatable infra. `apply` also tracks intent via annotations so it can reconcile drift.

---

### Q19. How do you handle application configuration that differs per environment?

**A:** Keep the image identical across environments and inject config externally — ConfigMaps/Secrets per environment, with tooling like Kustomize overlays or Helm values files to manage the differences. This preserves the "build once, deploy anywhere" principle and avoids environment-specific images. Secrets come from an external manager per environment.

---

### Q20. What happens end-to-end when you run `kubectl apply -f deployment.yaml`?

**A:** kubectl authenticates to the API server, which validates and persists the Deployment object in etcd. The Deployment controller creates a ReplicaSet; the ReplicaSet controller creates Pod objects. The scheduler assigns each Pod to a node based on resources/affinity/taints. The kubelet on that node pulls the image and starts the container via the container runtime (CRI). The CNI wires up networking, kube-proxy programs Service routing, and probes gate readiness. Controllers continuously reconcile actual state toward desired state — that reconciliation loop is the heart of Kubernetes.

---

## Rapid-Fire Concepts

- **What keeps desired state?** Controllers via the reconciliation loop.
- **Where is cluster state stored?** etcd.
- **What load-balances Service traffic on nodes?** kube-proxy (iptables/IPVS) or eBPF (Cilium).
- **What resolves service DNS?** CoreDNS.
- **What's the smallest deployable unit?** A Pod.
- **How do you scale automatically?** HPA (pods, on metrics), VPA (right-sizing), Cluster Autoscaler (nodes).
- **What guarantees availability during node drains?** PodDisruptionBudget.
- **How do containers in a Pod share data?** Shared network namespace + shared volumes.
