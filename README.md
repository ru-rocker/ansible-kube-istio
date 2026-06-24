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
  - `security_guardrails`: Configure strict mTLS and zero-trust default-deny authorization policies.
- [GEMINI.md](file:///Users/ricky/Documents/workspaces/workspace-devops/ansible-kube-istio/GEMINI.md) - Project-specific guardrails, OS/hardware prerequisites, and security standards.

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

- **Istio Installation Only** (sets up istioctl, Helm charts, sidecar CNI plugins):
  ```bash
  ansible-playbook -i inventory/hosts.ini site.yml --tags istio
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
