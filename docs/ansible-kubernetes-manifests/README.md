# Ansible Inline Kubernetes Manifests Documentation

This directory contains the standalone Kubernetes YAML resource definitions extracted from the inline `kubernetes.core.k8s` tasks in the Ansible playbooks. These manifests represent core security guardrails, mesh policies, and service routing definitions.

---

## 1. Istio Service Mesh Policies (`roles/security_guardrails`)

### 🛡️ [global-mtls.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/global-mtls.yaml)
* **Resource Kind:** `PeerAuthentication` (`default-mtls`)
* **Namespace:** `istio-system`
* **Purpose:** Sets the global Mutual TLS (mTLS) mode to `STRICT` cluster-wide, ensuring all pods in the mesh only accept encrypted communication.

### 🛡️ [global-default-deny.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/global-default-deny.yaml)
* **Resource Kind:** `AuthorizationPolicy` (`global-default-deny`)
* **Namespace:** `istio-system`
* **Purpose:** Implements a default-deny zero-trust security posture for all traffic inside the service mesh. No communication is permitted unless explicitly allowed by an matching policy.

### 🔓 [vm-namespace-permissive.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/vm-namespace-permissive.yaml)
* **Resource Kind:** `PeerAuthentication` (`vm-namespace-permissive`)
* **Namespace:** `{{ vm_namespace }}` (e.g., `mesh-services`)
* **Purpose:** Sets mTLS mode to `PERMISSIVE` for the VM namespace. This permits plaintext fallback to allow the L7 Waypoint proxy to establish connections to the VM workload endpoints.

### 🧭 [waypoint.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/waypoint.yaml)
* **Resource Kind:** `Gateway` (`waypoint`)
* **Namespace:** `{{ vm_namespace }}`
* **Purpose:** Deploys the Layer 7 Waypoint Proxy inside the target namespace using the `istio-waypoint` GatewayClass. It acts as the L7 policy enforcer for target pods in Ambient Mode.

### ⚙️ [vm-workloadgroup.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/vm-workloadgroup.yaml)
* **Resource Kind:** `WorkloadGroup` (`{{ vm_auto_register_group }}`)
* **Namespace:** `{{ vm_namespace }}`
* **Purpose:** Serves as the template metadata sheet for dynamic auto-registration of VMs connecting to the mesh, mapping identities (`ServiceAccount`), labels, and ports.

---

## 2. Kuma Service Mesh Policies (`roles/security_guardrails` & `roles/kuma`)

### 🛡️ [kuma-mtls-mesh.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/kuma-mtls-mesh.yaml)
* **Resource Kind:** `Mesh`
* **Purpose:** Declares the Kuma Mesh configuration, activating the built-in CA backend and enforcing Mutual TLS (mTLS) for all workloads belonging to the mesh.

### 🛡️ [kuma-default-deny.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/kuma-default-deny.yaml)
* **Resource Kind:** `MeshTrafficPermission` (`deny-all`)
* **Namespace:** `kuma-system`
* **Purpose:** Standard Kuma zero-trust authorization setting that blocks all traffic inside the mesh by default.

---

## 3. Argo CD Routing Configurations (`roles/argo`)

### 🌐 [argo-gateway.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/argo-gateway.yaml)
* **Resource Kind:** `Gateway` (`argo-gateway`)
* **Namespace:** `{{ argo_namespace }}` (e.g. `argocd`)
* **Purpose:** Configures an HTTP port 80 listener bound to the ingress proxy to handle external connections targeted at the Argo CD host interface.

### 🌐 [argo-virtualservice.yaml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/ansible-kubernetes-manifests/argo-virtualservice.yaml)
* **Resource Kind:** `VirtualService` (`argo-virtualservice`)
* **Namespace:** `{{ argo_namespace }}`
* **Purpose:** Configures the routing rule to match HTTP prefix `/` on the `argo-gateway` and securely forward it to the internal `argocd-server` pod service.
