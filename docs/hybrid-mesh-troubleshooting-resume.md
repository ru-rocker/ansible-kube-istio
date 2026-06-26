# Hybrid Service Mesh VM Onboarding: Troubleshooting Resume

This document compiles the issues, root causes, and fixes encountered during the VM onboarding process under both **Kuma** and **Istio** architectures.

---

## 1. Kuma (Universal Mode VM Integration)

### Issue A: Control Plane Advertised xDS Port Mismatch
* **Symptom:** Kubernetes pods inside the mesh fail to bootstrap or synchronize configuration when the external xDS port parameter is changed.
* **Root Cause:** By default, Kuma control plane (`kuma-cp`) advertises port `5678` for bootstrap. When configuring the external VM to connect via NodePort `30578`, modifying the control plane to advertise port `30578` broke internal pods because the default Kubernetes Service was only listening on `5678`.
* **Fix:** Patched the `kuma-control-plane` Kubernetes service to listen on both port `5678` (for internal pods) and port `30578` (matching the NodePort/advertised port for the VM).

### Issue B: Hybrid Proxy Authenticator Mismatch
* **Symptom:** Kuma dataplane agent on the VM fails to register and gets authentication errors.
* **Root Cause:** In Kubernetes mode, Kuma default proxy authentication is `serviceAccountToken`, which requires projected tokens from Kubernetes pods. Universal VM proxies present Kuma JWT tokens (`dpToken`), which the Kubernetes token reviewer rejects.
* **Fix:** Updated the Kuma control plane's configuration `KUMA_DP_SERVER_AUTHN_DP_PROXY_TYPE` to `"none"` in the Helm values, allowing both Kubernetes service account tokens and Universal VM tokens to be accepted.

### Issue C: DNS Resolution Failures on VM (.mesh domains)
* **Symptom:** Workloads on the VM cannot resolve `.mesh` domain names.
* **Root Cause:** Kuma's local agent (`kuma-dp`) requires CoreDNS to perform DNS interception on VMs. The playbook initially did not install CoreDNS or configure transparent proxying.
* **Fix:** Updated the playbook to extract the `coredns` binary included in the Kuma installation tarball, copy it to `/usr/local/bin`, and enable the `--redirect-dns` flag in `kumactl install transparent-proxy` on the VM.

### Issue D: Flat Routing Network Failures (Connection Reset)
* **Symptom:** VM proxy receives pod IPs from Envoy but calls time out or return connection resets.
* **Root Cause:** The VM node lacks routes to the private Kubernetes pod CIDRs (e.g., `10.244.x.x`).
* **Fix:** Added static routes on the VM (`10.244.0.0/24` and `10.244.1.0/24`) routing via the Kubernetes worker nodes' private VPC IPs.

---

## 2. Istio (Ambient Mesh VM Integration)

When integrating a sidecar-based VM workload with a Kubernetes namespace running in **Istio Ambient Mode**, the following architectural limitations apply:

### Issue A: Traffic Protocol Mismatch (HBONE vs. Sidecar mTLS)
* **Symptom:** Direct communication from cluster pods to VM or vice-versa fails.
* **Root Cause:** Ambient mode uses node-level `ztunnels` communicating via **HBONE** (HTTP-Based Overlay Network Encapsulation, port `15008`). VMs run traditional Envoy sidecars using standard mTLS on application ports. Ztunnel does not natively translate standard sidecar mTLS to HBONE or resolve `WorkloadEntry` destinations.
* **Fix:** 
  1. **Option A (Stable):** Revert the Kubernetes namespace to standard sidecar injection (`istio-injection: enabled`).
  2. **Option B (Hybrid Waypoint):** Keep Ambient mode active but deploy namespace **Waypoint Proxies** to translate the VM's standard mTLS traffic to HBONE before it reaches the destination pods.

### Issue B: Authorization Policy Propagation (RBAC Denials)
* **Symptom:** Calls to the VM sidecar return `403 Forbidden` (RBAC access denied).
* **Root Cause:** In Ambient namespaces, `istiod` configures `AuthorizationPolicy` resources to target node-level ztunnels and waypoints. The VM sidecar proxy receives the xDS stream but lacks rules matching its workload selector.
* **Fix:** Remove the workload `selector` block from the `AuthorizationPolicy` in Kubernetes, allowing the rules to apply namespace-wide so they are correctly propagated to the VM sidecar.
