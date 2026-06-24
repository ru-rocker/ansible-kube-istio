# Virtual Machine Integration in Istio Ambient Mesh: Compatibility & Architectural Gaps

> [!NOTE]
> This document details the compatibility status, technical limitations, and architectural recommendations regarding virtual machine (VM) onboarding in an Istio Ambient-enabled service mesh.

---

## Compatibility Status Overview

As of current upstream open-source Istio releases (through `v1.31`), **native Virtual Machine (VM) integration for Ambient Mode is not supported**. 

While Kubernetes namespaces can be configured to run in Ambient Mode (using node-level `ztunnels` and optional namespace `waypoints`), VMs must continue to use the **traditional sidecar proxy data plane** to communicate with the mesh.

---

## Key Architectural and Data Plane Gaps

Running a hybrid mesh (Kubernetes in Ambient, VM in Sidecar) exposes several deep architectural gaps:

### 1. No VM Ztunnel (No Shared Node Proxy)
* **Sidecar Requirement:** Ambient mode relies on a shared node-level proxy (`ztunnel`) running as a DaemonSet to offload Layer 4 processing. Since a VM is a standalone operating system instance, it cannot share a cluster daemon. As a result, the VM must still run a dedicated local `istio-proxy` (Envoy) sidecar.
* **No Resource Savings:** Running a sidecar on the VM defeats the core value proposition of Ambient mode (sidecarless L4 resource savings).

### 2. Traffic Protocol Mismatch (HBONE vs. mTLS)
* **HBONE vs. Standard mTLS:** The `ztunnel` in Ambient mode encapsulates and secures traffic using **HBONE** (HTTP-Based Overlay Network Encapsulation, port `15008`). Traditional sidecars, by default, communicate using standard TLS wrappers on the application ports.
* **Waypoint Dependency:** For a VM sidecar to securely communicate with an ambient pod in the cluster, you **must deploy a Waypoint Proxy** in the pod's namespace. The VM sidecar connects to the Waypoint using standard sidecar mTLS, and the Waypoint translates this connection to HBONE before passing it to the target pod's `ztunnel`.

### 3. Control Plane Policy Targeting Conflicts
* **Targeting Mismatch:** When a namespace is labeled `istio.io/dataplane-mode: ambient`, `istiod` configures the security policies (like `AuthorizationPolicy`) to be consumed by the namespace's `ztunnels` and `waypoint` proxies rather than sidecars.
* **RBAC Denials:** Because the VM proxy runs as a standard sidecar Envoy, it connects to XDS but fails to receive the authorization rules targeting its specific selector (since the control plane maps these rules to `ztunnels`). As a result, the VM proxy falls back to a global default-deny posture and returns `403 Forbidden` (RBAC access denied) for inbound traffic.

### 4. Live Verification Test Gaps
During active verification of a hybrid mesh (Kubernetes namespace in Ambient Mode, VM in Sidecar Mode), the following L4/L7 translation gaps were observed:
* **Outbound routing to VM fails (Kube-to-VM)**: Pod to VM connections return `Empty reply from server` (exit code 52). Ztunnel does not support resolving or routing to service endpoints backed purely by custom `WorkloadEntry` resources. Ztunnel logs: `warn access connection failed [...] error="no service for target address"`.
* **Inbound routing to Kube fails (VM-to-Kube)**: VM to pod connections fail with `503 Service Unavailable (no healthy upstream)` at the gateway. The gateway routes standard TLS/mTLS to the pod IP directly on port 80. However, in STRICT mTLS Ambient Mode, the pod's ztunnel expects HBONE-encapsulated traffic (tunneling via port 15008) and rejects standard sidecar mTLS handshakes on port 80.

---

## Recommended Alternatives

Depending on your architecture, choose one of the following paths:

### Option A: Standard Sidecar Mesh (Recommended)
If VM workloads are active participants in your mesh:
* **Action:** Retain standard sidecar injection in target namespaces (`istio-injection: enabled`).
* **Benefit:** Fully supported, stable, out-of-the-box cross-network endpoint sync, and robust policy propagation to the VM sidecars.

### Option B: Namespace-Wide Allowed Rules
If you must run a hybrid mesh (pods in Ambient, VM in Sidecar) and have namespace waypoints deployed:
* **Action:** Omit the workload `selector` block from VM-targeted `AuthorizationPolicies` in Kubernetes.
* **Benefit:** Allows the policy to be pushed namespace-wide, enabling the VM proxy connection (which lacks workload labels due to connection address rewriting across NAT gateways) to receive the rules and successfully authorize requests.

### Option C: Enterprise Distributions
* **Action:** Utilize commercial Istio offerings (such as Solo.io Gloo Mesh).
* **Benefit:** Access proprietary agents that extend ambient features natively to virtual machines.
