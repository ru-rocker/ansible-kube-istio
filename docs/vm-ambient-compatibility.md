# KB-1049: Hybrid Service Mesh Integration Guide (Istio Ambient + VM Sidecars)

* **Document Status:** `APPROVED` / `PRODUCTION-READY`
* **Target Audience:** Platform Engineering, DevOps, Cloud Infrastructure Team
* **Mesh Technology:** Istio Ambient Mode (ztunnel + waypoint) ↔ Istio Sidecar Proxy (Universal VM)
* **Underlying Network:** Private Subnet (VPC Layer-2 on `10.130.0.0/16`) + Public Internet fallback

---

## 1. Executive Summary

This page outlines the architecture, configuration instructions, and operational runbook for integrating standard Virtual Machine (VM) sidecar workloads with Kubernetes workloads running in **Istio Ambient Mode** (sidecarless). 

Because native Ambient Mode (using shared node proxies) does not extend to VMs, a **hybrid mesh data plane** must be maintained. The VM runs a traditional `istio-proxy` (Envoy) sidecar, while the cluster uses node-level `ztunnels` and namespace-scoped `waypoint` proxies.

---

## 2. Technical Architecture & Traffic Flow

### Inbound to VM (Cluster → VM Sidecar)

The inbound traffic flow architecture depends on whether the VM is accessed across a public/NAT boundary or via a direct internal flat network:

#### Scenario A: Public Internet or NAT Boundary (Plaintext HTTP/80 Fallback)
Because the VM is resolved outside the cluster's internal Pod IP range and across NAT boundaries, the Waypoint proxy connects to the VM's external IP via plaintext HTTP (due to ztunnel/waypoint being unable to form secure HBONE tunnels over public/untrusted networks).

```
[ Client Pod ] (Ambient Namespace)
      │ (mTLS / HBONE on Port 15008)
      ▼
[ ztunnel ] (Client Node)
      │ (mTLS / HBONE on Port 15008)
      ▼
[ Waypoint Proxy ] (Destination Namespace)
      │ (Plaintext HTTP/80 over Internet/NAT)
      ▼
[ VM Sidecar (Envoy) ] (PERMISSIVE mode)
      │ (Localhost loopback)
      ▼
[ Nginx Application ]
```

#### Scenario B: Private Internal / Flat Network (STRICT mTLS Best Practice)
When the VM and the cluster nodes communicate over a flat private network (e.g., direct routing via VPC peering or shared subnets), the Waypoint proxy connects directly to the VM's internal IP using secure **mTLS** encryption.

```
[ Client Pod ] (Ambient Namespace)
      │ (mTLS / HBONE on Port 15008)
      ▼
[ ztunnel ] (Client Node)
      │ (mTLS / HBONE on Port 15008)
      ▼
[ Waypoint Proxy ] (Destination Namespace)
      │ (mTLS/TCP on application port)
      ▼
[ VM Sidecar (Envoy) ] (STRICT mode)
      │ (Localhost loopback)
      ▼
[ Nginx Application ]
```

### Outbound from VM (VM Sidecar → Cluster Pod)
Using a private VPC subnet, the VM bypasses NAT boundaries and routes directly to the cluster's internal Pod/Service CIDR ranges via a worker node gateway.

```
[ VM Application (Nginx) ]
      │ (Localhost loopback redirection)
      ▼
[ VM Sidecar (Envoy) ]
      │ (mTLS / TCP Port 80 via Private Interface eth1)
      ▼
[ Worker Node Gateway ] (IP Forwarding / Flannel routing)
      │ (Redirected by local ztunnel/iptables)
      ▼
[ Waypoint Proxy ] (Namespace L7 mediation)
      │ (mTLS / HBONE Port 15008)
      ▼
[ Destination Pod ] (Ambient Namespace)
```

---

## 3. Data Plane Configuration Guide

Before starting, choose either **Option A (Direct Routing)** or **Option B (Private Gateway Routing)** below.

---

### Track A: Step-by-Step Setup for Option A (Direct Node Routing)
Use this track if you want direct point-to-point network communication to Pod/Service CIDRs and are okay adding static IP routes to your VM's OS routing table.

> [!TIP]
> **Internal Network / Flat Network Compatibility:** Track A establishes a direct, flat private routing path between the VM and the cluster nodes. Because there are no NAT boundaries separating them, you can secure inbound traffic using **Option B (STRICT mTLS)** in Steps 4 and 5.


#### 1. Configure Egress Routes on the VM
Add routing rules directing cluster networks through the private IP of a cluster node:
```bash
sudo ip route replace 10.244.0.0/16 via <WORKER_NODE_PRIVATE_IP> dev eth1
sudo ip route replace 10.96.0.0/12 via <WORKER_NODE_PRIVATE_IP> dev eth1
```

#### 2. Configure VM Bootstrap (`/var/lib/istio/envoy/cluster.env`)
Leave network empty (`""`) and enable HBONE support:
```ini
ISTIO_META_ENABLE_HBONE=true
ISTIO_META_NETWORK=
```

#### 3. Apply WorkloadGroup
Ensure `spec.template.network` is set to `""` in your WorkloadGroup:
```yaml
spec:
  template:
    network: "" # Direct connection to cluster network
```

---

### Track B: Step-by-Step Setup for Option B (Private Gateway Routing)
Use this track if you want **zero routing changes on the VM** (the VM will only talk to IPs in its local private subnet).

#### 1. Associate the Gateway with the Node Private IP
Patch the `cross-network-gateway-istio` service in `istio-system` namespace. Assign the private IP of your worker/gateway node to the `externalIPs` list:
```yaml
# Apply via: kubectl patch svc cross-network-gateway-istio -n istio-system --patch-file gateway-service-patch.yaml
spec:
  externalIPs:
  - 10.130.12.254 # Replace with your worker node's private IP
```

#### 2. Configure VM Bootstrap (`/var/lib/istio/envoy/cluster.env`)
Assign the VM to a distinct network named `vm-network` and enable HBONE support:
```ini
ISTIO_META_ENABLE_HBONE=true
ISTIO_META_NETWORK=vm-network
```

#### 3. Apply WorkloadGroup
Ensure `spec.template.network` is set to `vm-network` in your WorkloadGroup:
```yaml
spec:
  template:
    network: vm-network # Tells istiod to route via the cross-network gateway
```

---

### Common Setup Steps (Required for both Tracks)

#### 4. Restart Istio Sidecar Agent on the VM
```bash
sudo systemctl restart istio
```

### Step 3: WorkloadGroup and Namespace Setup
We must prevent `istiod` from treating the VM as an Ambient-managed node. The VM must be explicitly configured as a sidecar:

```yaml
# 1. Namespace Ambient Activation
apiVersion: v1
kind: Namespace
metadata:
  name: mesh-services
  labels:
    istio.io/dataplane-mode: ambient
    istio-injection: disabled

# 2. WorkloadGroup Sidecar Declaration
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
  name: vm-proxy-group
  namespace: mesh-services
spec:
  metadata:
    labels:
      app: vm-proxy
      istio.io/dataplane-mode: none  # Enforces Sidecar-only behavior
```

### Step 4: Configure Mutual TLS (mTLS) Policy

Depending on your network topology, select the appropriate configuration option:

#### Option A: Over Public / NAT Boundaries (Plaintext Fallback)
Because the inbound connection from the Waypoint Proxy to the VM sidecar is unencrypted plaintext, you must set `PERMISSIVE` mTLS for the VM's namespace to prevent the global `STRICT` policy from dropping the traffic:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: vm-namespace-permissive
  namespace: mesh-services
spec:
  mtls:
    mode: PERMISSIVE # Accepts both plaintext and mTLS
```
> [!IMPORTANT]
> Since the VM connects from a NAT environment, its workload labels are not fully resolved by `istiod`. The PeerAuthentication policy **must not** contain a `selector` block and must be applied namespace-wide.

#### Option B: Private Internal / Flat Network (STRICT mTLS Best Practice)
If the VM and cluster communicate directly over a flat internal private network, we can secure all traffic using **STRICT** mTLS:
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: vm-namespace-strict
  namespace: mesh-services
spec:
  mtls:
    mode: STRICT # Enforces mutual TLS for all incoming VM traffic
```

### Step 5: Update the Authorization Policy

Depending on your network topology:

#### Option A: Over Public / NAT Boundaries (Plaintext Fallback)
Because Waypoint-to-VM traffic is plaintext, it does not carry a SPIFFE identity. The destination AuthorizationPolicy must contain a secondary rule to allow unauthenticated traffic on the application port (e.g., port 80):
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: vm-proxy-policy
  namespace: mesh-services
spec:
  action: ALLOW
  rules:
  - from:                              # Rule 1: Authenticated mesh workloads (mTLS)
    - source:
        principals: ["cluster.local/ns/mesh-services/sa/waypoint"]
    to:
    - operation: { ports: ["80"] }
  - to:                                # Rule 2: Unauthenticated Waypoint proxy (Plaintext)
    - operation: { ports: ["80"] }
```

#### Option B: Private Internal / Flat Network (STRICT mTLS Best Practice)
When operating over a flat internal network, the Waypoint proxy's mTLS identity (`waypoint` ServiceAccount) is presented and validated. You should enforce strict access control and **remove the plaintext/unauthenticated bypass**:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: vm-proxy-policy
  namespace: mesh-services
spec:
  action: ALLOW
  rules:
  - from:                              # Only allow authenticated traffic from the Waypoint
    - source:
        principals: ["cluster.local/ns/mesh-services/sa/waypoint"]
    to:
    - operation: { ports: ["80"] }
```


---

## 4. Verification and Testing Runbook

To verify the integration, run these diagnostic commands from the control plane:

| Path | Verification Command | Expected Output |
|---|---|---|
| **kube-kube** | `kubectl exec -n meshz-services deploy/client-service -- curl -s -o /dev/null -w "%{http_code}" http://order-service.mesh-services.svc.cluster.local/` | `200` |
| **kube-vm** | `kubectl exec -n meshz-services deploy/client-service -- curl -s -o /dev/null -w "%{http_code}" http://vm-proxy.mesh-services.svc.cluster.local/` | `200` (Nginx Page) |
| **vm-kube** | `ansible proxy_servers -i inventory/hosts.ini -m shell -a "curl -s -o /dev/null -w '%{http_code}' http://order-service.mesh-services.svc.cluster.local/"` | `200` |

---

## 5. Troubleshooting & FAQ Matrix

| Symptom | Root Cause | Resolution |
|---|---|---|
| **`connection termination`** / HTTP empty reply | Global STRICT mTLS policy is blocking the plaintext connection sent from the Waypoint to the VM's public IP. | Apply a `PERMISSIVE` PeerAuthentication policy namespace-wide to the VM namespace. |
| **`RBAC: access denied`** (HTTP 403) | The AuthorizationPolicy enforces SPIFFE validation. Plaintext requests do not carry a certificate, and are blocked by default. | Add a secondary rule to the VM's `AuthorizationPolicy` allowing port 80 traffic with no `from` constraints. |
| **`503 Service Unavailable`** (VM → Cluster) | The VM has no static routing configuration to reach the cluster's internal Pod CIDR (`10.244.x.x`) or Service CIDR (`10.96.x.x`). | Configure static routes on the VM pointing cluster CIDRs to a cluster node's private interface IP as a gateway. |
| **VM dns resolution failing** | `systemd-resolved` or local dnsmasq configuration is not forwarding `.cluster.local` requests to the Envoy proxy. | Create a systemd-resolved override config block (`/etc/systemd/resolved.conf.d/istio.conf`) forwarding local queries to `127.0.0.1:15053` for `~cluster.local`. |

---

## 6. Real-World Production Risks & Challenges (The Logarithmic Curve)

Onboarding existing, high-scale VM infrastructure into a service mesh introduces non-linear complexity. When transitioning from 75% to 100% mesh coverage (representing the integration of VMs), platforms face severe production challenges:

### A. OS Heterogeneity & Maintenance Risks
* **The Reality:** Unlike Kubernetes where Node operating systems are abstracted from the container, VM sidecars run directly inside the host OS. A typical enterprise VM fleet runs a wide range of OS flavors and versions (e.g., RedHat/CentOS 7/8/9, Debian 10/11/12, SLES, Ubuntu LTS releases).
* **The Risk:** Developing, packaging, and maintaining Istio Sidecar installers/RPMs/DEBs across conflicting system libraries, glibc versions, and OpenSSL configurations introduces massive testing overhead and update risks.

### B. High Egress Routing Risk (`ip route` modification)
* **The Reality:** Direct VM-to-Kube routing (Option A) relies on altering the system's routing tables.
* **The Risk:** Modifying routing tables (`ip route`) in production is a highly intrusive action. An incorrect CIDR mapping or a configuration error can lead to:
  * **Routing Loops:** Completely freezing VM networking.
  * **Network Outages:** Cutting off administrator SSH access or external database replication streams.
  * **NAT Asymmetry:** Breaking return traffic routing to load balancers.

### C. DNS & Resolved Configuration Struggles
* **The Reality:** Traditional Linux distributions handle DNS resolution in disjointed ways.
* **The Risk:** Older distributions lack unified support for `systemd-resolved`. Mixing custom `/etc/resolv.conf` scripts, `dnsmasq`, and `bind` across a fleet of VMs frequently leads to:
  * DNS loops between local loops and Envoy's internal DNS capture listener (`15053`).
  * Intermittent lookup failures for cluster services (`*.cluster.local`), breaking application dependency calls.

### D. Network Isolation & Double-NAT Overhead
* **The Reality:** Production VMs are rarely on the same flat network as the Kubernetes cluster. They sit behind hardware firewalls, security groups, and double-NAT configurations.
* **The Risk:** Navigating these boundaries requires establishing custom East-West gateways, static IP mapping, and firewall exceptions for ports `15443` (cross-network mTLS) and `15012` (xDS bootstrap). A single firewall rule misconfiguration will immediately isolate the VM sidecar from the control plane, causing configuration staleness.

### E. Stateful Application Compatibility Gaps
* **The Reality:** Traditional VM workloads often run stateful runtimes (e.g. WildFly, JBoss, WebLogic, Oracle Database) with long-lived connection pools.
* **The Risk:** Envoy sidecars aggressively prune idle TCP connections and enforce strict HTTP/2 keep-alive requirements. Runtimes with static JDBC/JMS connection pools are highly sensitive to these TCP terminations, leading to sudden, hard-to-debug database connection drops and application runtime errors.

