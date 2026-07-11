# Sample: Private Gateway Routing (Option B)

This directory contains the sample configurations required to establish connectivity between the VM and the cluster using **Private Gateway Routing**, removing the need to add any static IP routing rules (`ip route`) on the VM.

## How to Apply

### 1. Patch the Gateway Service
Assign the private IP of a cluster node (e.g. `10.130.12.254` on the shared private subnet) to the `externalIPs` list of the `cross-network-gateway-istio` service:
```bash
# Update the IP in gateway-service-patch.yaml with your worker/gateway node private IP, then apply:
kubectl patch service cross-network-gateway-istio -n istio-system --patch-file gateway-service-patch.yaml
```

### 2. Apply the WorkloadGroup Configuration
Apply the WorkloadGroup containing the explicit `vm-network` parameter:
```bash
kubectl apply -f vm-workloadgroup-option-b.yaml
```

### 3. Update the VM Bootstrap Configuration
In `/var/lib/istio/envoy/cluster.env` on the VM, configure the VM to participate in `vm-network`:
```ini
ISTIO_META_NETWORK=vm-network
```
Restart the Istio agent on the VM to reload config:
```bash
sudo systemctl restart istio
```
