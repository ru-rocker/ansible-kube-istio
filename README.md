# Kubernetes & Istio Deployment with Ansible

This repository contains Ansible playbooks and roles to deploy a production-ready Kubernetes cluster and configure the Istio service mesh with zero-trust security guardrails.

---

## Workspace Structure

- [site.yml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/site.yml) - The main Ansible entrypoint playbook.
- [inventory/hosts.ini](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/inventory/hosts.ini) - Target host definitions and roles (control plane vs. worker nodes).
- [group_vars/all.yml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/group_vars/all.yml) - Configuration variables (RAM/CPU/disk thresholds, Kubernetes/Istio versions, security modes).
- [roles/](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/roles) - Modular roles for setup and configuration:
  - `preflight`: Pre-checks to validate kernel modules, sysctl parameters, OS, swap status, and resource limits.
  - `kubernetes`: Setup control plane and worker nodes via kubeadm.
  - `istio`: Install Istioctl/Helm templates for service mesh initialization.
  - `istio_vm`: Install Nginx runtime and Istio sidecar proxy on VMs.
  - `security_guardrails`: Configure strict mTLS and zero-trust default-deny authorization policies.
  - `kuma`: Deploy Kuma Control Plane using Helm on Kubernetes.
  - `kuma_vm`: Setup and register VM workloads with Kuma in Universal Mode.
  - `argo`: Install Argo CD and Argo Rollouts via Helm with sidecar injection and ingress gateway routing.
- [GEMINI.md](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/GEMINI.md) - Project-specific guardrails, OS/hardware prerequisites, and security standards.
- [docs/vm-ambient-compatibility.md](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/vm-ambient-compatibility.md) - Compatibility status, gaps, and alternatives for VM integration in Istio Ambient Mesh.
- [docs/kuma_kong_mesh_comparison.md](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/kuma_kong_mesh_comparison.md) - Architectural and cost comparison of Kuma and Kong Mesh vs. Istio.


---

## Prerequisites & Requirements

Before running the playbook, verify that your hosts meet the following minimum targets (enforced during the `preflight` stage):
- **OS**: Ubuntu 22.04+, Debian 11+, or RHEL 8+
- **Hardware**: Minimum 1 CPU core, 1 GB RAM, 10 GB free disk space
- **Networking**: Keep ports `6443` (API Server) and Istio ports (e.g. `15017`, `15021`) free
- **Privilege**: Tasks require root privilege (`become: true` is configured in `site.yml`)

---

## How to Run the Playbook

### 1. Configure the Inventory
Verify the target node IPs and SSH connection details in [inventory/hosts.ini](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/inventory/hosts.ini):
```ini
[all:vars]
ansible_user=ubuntu
ansible_become=true

[control_plane]
master-node ansible_host=192.168.1.10

[workers]
worker-node-1 ansible_host=192.168.1.11
worker-node-2 ansible_host=192.168.1.12

[k8s_cluster:children]
control_plane
workers
```

### 2. Configure the Active Mesh Type
In [group_vars/all.yml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/group_vars/all.yml), configure which service mesh you want to deploy:
```yaml
# Set to "istio" or "kuma"
mesh_type: "istio"
```

### 3. Run the Full Deployment
To run the entire installation from preflight checks to security configuration for your chosen mesh:
```bash
ansible-playbook -i inventory/hosts.ini site.yml
```
*Note: The playbook dynamically loads the correct control plane and VM proxy roles matching your active `mesh_type` configuration.*

If you require custom SSH private keys or sudo passwords, you can append the relevant flags:
```bash
# Using a specific SSH key
ansible-playbook -i inventory/hosts.ini site.yml --private-key=/path/to/your/id_rsa

# Asking for SSH connection password and sudo/become password
ansible-playbook -i inventory/hosts.ini site.yml --ask-pass --ask-become-pass
```


### 4. Dry-Run / Check Mode
To simulate the execution and see what changes would be applied without modifying the targets:
```bash
ansible-playbook -i inventory/hosts.ini site.yml --check
```

### 5. Running Specific Setup Phases (Using Tags)
The playbook uses structured tags for targeted executions:

- **Pre-flight Checks Only** (validates OS, resource limits, swap, ports, sysctl, kernel modules):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags preflight
  ```

- **Kubernetes Cluster Setup Only** (sets up kubelet, kubeadm, initiates control plane, joins nodes):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags kubernetes
  ```

- **Istio Control Plane Setup Only** (sets up Helm charts and control plane for Istio Ambient Profile):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags istio
  ```

- **Istio VM Proxy Setup Only** (onboards VM runtime with Nginx and Istio sidecar proxy):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags vm_proxy
  ```

- **Kuma Control Plane Setup Only** (deploys Kuma Control Plane using Helm on Kubernetes):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags kuma
  ```

- **Kuma VM Proxy Setup Only** (onboards VM runtime with Nginx and Kuma data plane proxy in Universal mode):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags kuma_vm
  ```

- **Security Policy/Guardrails Only** (enforces namespace isolation labels, strict mTLS, zero-trust rules):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags security
  ```

- **Argo CD & Argo Rollouts Setup Only** (downloads Helm charts, sets up sidecar-injected namespaces, installs controllers, and configs ingress routing):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags argo
  ```

- **All Guardrails Validation** (combines preflight checks and security guardrails):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags guardrails
  ```



### 5. Overriding Configuration Variables
You can customize variables defined in [group_vars/all.yml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/group_vars/all.yml) on the fly using `--extra-vars`:
```bash
# E.g. Deploying a different version of Kubernetes or disabling automatic swap handling
ansible-playbook -i inventory/hosts.ini site.yml --extra-vars "kubernetes_version=1.29.0 disable_swap_automatically=false"
```

---

## Accessing the Cluster Locally

To manage the Kubernetes cluster directly from your local machine, retrieve the administration kubeconfig file using `scp` and export the `KUBECONFIG` environment variable:

```bash
# 1. Copy the admin config from the control plane node
scp root@[contro_plane_address]:/etc/kubernetes/admin.conf ~/.kube/config-kubeadm

# 2. Export the environment variable for your active shell
export KUBECONFIG=~/.kube/config-kubeadm

# 3. Verify connection to the cluster
kubectl get nodes
```

---

## Deploying Sample Applications

Once you have local `kubectl` access, deploy the resources from the `deployment-sample` directory. The installation commands depend on which service mesh you are running:

---

### Scenario A: Deploying under Istio (`mesh_type: "istio"`)

#### 1. Common Application Workloads
Deploy the microservices (order-service and payment-service) and the curl verification client:
```bash
# Apply common microservices
kubectl apply -f deployment-sample/common/order-service.yaml
kubectl apply -f deployment-sample/common/payment-service.yaml
kubectl apply -f deployment-sample/common/ambient-client.yaml
```

#### 2. Istio-Specific Ingress & VM Registration
Configure the East-West Gateway, TLS routing, and VM WorkloadGroup for registration:
```bash
# Apply the East-West Gateway
kubectl apply -f deployment-sample/istio/eastwest-gateway.yaml

# Patch the gateway to expose the bootstrap port (15012) on NodePort 30185
kubectl patch service cross-network-gateway-istio -n istio-system --type='merge' -p '{"spec": {"type": "NodePort", "ports": [{"name": "tls-istiod", "port": 15012, "nodePort": 30185}]}}'

# Apply the TLS Route mapping for istiod
kubectl apply -f deployment-sample/istio/istiod-route.yaml

# Allow external traffic to hit gateway ports
kubectl apply -f deployment-sample/istio/gateway-auth-policy.yaml

# Deploy VM WorkloadGroup & ServiceAccount for auto-registration
kubectl apply -f deployment-sample/istio/vm-workloadgroup.yaml

# Deploy Kubernetes service pointing to the VM selector
kubectl apply -f deployment-sample/istio/vm-service.yaml
```

#### 3. Istio Zero-Trust Authorization Policies
Enforce strict rules allowing traffic only along validated paths:
```bash
# Apply authorization policies
kubectl apply -f deployment-sample/istio/auth-policy.yaml
```

---

### Scenario B: Deploying under Kuma (`mesh_type: "kuma"`)

#### 1. Common Application Workloads
Deploy the microservices and client pod:
```bash
# Apply common microservices
kubectl apply -f deployment-sample/common/order-service.yaml
kubectl apply -f deployment-sample/common/payment-service.yaml
kubectl apply -f deployment-sample/common/ambient-client.yaml
```

#### 2. Kuma-Specific Zero-Trust Traffic Permissions
Deploy the `MeshTrafficPermission` resources to allow communication through the mesh's default-deny rule:
```bash
# Apply Kuma traffic permissions
kubectl apply -f deployment-sample/kuma/traffic-permissions.yaml
```

---

## Architectural Deep Dive: East-West Gateway

In an Istio service mesh, gateways route traffic at the edge of the mesh:
* **North-South (Ingress/Egress):** Handles client-to-service traffic entering or exiting the mesh.
* **East-West:** Handles service-to-service traffic crossing logical network, cluster, or platform boundaries.

In our hybrid Kubernetes-VM setup, the `cross-network-gateway` (labeled `istio: eastwestgateway`) serves two critical roles:

### 1. Control Plane Bootstrap (Port `15012`)
To onboard onto the mesh, the VM's `pilot-agent` must securely request certificates and pull configurations (xDS) from the control plane (`istiod`). 
* The VM sends TLS requests addressed to `istiod.istio-system.svc` on NodePort `30185`.
* The Gateway intercepts these packets on port `15012` and uses **TLS Passthrough (SNI-based routing)** to proxy the raw handshake bytes directly to the `istiod` pod in the cluster.
* Because it runs in `Passthrough` mode, the Gateway itself does not decrypt the token or cert payload, keeping the credential exchange fully encrypted end-to-end.

**The Role of the `TLSRoute` (`istiod-route.yaml`):**
A `Gateway` resource only defines the listening ports and protocols; it does not know where to direct traffic. The `istiod-route` resource (a `TLSRoute`) binds to the gateway's `tls-istiod` listener and instructs it to inspect the **SNI (Server Name Indication)** header in the TLS Client Hello handshake. When it matches `istiod.istio-system.svc`, it forwards the encrypted connection directly to the `istiod` service on port `15012` in the cluster. Without this route, the gateway has no destination for the traffic and will omit the listener entirely.

### 2. Cross-Network Data Plane Communication (Port `15443`)
Once the VM successfully registers, workloads on the VM must be able to securely communicate with Kubernetes workloads (and vice-versa).
* All internal service-to-service traffic crossing the boundary is routed through the Gateway on port `15443`.
* This traffic is encrypted using strict mutual TLS (mTLS). The Gateway inspects the SNI header to route the connection to the correct target container without decrypting the payload, preserving a zero-trust posture across network zones.

### 3. Gateway Ingress Authorization (`gateway-auth-policy.yaml`)
To enforce a "zero-trust" security architecture, the playbooks deploy a global default-deny authorization policy (`global-default-deny`) in the `istio-system` namespace. By default, this policy blocks all traffic to all workloads in the namespace—including the `cross-network-gateway-istio` pods.
* **The Role of the Policy:** `gateway-auth-policy.yaml` defines a target-specific `AuthorizationPolicy` matching workloads labeled `istio: eastwestgateway`. It uses `action: ALLOW` with an empty rule (`rules: - {}`) to explicitly permit external, unauthenticated traffic (like the VM's bootstrap connection) to pass through the gateway ports. Without this policy, the gateway's Envoy proxy blocks the incoming TCP stream immediately during the TLS handshake, causing a connection failure.

---

## Scaling to Multiple VMs

A common question when scaling hybrid deployments is: **"If I add more VMs, do I need to open more ports on the cluster?"**

The answer is **no**. You only need to expose the same two ports (`30185` and `15443`) regardless of how many VMs join the mesh.

### How Port Sharing Works:
1. **Shared Ingress Endpoints:**
   * All VMs connect to the same Bootstrap port (`30185` / container port `15012`) to register and fetch configurations.
   * All VMs route service-to-service data traffic through the same Data Plane port (`15443`).
2. **Workload Isolation & Discovery:**
   * **Unique IPs:** Each VM connects using its own network IP address.
   * **Dynamic WorkloadEntries:** The `pilot-agent` on each VM auto-registers with `istiod` on startup. This dynamically creates a unique `WorkloadEntry` resource in Kubernetes (e.g., `vm-proxy-group-<VM-IP>`) containing the VM's specific IP and metadata labels.
    * **SNI-Based Routing:** For data plane traffic on port `15443`, the Gateway uses the Server Name Indication (SNI) header in the mTLS handshake to route the packets to the correct VM instance without needing individual port allocations.

---
## Testing Connectivity Between Mesh Services

Choose the instructions matching your configured mesh:

---

### Scenario A: Testing Connectivity under Istio

#### 1. Kube-to-Kube (kube-kube)
Tests direct communication from `order-service` to `payment-service` inside the cluster:
```bash
kubectl exec -n mesh-services deploy/order-service -c order-service -- curl -s http://payment-service/
```
**Expected Output:**
```json
{"service":"payment-service","status":"ok"}
```

#### 2. Kube-to-VM (kube-vm)
Tests communication from `order-service` inside the cluster to the Nginx VM service:
```bash
kubectl exec -n mesh-services deploy/order-service -c order-service -- curl -s http://vm-proxy/
```
**Expected Output:**
Nginx welcome HTML page.

#### 3. VM-to-Kube (vm-kube)
Tests direct communication from the external VM to `order-service` inside the cluster:
```bash
ssh root@<VM-IP> "curl -s http://order-service.mesh-services.svc.cluster.local/"
```
**Expected Output:**
```json
{"service":"order-service","status":"ok"}
```

#### 4. Transit Path Verification (VM -> K8s -> K8s)
Tests calling the payment processor via the transit service from the VM:
```bash
ssh root@<VM-IP> "curl -s http://order-service.mesh-services.svc.cluster.local/pay"
```
**Expected Output:**
```json
{"service":"payment-service","transaction":"processed","mTLS":"ztunnel-secured"}
```

#### 5. Ambient-to-VM (ambient-vm) [KNOWN LIMITATION]
Tests communication from Ambient `client-service` (in `meshz-services` namespace) to the VM:
```bash
kubectl exec -n meshz-services deploy/client-service -- curl -s http://vm-proxy.mesh-services.svc.cluster.local/
```
**Expected Output:**
`command terminated with exit code 56 (Connection reset by peer)`. This is a known limitation since `ztunnel` doesn't resolve VM WorkloadEntries.

---

### Scenario B: Testing Connectivity under Kuma

In Kuma, all workloads in `mesh-services` and `meshz-services` namespaces are equipped with Kuma sidecar proxies. Inter-service DNS names use the `<service-name>_<namespace>_svc_<port>.mesh` naming convention.

#### 1. Kube-to-Kube (kube-kube)
Tests communication from the `client-service` pod (in `meshz-services`) to the `order-service` pod (in `mesh-services`):
```bash
kubectl exec -n meshz-services deploy/client-service -c client-service -- \
  curl -s http://order-service_mesh-services_svc_80.mesh
```
**Expected Output:**
```json
{"service":"order-service","status":"ok"}
```

#### 2. Kube-to-VM (kube-vm)
Tests communication from a pod to the Kuma VM proxy (`nginx-vm-kuma`) directly:
```bash
kubectl exec -n meshz-services deploy/client-service -c client-service -- \
  curl -s http://nginx-vm-kuma.mesh/
```
**Expected Output:**
Nginx welcome HTML page from the VM.

#### 3. Transit Path Verification: Kube-to-VM via Order Service
Tests calling the VM proxy via the `/vm` endpoint on `order-service` (acting as a reverse-proxy transit pod inside the cluster):
```bash
kubectl exec -n meshz-services deploy/client-service -c client-service -- \
  curl -s http://order-service_mesh-services_svc_80.mesh/vm
```
**Expected Output:**
Nginx welcome HTML page from the VM.

#### 4. VM-to-Kube (vm-kube)
Tests communication from the Kuma VM directly to `order-service` in the cluster:
```bash
ssh root@<VM-IP> "curl -s http://order-service_mesh-services_svc_80.mesh"
```
**Expected Output:**
```json
{"service":"order-service","status":"ok"}
```

#### 5. Transit Path Verification: VM-to-Kube Payment
Tests calling the payment processor path `/pay` on `order-service` from the VM (proxies traffic from VM to `order-service` in k8s, which then calls `payment-service` in k8s):
```bash
ssh root@<VM-IP> "curl -s http://order-service_mesh-services_svc_80.mesh/pay"
```
**Expected Output:**
```json
{"service":"payment-service","transaction":"processed","mTLS":"ztunnel-secured"}
```

---

## Deploying & Securing Argo CD & Argo Rollouts

The repository includes a dedicated `argo` role that installs Argo CD and Argo Rollouts, integrating them directly with the active service mesh (`mesh_type: "istio"` or `mesh_type: "kuma"`).

### 1. Variables Configuration
Ensure `argo_enabled: true` is configured in [group_vars/all.yml](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/group_vars/all.yml). You can customize versions and settings as needed.

### 2. Execution
Run the playbook targeting the `argo` tag to initialize the controllers and components:
```bash
ansible-playbook -i inventory/hosts.ini site.yml --tags argo
```

### 3. Service Mesh & Zero-Trust Policies
When the main playbook or the `security` tag is run, zero-trust policies for the Argo namespaces are applied:
- **Strict mTLS:** The `argocd` and `argo-rollouts` namespaces are configured with strict mTLS (`mode: STRICT` in Istio).
- **Intra-Namespace Communication:** Zero-trust namespace boundary policies are created (e.g. `AuthorizationPolicy` in Istio / `MeshTrafficPermission` in Kuma) to allow internal components like the server to talk to Redis and repo-server.
- **Console Ingress (Istio only):** Deploys an Istio `Gateway` and `VirtualService` targeting the `cross-network-gateway` on port `80` to route external requests for `argo_cd_server_host` (e.g., `argocd.example.com`) directly to `argocd-server`.

---

## Additional Documentation

For more information on virtual machine integration and architectural compatibility in an Istio Ambient Mesh environment, see:
* [VM Ambient Mesh Compatibility & Architectural Gaps](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/vm-ambient-compatibility.md)
* [Kuma Architecture & Kong Mesh Comparison](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/kuma_kong_mesh_comparison.md)
