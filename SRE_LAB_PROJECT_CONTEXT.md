# Project Context — SRE Lab Infrastructure

## Overview

```
PROJECT:     sre-lab
REPO:        github.com/your-org/sre-lab
DESCRIPTION: Production-grade SRE observability lab running Talos Linux,
             Kubernetes, and full monitoring stack. Deployable locally
             (VirtualBox/Vagrant) or any cloud provider.
TEAM:        Platform/SRE Team
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SRE LAB                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     TALOS LINUX NODES                                 │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                      KUBERNETES                                 │  │  │
│  │  │  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │  │  │
│  │  │  │ Cilium Agent     │  │ Kube-state-      │  │ Node          │  │  │  │
│  │  │  │ + Hubble         │  │ metrics          │  │ Exporter      │  │  │  │
│  │  │  │ (eBPF Metrics)   │  │                  │  │               │  │  │  │
│  │  │  └────────┬─────────┘  └────────┬─────────┘  └───────┬───────┘  │  │  │
│  │  │           │ Metrics endpoint    │                    │          │  │  │
│  │  └───────────┼─────────────────────┼────────────────────┼──────────┘  │  │
│  └──────────────┼─────────────────────┼────────────────────┼─────────────┘  │
│                 │                     │                    │                │
│                 ▼                     ▼                    ▼                │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    PROMETHEUS OPERATOR                              │    │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                   │    │
│  │  │     Prometheus      │  │    Alertmanager     │                   │    │
│  │  └─────────────────────┘  └─────────────────────┘                   │    │
│  │  ┌─────────────────────────────────────────────────────────────┐    │    │
│  │  │                   ServiceMonitors                           │    │    │
│  │  └─────────────────────────────────────────────────────────────┘    │    │
│  └──────────────────────────────────┬──────────────────────────────────┘    │
│                                     │                                       │
│        ┌────────────────────────────┼────────────────────────────┐          │
│        │                            │                            │          │
│        ▼                            ▼                            ▼          │
│  ┌───────────┐              ┌───────────────┐              ┌──────────┐     │
│  │   Loki    │◄─────────────│   Grafana     │─────────────►│ Ingress  │     │
│  │  (Logs)   │   Query      │ (Dashboards)  │   Expose     │          │     │
│  └───────────┘              └───────────────┘              └──────────┘     │
│        ▲                                                                    │
│        │ Promtail                                                           │
│        │ (Log collection)                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Deployment Topology

### Local Development (VirtualBox + Vagrant)

| Node | Role | vCPU | RAM | Disk | IP |
|------|------|------|-----|------|-----|
| talos-cp-01 | Control Plane | 2 | 4GB | 40GB | 192.168.56.10 |
| talos-cp-02 | Control Plane | 2 | 4GB | 40GB | 192.168.56.11 |
| talos-cp-03 | Control Plane | 2 | 4GB | 40GB | 192.168.56.12 |
| talos-worker-01 | Worker | 2 | 4GB | 50GB | 192.168.56.20 |
| talos-worker-02 | Worker | 2 | 4GB | 50GB | 192.168.56.21 |

### Cloud Deployment Options

| Provider | Control Plane | Workers | Instance Type | Notes |
|----------|---------------|---------|---------------|-------|
| AWS | 3x EC2 | 2-5x EC2 | t3.medium+ | EKS not used (bare Talos) |
| GCP | 3x GCE | 2-5x GCE | e2-medium+ | GKE not used (bare Talos) |
| Azure | 3x VM | 2-5x VM | Standard_B2s+ | AKS not used (bare Talos) |
| Hetzner | 3x CX21 | 2-5x CX21 | CX21+ | Cost-effective option |
| Proxmox | 3x VM | 2-5x VM | Custom | On-premise/homelab |

### Environment Matrix

| Environment | Platform | Nodes | Purpose |
|-------------|----------|-------|---------|
| local | VirtualBox/Vagrant | 3 CP + 2 Worker | Development, learning |
| dev | Cloud (any) | 1 CP + 1 Worker | Minimal testing |
| staging | Cloud (any) | 3 CP + 2 Worker | Pre-production |
| prod | Cloud (any) | 3 CP + 3+ Worker | Production |

---

## Tech Stack

### Infrastructure Layer

```
Base OS:           Talos Linux v1.6+ (immutable, secure, minimal)
Provisioning:      
  - Local: Vagrant + VirtualBox
  - Cloud: Terraform + talos-provider / cloud-init
Talos Config:      talosctl + machine configs (controlplane.yaml, worker.yaml)
```

### Kubernetes Layer

```
Distribution:      Talos-managed Kubernetes v1.29+
CNI:               Cilium v1.15+ (with Hubble for observability)
Ingress:           Ingress-NGINX or Cilium Ingress
Storage:           Local-path-provisioner (local) / CSI driver (cloud)
Secrets:           Sealed Secrets or External Secrets Operator
```

### Observability Stack

```
Metrics:
  - Prometheus Operator (kube-prometheus-stack Helm chart)
  - Prometheus (metrics collection & storage)
  - Alertmanager (alerting)
  - ServiceMonitors (auto-discovery)
  - kube-state-metrics (K8s object metrics)
  - node-exporter (host metrics)

Logging:
  - Loki (log aggregation)
  - Promtail (log collection from nodes/pods)

Visualization:
  - Grafana (dashboards, unified view)

Network Observability:
  - Cilium Hubble (eBPF-based network metrics)
  - Hubble UI (network flow visualization)
```

### CI/CD & GitOps

```
Pipeline:          GitHub Actions
GitOps:            Flux CD or ArgoCD (optional)
Image Registry:    GHCR / Docker Hub
IaC:               Terraform (cloud), Vagrant (local)
Config Mgmt:       Helm + Kustomize
```

---

## Repository Structure

```
sre-lab/
├── infrastructure/
│   ├── vagrant/                    # Local VirtualBox deployment
│   │   ├── Vagrantfile
│   │   ├── scripts/
│   │   │   ├── install-talos.sh
│   │   │   └── bootstrap-cluster.sh
│   │   └── config/
│   │       └── cluster.yaml
│   │
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── talos-cluster/      # Generic Talos cluster module
│   │   │   ├── aws/                # AWS-specific resources
│   │   │   ├── gcp/                # GCP-specific resources
│   │   │   ├── azure/              # Azure-specific resources
│   │   │   └── hetzner/            # Hetzner-specific resources
│   │   │
│   │   └── environments/
│   │       ├── local/              # Terraform for local (libvirt alternative)
│   │       ├── dev/
│   │       ├── staging/
│   │       └── prod/
│   │
│   └── talos/                      # Talos machine configurations
│       ├── controlplane.yaml.tpl
│       ├── worker.yaml.tpl
│       ├── patches/
│       │   ├── cilium.yaml
│       │   └── metrics.yaml
│       └── talconfig.yaml          # talhelper configuration
│
├── kubernetes/
│   ├── bootstrap/                  # Initial cluster setup
│   │   ├── namespaces.yaml
│   │   └── storage-class.yaml
│   │
│   ├── infrastructure/             # Core infrastructure components
│   │   ├── cilium/
│   │   │   ├── kustomization.yaml
│   │   │   ├── helm-release.yaml
│   │   │   └── values.yaml
│   │   │
│   │   ├── ingress-nginx/
│   │   │   ├── kustomization.yaml
│   │   │   └── values.yaml
│   │   │
│   │   └── cert-manager/
│   │       └── ...
│   │
│   ├── observability/              # Monitoring stack
│   │   ├── prometheus-stack/
│   │   │   ├── kustomization.yaml
│   │   │   ├── helm-release.yaml
│   │   │   ├── values.yaml
│   │   │   ├── values-local.yaml
│   │   │   ├── values-cloud.yaml
│   │   │   └── dashboards/
│   │   │       ├── cluster-overview.json
│   │   │       └── cilium-hubble.json
│   │   │
│   │   ├── loki-stack/
│   │   │   ├── kustomization.yaml
│   │   │   ├── helm-release.yaml
│   │   │   └── values.yaml
│   │   │
│   │   └── hubble/
│   │       └── values.yaml
│   │
│   └── apps/                       # Sample applications (optional)
│       └── demo-app/
│
├── scripts/
│   ├── local-up.sh                 # One-command local deployment
│   ├── local-down.sh               # Destroy local cluster
│   ├── cloud-deploy.sh             # Cloud deployment wrapper
│   ├── get-kubeconfig.sh           # Retrieve kubeconfig from Talos
│   └── validate.sh                 # Validate all configs
│
├── docs/
│   ├── PROJECT_CONTEXT.md
│   ├── architecture.md
│   ├── local-setup.md
│   ├── cloud-deployment.md
│   └── runbooks/
│       ├── troubleshooting.md
│       └── disaster-recovery.md
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── deploy-cloud.yml
│       └── validate.yml
│
├── Taskfile.yml                    # Task runner
├── .cursorrules
└── README.md
```

---

## Naming Conventions

### Infrastructure

```
Local VMs:    talos-{role}-{index}
              talos-cp-01, talos-worker-01

Cloud:        {env}-sre-{role}-{index}
              dev-sre-cp-01, prod-sre-worker-03

Networks:     sre-lab-{env}-{purpose}
              sre-lab-dev-vpc, sre-lab-prod-node-subnet
```

### Kubernetes

```
Namespaces:   {category}
              monitoring, logging, ingress, apps

Resources:    {component}-{descriptor}
              prometheus-main, grafana-dashboards, loki-stack

Helm:         {chart-name}
              kube-prometheus-stack, loki, ingress-nginx
```

---

## Key Components & Versions

| Component | Version | Helm Chart | Notes |
|-----------|---------|------------|-------|
| Talos Linux | 1.6.x | N/A | Immutable OS |
| Kubernetes | 1.29.x | N/A | Via Talos |
| Cilium | 1.15.x | cilium/cilium | CNI + eBPF |
| Hubble | 1.15.x | (part of Cilium) | Network observability |
| Prometheus Operator | 0.72.x | prometheus-community/kube-prometheus-stack | |
| Prometheus | 2.50.x | (part of stack) | |
| Alertmanager | 0.27.x | (part of stack) | |
| Grafana | 10.x | (part of stack) | |
| Loki | 2.9.x | grafana/loki-stack | |
| Promtail | 2.9.x | (part of loki-stack) | |
| kube-state-metrics | 2.10.x | (part of prometheus-stack) | |
| node-exporter | 1.7.x | (part of prometheus-stack) | |

---

## Access & Endpoints

### Local Development

```
Grafana:       http://grafana.localhost:8080 (or NodePort 30300)
Prometheus:    http://prometheus.localhost:8080 (or NodePort 30900)
Alertmanager:  http://alertmanager.localhost:8080 (or NodePort 30903)
Hubble UI:     http://hubble.localhost:8080 (or port-forward)

Kubernetes API: https://192.168.56.10:6443
Talos API:      https://192.168.56.10:50000
```

### Cloud

```
Grafana:       https://grafana.{domain}
Prometheus:    https://prometheus.{domain} (internal only)
Alertmanager:  https://alertmanager.{domain} (internal only)
Hubble UI:     https://hubble.{domain}

LoadBalancer:  Provisioned by cloud provider
```

---

## Prerequisites

### Local Development

```
Required:
  - VirtualBox >= 7.0
  - Vagrant >= 2.4
  - talosctl >= 1.6.0
  - kubectl >= 1.29
  - helm >= 3.14

Optional:
  - task (taskfile runner)
  - k9s (terminal UI)
  - cilium-cli

Host Resources:
  - Minimum: 16GB RAM, 8 CPU cores, 100GB disk
  - Recommended: 32GB RAM, 12 CPU cores, 200GB SSD
```

### Cloud Deployment

```
Required:
  - Terraform >= 1.6
  - Cloud provider CLI (aws/gcloud/az)
  - talosctl >= 1.6.0
  - kubectl >= 1.29
  - helm >= 3.14

Credentials:
  - Cloud provider credentials configured
  - DNS zone access (for ingress)
```

---

## Deployment Workflow

### Local (One-Command)

```bash
# Start everything
./scripts/local-up.sh

# Or with task
task local:up

# Access Grafana
open http://grafana.localhost:8080
```

### Cloud

```bash
# Set target environment
export ENV=dev
export CLOUD=aws  # or gcp, azure, hetzner

# Deploy infrastructure
task cloud:plan
task cloud:apply

# Bootstrap Kubernetes components
task k8s:bootstrap
task k8s:observability

# Verify
task verify
```

---

## Known Constraints

### Talos Linux Limitations

- [x] No SSH access (by design) — use `talosctl`
- [x] No package manager — immutable OS
- [x] Requires separate machine for `talosctl` operations
- [x] CNI must be deployed post-bootstrap (Cilium)

### Local Development

- [x] Resource intensive (16GB+ RAM recommended)
- [x] VirtualBox networking can be finicky
- [x] No cloud LoadBalancer — use NodePort or MetalLB

### Observability Stack

- [x] Prometheus storage is ephemeral by default — configure PV for persistence
- [x] Loki requires storage backend for production (S3/GCS/MinIO)
- [x] Grafana dashboards need provisioning via ConfigMaps

---

## Contacts

| Role | Contact |
|------|---------|
| Project Owner | @your-name |
| Documentation | `docs/` folder |
| Issues | GitHub Issues |

---

*Last updated: 2024-01-20*
