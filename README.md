# SRE Lab — Talos Linux Observability Platform

A production-grade SRE observability lab running Talos Linux, Kubernetes, Cilium, Prometheus, Loki, and Grafana. Deployable locally (VirtualBox/Vagrant) or on any major cloud provider.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SRE LAB                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     TALOS LINUX NODES                                 │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                      KUBERNETES                                 │  │  │
│  │  │                                                                 │  │  │
│  │  │  ┌──────────────────┐  ┌─────────────────┐  ┌────────────────┐  │  │  │
│  │  │  │ Cilium + Hubble  │  │ kube-state-     │  │ node-exporter  │  │  │  │
│  │  │  │ (eBPF Metrics)   │  │ metrics         │  │                │  │  │  │
│  │  │  └────────┬─────────┘  └────────┬────────┘  └───────┬────────┘  │  │  │
│  │  │           └──────────────┬──────┴──────────────────┬┘           │  │  │
│  │  └──────────────────────────┼─────────────────────────┼────────────┘  │  │
│  └─────────────────────────────┼─────────────────────────┼────────────────┘  │
│                                ▼                         ▼                   │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    PROMETHEUS OPERATOR                              │    │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                   │    │
│  │  │     Prometheus      │  │    Alertmanager     │                   │    │
│  │  └─────────────────────┘  └─────────────────────┘                   │    │
│  │  ┌─────────────────────────────────────────────────────────────────┐│    │
│  │  │                   ServiceMonitors                               ││    │
│  │  └─────────────────────────────────────────────────────────────────┘│    │
│  └─────────────────────────────┬───────────────────────────────────────┘    │
│                                │                                            │
│        ┌───────────────────────┼────────────────────────┐                   │
│        ▼                       ▼                        ▼                   │
│  ┌───────────┐          ┌───────────────┐         ┌──────────┐              │
│  │   Loki    │◄─────────│   Grafana     │────────►│ Ingress  │              │
│  │  (Logs)   │          │ (Dashboards)  │         │          │              │
│  └───────────┘          └───────────────┘         └──────────┘              │
│        ▲                                                                    │
│        │ Promtail                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Features

| Feature | Description |
|---------|-------------|
| **Talos Linux** | Immutable, secure, minimal OS designed for Kubernetes |
| **Cilium CNI** | eBPF-based networking with kube-proxy replacement |
| **Hubble** | Network flow observability and visualization |
| **Prometheus** | Metrics collection, alerting rules, ServiceMonitors |
| **Grafana** | Unified dashboards for metrics and logs |
| **Loki** | Log aggregation with Prometheus-like labels |
| **Multi-Environment** | Same stack works locally and on any cloud |

---

## Quick Start

### Prerequisites

```bash
# Required tools
brew install vagrant virtualbox talosctl kubectl helm task

# Optional but recommended
brew install cilium-cli hubble k9s
```

**Host requirements (local):** 16GB RAM minimum, 32GB recommended

### One-Command Local Deployment

```bash
# Clone the repository
git clone https://github.com/your-org/sre-lab.git
cd sre-lab

# Check prerequisites
task prereqs

# 🚀 Start everything (VMs, Talos, K8s, observability)
task local:up

# Access Grafana
task pf:grafana
# Then open http://localhost:3000
# Get password: task get:grafana-password
```

### What `task local:up` Does

```
1. Provisions 5 VirtualBox VMs with Talos Linux
   ├── 3 control plane nodes (192.168.56.10-12)
   └── 2 worker nodes (192.168.56.20-21)

2. Generates Talos machine configurations
   └── With patches for Cilium (no default CNI/kube-proxy)

3. Bootstraps Kubernetes cluster
   └── HA control plane with etcd

4. Deploys Cilium CNI + Hubble
   └── Nodes transition to Ready state

5. Deploys observability stack
   ├── Prometheus Operator
   ├── Prometheus + Alertmanager
   ├── Grafana with dashboards
   ├── Loki + Promtail
   └── ServiceMonitors for everything
```

---

## Project Structure

```
sre-lab/
├── infrastructure/
│   ├── vagrant/                    # Local VirtualBox setup
│   │   ├── Vagrantfile
│   │   └── scripts/
│   ├── terraform/                  # Cloud deployments
│   │   ├── modules/
│   │   │   ├── talos-cluster/      # Generic Talos module
│   │   │   ├── aws/
│   │   │   ├── gcp/
│   │   │   ├── azure/
│   │   │   └── hetzner/
│   │   └── environments/
│   │       ├── aws-dev/
│   │       ├── gcp-prod/
│   │       └── ...
│   └── talos/                      # Talos machine configs
│       ├── patches/
│       │   ├── cilium.yaml         # Disable default CNI
│       │   └── metrics.yaml        # Enable metrics endpoints
│       └── talconfig.yaml
│
├── kubernetes/
│   ├── infrastructure/
│   │   ├── cilium/                 # CNI + Hubble
│   │   └── ingress-nginx/
│   └── observability/
│       ├── prometheus-stack/       # Prometheus, Grafana, Alertmanager
│       └── loki-stack/             # Loki, Promtail
│
├── scripts/
│   ├── local-up.sh
│   └── local-down.sh
│
├── docs/
│   ├── PROJECT_CONTEXT.md          # Detailed project context
│   └── tasks/                      # Task definitions
│
├── Taskfile.yml                    # Task runner
├── .cursorrules                    # AI assistant rules
└── README.md
```

---

## Deployment Targets

### Local (VirtualBox + Vagrant)

| Node | Role | IP | Resources |
|------|------|-----|-----------|
| talos-cp-01 | Control Plane | 192.168.56.10 | 2 vCPU, 4GB RAM |
| talos-cp-02 | Control Plane | 192.168.56.11 | 2 vCPU, 4GB RAM |
| talos-cp-03 | Control Plane | 192.168.56.12 | 2 vCPU, 4GB RAM |
| talos-worker-01 | Worker | 192.168.56.20 | 2 vCPU, 4GB RAM |
| talos-worker-02 | Worker | 192.168.56.21 | 2 vCPU, 4GB RAM |

### Cloud (Terraform)

| Provider | Module | Est. Monthly Cost (Dev) |
|----------|--------|-------------------------|
| AWS | `terraform/environments/aws-dev/` | ~$30-40 |
| GCP | `terraform/environments/gcp-dev/` | ~$25-35 |
| Azure | `terraform/environments/azure-dev/` | ~$30-40 |
| Hetzner | `terraform/environments/hetzner-dev/` | ~$10-15 |

**Cloud deployment:**
```bash
export CLOUD=aws ENV=dev
task cloud:plan
task cloud:apply
task cloud:bootstrap
```

---

## Available Commands

```bash
task --list                   # Show all commands

# Local Environment
task local:up                 # 🚀 Full local deployment
task local:down               # 🗑️  Destroy local environment
task local:status             # Check cluster status
task local:info               # Show access information

# Cloud Environment
task cloud:plan CLOUD=aws ENV=dev
task cloud:apply CLOUD=aws ENV=dev
task cloud:destroy CLOUD=aws ENV=dev

# Deployments
task deploy:prometheus        # Deploy Prometheus stack
task deploy:loki              # Deploy Loki stack
task deploy:hubble-ui         # Enable Hubble UI

# Port Forwards
task pf:grafana               # localhost:3000
task pf:prometheus            # localhost:9090
task pf:alertmanager          # localhost:9093
task pf:hubble                # localhost:12000

# Validation
task validate                 # Run all checks
task validate:talos           # Check Talos health
task validate:cilium          # Check Cilium status

# Utilities
task shell                    # Open k9s
task get:grafana-password     # Get Grafana admin password
task logs:talos               # Stream Talos logs
task logs:cilium              # Stream Cilium logs
```

---

## Accessing Services

### Local Environment

| Service | Access Method |
|---------|---------------|
| Grafana | `task pf:grafana` → http://localhost:3000 |
| Prometheus | `task pf:prometheus` → http://localhost:9090 |
| Alertmanager | `task pf:alertmanager` → http://localhost:9093 |
| Hubble UI | `task pf:hubble` → http://localhost:12000 |
| Kubernetes API | `kubectl` (auto-configured) |
| Talos API | `talosctl` (auto-configured) |

### Grafana Credentials

```bash
# Username: admin
# Password:
task get:grafana-password
```

---

## Understanding Talos Linux

Talos Linux is different from traditional Linux distributions:

| Traditional Linux | Talos Linux |
|-------------------|-------------|
| SSH access | **No SSH** — use `talosctl` |
| Package manager (apt/yum) | **No packages** — immutable OS |
| Edit config files | **Machine configs** — YAML applied via API |
| Multiple services | **Kubernetes only** — single purpose |
| Manual updates | **Atomic upgrades** — full image replacement |

### Key Talos Commands

```bash
# Health check
talosctl health

# Get cluster info
talosctl cluster show

# Stream logs
talosctl logs -f kubelet
talosctl logs -f containerd

# Interactive dashboard
talosctl dashboard

# Get kubeconfig
talosctl kubeconfig ~/.kube/config

# Apply config changes
talosctl apply-config --nodes 192.168.56.10 --file controlplane.yaml
```

---

## Observability Stack Details

### Metrics Flow

```
Node Exporter (host metrics) ─────────┐
kube-state-metrics (K8s objects) ─────┼──► Prometheus ──► Grafana
Cilium Agent (eBPF network) ──────────┤
Hubble (flow metrics) ────────────────┤
kubelet (container metrics) ──────────┘
```

### Logging Flow

```
Container stdout/stderr ──┐
                          ├──► Promtail ──► Loki ──► Grafana
System logs (Talos) ──────┘
```

### Pre-configured Dashboards

- Kubernetes Cluster Overview
- Node Exporter Full
- Cilium Agent & Hubble
- Kubernetes Pods
- etcd Dashboard

---

## Troubleshooting

### Nodes Stuck in NotReady

```bash
# Usually means Cilium not deployed yet
kubectl get pods -n kube-system -l k8s-app=cilium
cilium status

# If Cilium pods are crashing, check logs
kubectl logs -n kube-system -l k8s-app=cilium
```

### Talos Bootstrap Fails

```bash
# Check Talos is reachable
talosctl health --nodes 192.168.56.10

# Check machine config was applied
talosctl get machineconfig --nodes 192.168.56.10

# View Talos logs
talosctl logs --nodes 192.168.56.10 -f
```

### VirtualBox Network Issues

```bash
# Verify host-only network exists
VBoxManage list hostonlyifs

# Recreate if needed
VBoxManage hostonlyif create
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1
```

### Prometheus Not Scraping Targets

```bash
# Check ServiceMonitors
kubectl get servicemonitors -A

# Verify target in Prometheus UI
task pf:prometheus
# Navigate to Status > Targets
```

---

## Development Workflow

### Minimal Local Cluster (Faster Iteration)

```bash
# Start only 1 CP + 1 Worker (saves resources)
task dev:minimal

# Full cluster later
task local:up
```

### Adding New Components

1. Create Helm values in `kubernetes/<category>/<component>/`
2. Add `values.yaml` (base) + `values-local.yaml` + `values-cloud.yaml`
3. Create deploy task in `Taskfile.yml`
4. Document in this README

### Modifying Talos Configuration

1. Edit patch files in `infrastructure/talos/patches/`
2. Regenerate configs: `task local:gen-config`
3. Apply to nodes: `task local:apply-config`
4. For major changes, rebuild cluster: `task local:down && task local:up`

---

## Task Templates

This project uses structured task templates for AI-assisted development. See `docs/tasks/` for examples:

- `TASK_01_local_vagrant_setup.md` — Vagrant/VirtualBox provisioning
- `TASK_02_cilium_hubble.md` — CNI deployment
- `TASK_03_prometheus_stack.md` — Prometheus Operator
- `TASK_04_loki_logging.md` — Log aggregation
- `TASK_05_cloud_terraform.md` — Multi-cloud Terraform

---

## Contributing

1. Use task templates for new features
2. Test on local environment first
3. Ensure works on both local and cloud
4. Update documentation

---

## License

MIT

---

## Credits

- [Talos Linux](https://www.talos.dev/) by Sidero Labs
- [Cilium](https://cilium.io/) by Isovalent
- [Prometheus](https://prometheus.io/) by CNCF
- [Grafana](https://grafana.com/)
- [Loki](https://grafana.com/oss/loki/)
- Architecture diagram inspired by Anselem Okeke
