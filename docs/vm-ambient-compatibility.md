# Istio Ambient Mesh ↔ VM Sidecar Integration Guide

This document details the architectural compatibility, step-by-step setup instructions, and troubleshooting steps for connecting Kubernetes workloads in **Ambient Mode** to virtual machine (VM) workloads running a traditional **Istio sidecar proxy**.

---

## 1. Compatibility & Connectivity Status

### **Status: Fully Working** ✅
Kubernetes workloads in Ambient namespaces can communicate with VM workloads running a traditional sidecar proxy. However, because VM workloads are outside the flat cluster network, the integration requires a specific translation layout.

### How Traffic Flows:
```
Client Pod (Ambient Namespace)
  └── [mTLS/HBONE] ──> Waypoint Proxy (L7 policy enforcement)
                         └── [Plaintext HTTP/80] ──> VM Sidecar (PERMISSIVE mode)
                                                       └── [Localhost] ──> Application (Nginx)
```

1. **Client to Waypoint:** The client pod's `ztunnel` captures the traffic and routes it securely to the **Waypoint Proxy** using HBONE (`15008`).
2. **Waypoint to VM:** Since the VM uses a public or external IP outside the cluster's pod CIDR, the Waypoint drops back to plaintext HTTP/80 (as ztunnel cannot establish an HBONE tunnel to the VM's public IP).
3. **VM Inbound:** The VM sidecar is set to `PERMISSIVE` mTLS mode to accept this plaintext traffic and proxy it to the local application.

---

## 2. Configuration Setup Guide

To successfully configure this hybrid mesh, follow these settings:

### A. VM Configuration (`/var/lib/istio/envoy/cluster.env`)
Ensure the VM sidecar enables HBONE capabilities in its bootstrap environment:
```ini
ISTIO_META_ENABLE_HBONE=true
```
After editing, restart the Istio service on the VM:
```bash
sudo systemctl restart istio
```

### B. WorkloadGroup Labeling
To prevent the control plane from treating the VM as an Ambient node (which would break sidecar configuration delivery), set the dataplane mode to `none` inside your `WorkloadGroup` specification:
```yaml
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
  name: vm-proxy-group
  namespace: mesh-services
spec:
  metadata:
    labels:
      app: vm-proxy
      version: v1
      istio.io/dataplane-mode: none  # Enforces traditional sidecar mode
```

### C. Set Namespace PeerAuthentication to PERMISSIVE
Since the Waypoint proxy connects to the VM's external IP via plaintext, you must override the global `STRICT` mTLS setting for the VM namespace:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: vm-namespace-permissive
  namespace: mesh-services
spec:
  mtls:
    mode: PERMISSIVE
```
> [!IMPORTANT]
> The VM sidecar connects from a NAT'd environment where its workload labels (e.g. `app: vm-proxy`) cannot be resolved by `istiod` for PeerAuthentication. Thus, you **must not** use a workload `selector` block on this policy. It must apply namespace-wide.

### D. Update the Authorization Policy
Because the Waypoint → VM connection is plaintext, it does not present a SPIFFE client identity. The Authorization Policy needs a rule to allow unauthenticated traffic on the application port:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: vm-proxy-policy
  namespace: mesh-services
spec:
  action: ALLOW
  rules:
  - from:                              # Rule 1: Authenticated mTLS mesh mesh pods
    - source:
        principals:
        - "cluster.local/ns/mesh-services/sa/waypoint"
    to:
    - operation:
        ports: ["80"]
  - to:                                # Rule 2: Unauthenticated plaintext from waypoint
    - operation:
        ports: ["80"]

### E. Network Routing Setup (Private Subnet)
For the VM to route traffic back into the Kubernetes cluster networks without a public load balancer on the gateway:
1. Identify a Kubernetes node sharing a private subnet interface (e.g. `eth1` in `10.130.0.0/16`) with the VM.
2. Configure static routes on the VM to route the Kubernetes Pod CIDR (`10.244.0.0/16`) and Service CIDR (`10.96.0.0/12`) using the node's private IP as the next-hop gateway:
   ```bash
   sudo ip route add 10.244.0.0/16 via <node-private-ip> dev eth1
   sudo ip route add 10.96.0.0/12 via <node-private-ip> dev eth1
   ```
*(Note: These routing tasks are now dynamically handled and persisted by the VM Ansible provisioning playbook).*
```

---

## 3. Troubleshooting & FAQ (Errors Encountered)

### ❌ FAQ 1: `upstream connect error or disconnect/reset before headers. reset reason: connection termination`
* **Symptom:** Curl commands return `Empty reply from server` or exit code `52`.
* **Root Cause:** The global mesh policy enforces `STRICT` mTLS. Because the VM is targetable only via a public IP, the Waypoint sends plaintext HTTP on port 80. The VM sidecar expects mTLS, so it terminates the TCP connection immediately.
* **Solution:** Deploy the `PERMISSIVE` PeerAuthentication policy for the VM's namespace (mesh-services).

### ❌ FAQ 2: `RBAC: access denied`
* **Symptom:** The connection succeeds at the TCP level, but returns HTTP 403 with `RBAC: access denied` in the response body.
* **Root Cause:** A default-deny policy (like `global-default-deny` in `istio-system`) blocks traffic by default. The local policy (`vm-proxy-policy`) only allowed requests presenting authenticated SPIFFE identities. Because the connection is plaintext, no identity was sent.
* **Solution:** Add a second rule to the VM's AuthorizationPolicy (`vm-proxy-policy`) permitting any source on port 80 (omitting the `from` block).

### ❌ FAQ 3: VM unable to reach cluster service IPs or route DNS
* **Symptom:** VM cannot resolve `.cluster.local` DNS names, or connection timed out when curling cluster pods.
* **Root Cause:** Incorrect kernel routing rules or lack of transparent DNS proxying on the host.
* **Solution:** 
  1. Ensure the transparent proxy on the VM is initialized with DNS redirection:
     ```bash
     /usr/local/bin/istio-start.sh --redirect-dns
     ```
  2. Extract the `coredns` binary from the Kuma/Istio bundle and install it to `/usr/local/bin`.
  3. Ensure static routes are set on the VM host for the cluster Pod and Service CIDRs.
