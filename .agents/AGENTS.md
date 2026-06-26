# Project-Specific Rules: Hybrid Kubernetes-VM Mesh Integration

This file contains coding and configuration guardrails for deploying and troubleshooting Kuma (Universal Mode) and Istio (Ambient & Sidecar) service meshes with VM workloads.

---

## 1. Kuma Hybrid Deployment Guardrails

When configuring Kuma Control Plane for hybrid clusters and Universal VMs:

* **Control Plane Service Ports:** Never override the xDS bootstrap port to a NodePort without exposing both ports in the `kuma-control-plane` Service. Ensure port `5678` (default bootstrap) and the advertised NodePort (e.g. `30578`) are both declared.
* **Proxy Authentication:** In hybrid environments, do not rely purely on Kubernetes service accounts. Set `KUMA_DP_SERVER_AUTHN_DP_PROXY_TYPE: "none"` (or configure multiple explicit authenticators) to avoid rejecting Universal VM `dpToken` credentials.
* **VM DNS Resolution:** Ensure transparent proxy is installed with `--redirect-dns` to intercept port 53 traffic, and ensure the `coredns` binary is extracted from the Kuma bundle and placed in `/usr/local/bin`.
* **VM Network Routing:** Universal VMs require static routing rules to reach Kubernetes pod CIDRs when bypassing NAT gateways to avoid connection resets.

---

## 2. Istio Hybrid & Ambient Guardrails

When configuring VM workloads with Istio:

* **Ambient Mesh Limitations:** Do not deploy VMs as native Ambient Mode workloads. Ztunnel does not route to `WorkloadEntry` destinations.
* **Waypoint Integration:** To allow sidecar-equipped VMs to talk to Ambient-enabled Kubernetes pods, you must deploy Namespace Waypoints to translate standard mTLS into HBONE encapsulation.
* **Authorization Policies:** When using Waypoints for VM traffic, do not restrict policies with strict workload selectors on the VM targets. Gateways and waypoints rewrite/remove workload labels across NAT boundaries; use namespace-wide policies instead.
