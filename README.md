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

### 2. Run the Full Deployment
To run the entire installation from preflight checks to security configuration:
```bash
ansible-playbook -i inventory/hosts.ini site.yml
```

If you require custom SSH private keys or sudo passwords, you can append the relevant flags:
```bash
# Using a specific SSH key
ansible-playbook -i inventory/hosts.ini site.yml --private-key=/path/to/your/id_rsa

# Asking for SSH connection password and sudo/become password
ansible-playbook -i inventory/hosts.ini site.yml --ask-pass --ask-become-pass
```

### 3. Dry-Run / Check Mode
To simulate the execution and see what changes would be applied without modifying the targets:
```bash
ansible-playbook -i inventory/hosts.ini site.yml --check
```

### 4. Running Specific Setup Phases (Using Tags)
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
scp root@167.71.219.28:/etc/kubernetes/admin.conf ~/.kube/config-kubeadm

# 2. Export the environment variable for your active shell
export KUBECONFIG=~/.kube/config-kubeadm

# 3. Verify connection to the cluster
kubectl get nodes
```

---

## Deploying Sample Applications

Once you have local `kubectl` access, deploy the resources from the `deployment-sample` directory in the following logical order:

### 1. Mesh Infrastructure (Gateway & Ingress Routing)
These components configure the gateway and policies to securely route external VM traffic to the control plane:

```bash
# Apply the East-West Gateway (for external connectivity)
kubectl apply -f deployment-sample/eastwest-gateway.yaml

# Patch the generated service to expose port 15012 on NodePort 30185 (VM bootstrap port)
kubectl patch service cross-network-gateway-istio -n istio-system --type='merge' -p '{"spec": {"type": "NodePort", "ports": [{"name": "tls-istiod", "port": 15012, "nodePort": 30185}]}}'

# Apply the TLS Route mapping for istiod
kubectl apply -f deployment-sample/istiod-route.yaml

# Apply the authorization policy to allow incoming connection to the gateway
kubectl apply -f deployment-sample/gateway-auth-policy.yaml
```

### 2. VM Auto-Registration Group
Prepare the mesh namespace for receiving auto-registering VM workloads:

```bash
# Deploy the WorkloadGroup definition and ServiceAccount
kubectl apply -f deployment-sample/vm-workloadgroup.yaml

# Deploy the Service that maps to the VM workload selector
kubectl apply -f deployment-sample/vm-service.yaml
```
*(Note: You do not need to apply `vm-workloadentry.yaml` manually since the VM sidecar auto-registers and creates this dynamically upon connecting).*

### 3. Application Workloads & Zero-Trust Policies
Deploy sample microservices and authorization policies that enforce zero-trust communication rules:

```bash
# Deploy the order-service
kubectl apply -f deployment-sample/order-service.yaml

# Deploy the payment-service
kubectl apply -f deployment-sample/payment-service.yaml

# Apply authorization policies (allows communication between the services)
kubectl apply -f deployment-sample/auth-policy.yaml
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

To verify the cross-network and zero-trust service mesh configuration, you can perform testing in the three key communication scenarios. Ensure your terminal has the correct `KUBECONFIG` set up (e.g., `export KUBECONFIG=~/.kube/config-ansible-kube`).

### 1. Kube-to-Kube (kube-kube)
This scenario tests communication between two containerized services inside the Kubernetes cluster (e.g., `order-service` calling `payment-service`). 

Run the following command to execute a `curl` inside an `order-service` pod targeting the `payment-service` ClusterIP:
```bash
kubectl exec -n mesh-services deploy/order-service -c order-service -- curl -s http://payment-service/
```

**Expected Output:**
```json
{"service":"payment-service","status":"ok"}
```
*(Under the hood, the client sidecar proxy intercepts the plaintext call to `http://payment-service/` and wraps it in a secure mutual TLS tunnel directly targeting the destination sidecar proxy).*

### 2. Kube-to-VM (kube-vm)
This scenario tests communication from a containerized service inside the cluster to a service running on the external VM workload (e.g., `order-service` calling `vm-proxy`).

Run the following command to execute a `curl` inside the `order-service` pod targeting the `vm-proxy` service:
```bash
kubectl exec -n mesh-services deploy/order-service -c order-service -- curl -s http://vm-proxy/
```

**Expected Output:**
The default HTML welcome page of the Nginx server running on the VM:
```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```
*(Under the hood, the `order-service` sidecar routes the connection directly to the VM's public IP address on port 80 using strict mTLS. The VM's `iptables` redirects the inbound port 80 traffic to Envoy on port 15006, which terminates the mTLS connection and forwards it locally to the Nginx process).*

### 3. VM-to-Kube (vm-kube)
This scenario tests communication from the external VM workload to a containerized service in the cluster (e.g., the VM calling `order-service`).

SSH into the VM (or execute a remote command via SSH) and run a `curl` targeting the Kubernetes DNS name of the service:
```bash
ssh root@188.166.183.114 "curl -s http://order-service.mesh-services.svc.cluster.local/"
```

**Expected Output:**
```json
{"service":"order-service","status":"ok"}
```
*(Under the hood, the VM's local `systemd-resolved` forwards DNS queries for `*.cluster.local` to Envoy's local DNS capture port `15053`. Envoy resolves the hostname and intercepts the TCP connection, routing it to the cluster's East-West gateway on port 15443. The Gateway performs SNI routing to pass the mTLS connection directly to the target pod sidecar).*

### 4. Ambient-to-VM (ambient-vm) [KNOWN LIMITATION]
This scenario tests communication from a sidecarless workload running in an Ambient-enabled namespace (`meshz-services`) to a VM workload running in a traditional sidecar-enabled namespace (`mesh-services`).

Exec into the `client-service` pod in the `meshz-services` namespace and curl the VM service:
```bash
kubectl exec -n meshz-services deploy/client-service -- curl -s http://vm-proxy.mesh-services.svc.cluster.local/
```

**Expected Output:**
```text
command terminated with exit code 56 (Connection reset by peer)
```

**Description:**
This failure is a known architectural limitation of hybrid Ambient-Sidecar meshes. The Layer 4 `ztunnel` in the client's Ambient namespace does not support resolving or routing to service endpoints backed by custom `WorkloadEntry` resources. Checking the client node's `ztunnel` logs will show:
`warn access connection failed [...] error="no service for target address: <vm-proxy-cluster-ip>:80"`.
For VM connectivity to work, target workloads must remain in standard Sidecar Mode (`istio-injection: enabled`).

---

## Additional Documentation

For more information on virtual machine integration and architectural compatibility in an Istio Ambient Mesh environment, see:
* [VM Ambient Mesh Compatibility & Architectural Gaps](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/docs/vm-ambient-compatibility.md)
