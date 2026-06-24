# Ansible Kubernetes & Istio Deployment Guardrails

This project-specific instruction file outlines the coding standards, behavioral constraints, and security guardrails that must be followed when developing or executing Ansible playbooks for installing Kubernetes and Istio.

---

## 1. Safety & Pre-flight Guardrails (System Validation)

Before initiating any Kubernetes or Istio installation tasks, playbooks must validate target host conditions. The following checks are mandatory:

- **Operating System**: Only standard LTS distributions are supported (Ubuntu 22.04+, Debian 11+, or RHEL 8+).
- **Hardware Resources**:
  - Minimum **1 CPU Core** per node.
  - Minimum **1 GB of RAM** per node.
  - Minimum **10 GB of free disk space** under `/var/lib` and `/`.
- **Memory Swap**:
  - Swap must be disabled on all Kubernetes nodes (`kubelet` requirement).
  - Guardrail task must fail if swap is active unless explicitly configured to disable it automatically.
- **Port Availability**:
  - API Server port (`6443`) must not be in use prior to initialization.
  - Istio default ports (`15017`, `15021`, etc.) must be free on ingress nodes.
- **Kernel Modules & Network Parameters**:
  - Kernel modules `br_netfilter` and `overlay` must be loaded.
  - Sysctl parameters `net.bridge.bridge-nf-call-iptables` and `net.ipv4.ip_forward` must be set to `1`.

---

## 2. Playbook Coding Standards & Best Practices

To ensure idempotency, readability, and reliability of the playbooks:

- **Idempotency First**: Avoid using the `shell` or `command` modules unless no Ansible module exists. If `shell` is used, always provide `creates`, `removes`, or use `changed_when` to define when the task actually changes state.
- **Module Prefixes**: Use fully qualified collection names (FQCN) for all tasks (e.g., `ansible.builtin.apt` instead of `apt`, `kubernetes.core.k8s` instead of `k8s`).
- **No Hardcoded Secrets**: Credentials, tokens, and certificates must be handled securely (e.g., via Ansible Vault or environment variables). Never commit plain-text tokens or keys.
- **Tags Usage**: Annotate playbooks with logical tags (`preflight`, `kubernetes`, `istio`, `security`) to allow running specific phases of the setup.
- **Error Handling**: Use `failed_when` and `any_errors_fatal: true` where appropriate (especially during pre-flight checks and control plane initialization).

---

## 3. Kubernetes & Istio Security Policy Guardrails

Post-installation, the environment must be secured using these declarative guardrails:

- **Namespace Isolation**:
  - Automate labeling target namespaces with `istio-injection=enabled` to ensure sidecar proxies are injected.
- **Strict Mutual TLS (mTLS)**:
  - Apply a global or namespace-level `PeerAuthentication` policy setting `mode: STRICT` to enforce encrypted communication within the service mesh.
- **Default-Deny Authorization Policy**:
  - Deploy a default-deny `AuthorizationPolicy` in the root namespace (`istio-system`) or target namespaces to enforce a "zero-trust" network policy (fail-closed posture). Only explicitly defined traffic paths should be allowed.
- **Least Privilege RBAC**:
  - Limit Kubernetes service accounts used by Istio components to the bare minimum RBAC roles.
- **Istio CNI Usage**:
  - Use the Istio CNI plugin instead of `istio-init` containers where possible to eliminate the requirement for pods to run with `NET_ADMIN` privileges.

---

## 4. Playbook Architecture

The workspace should be structured as follows:

```text
ansible-kube-istio/
├── GEMINI.md                   # This guardrails document
├── inventory/
│   └── hosts.ini               # Node inventory definitions
├── group_vars/
│   └── all.yml                 # Configuration variables (versions, thresholds)
├── site.yml                    # Main entrypoint playbook
└── roles/
    ├── preflight/              # Pre-checks role
    ├── kubernetes/             # Kubeadm setup role
    ├── istio/                  # Istioctl/Helm installation role
    └── security_guardrails/    # Zero-trust and mTLS policy role
```
