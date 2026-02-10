# Task 01: Local Infrastructure — Vagrant + VirtualBox + Talos

## Objective

```
GOAL:         Create Vagrant configuration to provision Talos Linux VMs 
              locally using VirtualBox for SRE Lab development
SCOPE:        infrastructure/vagrant/
              scripts/local-up.sh, scripts/local-down.sh
OUT OF SCOPE: Kubernetes components, observability stack (separate tasks)
```

---

## Context

```
WHY:          Enable local development and learning environment that mirrors
              production Talos+K8s setup without cloud costs
TICKET:       SRE-001
URGENCY:      High (foundation for all other tasks)
DEADLINE:     Week 1
```

### Background

Talos Linux is an immutable, minimal OS designed for Kubernetes. Unlike traditional VMs, Talos:
- Has no SSH — managed via `talosctl` API
- Boots directly into Kubernetes-ready state
- Requires machine configs (controlplane.yaml, worker.yaml)
- CNI must be deployed separately post-bootstrap

### Blocked By

- [ ] None — this is the foundation task

---

## Acceptance Criteria

- [ ] Vagrantfile provisions 3 control plane + 2 worker Talos VMs
- [ ] VMs boot Talos Linux from official ISO
- [ ] Network configured: host-only network (192.168.56.0/24)
- [ ] Each VM has correct resources (CPU, RAM, disk per spec)
- [ ] `talosctl` can reach all nodes after boot
- [ ] Machine configs generated via `talosctl gen config`
- [ ] Control plane bootstraps successfully
- [ ] `kubectl get nodes` shows all nodes (NotReady until CNI)
- [ ] `local-up.sh` script handles full provisioning
- [ ] `local-down.sh` script destroys all resources cleanly
- [ ] Documentation covers prerequisites and usage

### Definition of Done

- [ ] All VMs boot and join cluster
- [ ] README with clear instructions
- [ ] Tested on macOS and Linux hosts

---

## Constraints

```
MUST:
  - Use Talos Linux v1.6+ official ISO/image
  - Use VirtualBox provider (not VMware/libvirt)
  - Control plane nodes: 2 vCPU, 4GB RAM, 40GB disk
  - Worker nodes: 2 vCPU, 4GB RAM, 50GB disk
  - Host-only network for inter-node communication
  - NAT network for internet access (ISO download, image pulls)
  - Generate Talos configs with cluster name "sre-lab-local"
  - Store secrets (talosconfig, kubeconfig) in .secrets/ (gitignored)

MUST NOT:
  - Hard-code IPs in multiple places (use variables)
  - Commit secrets or talosconfig to git
  - Use deprecated Talos config formats
  - Require manual steps after `local-up.sh`

PREFER:
  - Use talhelper for config generation if simpler
  - Parallel VM provisioning where possible
  - Idempotent scripts (safe to re-run)
```

---

## Technical Details

### Current State

```
No local infrastructure exists. Starting fresh.
```

### Desired State

```
VirtualBox VMs:
├── talos-cp-01    192.168.56.10  (control plane, leader)
├── talos-cp-02    192.168.56.11  (control plane)
├── talos-cp-03    192.168.56.12  (control plane)
├── talos-worker-01 192.168.56.20  (worker)
└── talos-worker-02 192.168.56.21  (worker)

Networks:
├── vboxnet0 (host-only): 192.168.56.0/24  — cluster network
└── NAT: DHCP                              — internet access

Generated Files:
├── .secrets/
│   ├── talosconfig           # talosctl client config
│   ├── controlplane.yaml     # control plane machine config
│   ├── worker.yaml           # worker machine config
│   └── kubeconfig            # kubectl config (post-bootstrap)
```

### Files / Resources in Scope

| File/Resource | Action | Notes |
|---------------|--------|-------|
| `infrastructure/vagrant/Vagrantfile` | Create | Main Vagrant config |
| `infrastructure/vagrant/config/cluster.yaml` | Create | Cluster parameters |
| `infrastructure/vagrant/scripts/generate-configs.sh` | Create | Generate Talos configs |
| `infrastructure/vagrant/scripts/bootstrap-talos.sh` | Create | Bootstrap cluster |
| `scripts/local-up.sh` | Create | One-command startup |
| `scripts/local-down.sh` | Create | Cleanup script |
| `.secrets/` | Create | Gitignored secrets directory |
| `.gitignore` | Update | Add secrets exclusions |

### Network Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     HOST MACHINE                                │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  VirtualBox Host-Only Network (vboxnet0)                │   │
│   │  192.168.56.1/24 (host interface)                       │   │
│   │                                                         │   │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐                   │   │
│   │  │ cp-01   │ │ cp-02   │ │ cp-03   │                   │   │
│   │  │ .10     │ │ .11     │ │ .12     │                   │   │
│   │  └────┬────┘ └────┬────┘ └────┬────┘                   │   │
│   │       │           │           │                         │   │
│   │  ┌────┴───────────┴───────────┴────┐                   │   │
│   │  │      Control Plane VIP          │                   │   │
│   │  │      192.168.56.100:6443        │                   │   │
│   │  └─────────────────────────────────┘                   │   │
│   │                                                         │   │
│   │  ┌─────────┐ ┌─────────┐                               │   │
│   │  │worker-01│ │worker-02│                               │   │
│   │  │ .20     │ │ .21     │                               │   │
│   │  └─────────┘ └─────────┘                               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   NAT Network (per VM) ──────────────────────► Internet         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Kubernetes API:  https://192.168.56.100:6443 (VIP)
                 https://192.168.56.10:6443  (direct to cp-01)
Talos API:       https://192.168.56.10:50000 (each node)
```

---

## Implementation Approach

```
Phase 1: Vagrantfile & VM Configuration
1. Create Vagrantfile with VM definitions
2. Configure dual networking (host-only + NAT)
3. Set resources per node role
4. Download/reference Talos ISO

Phase 2: Talos Configuration Generation
1. Script to run `talosctl gen config`
2. Apply patches for Cilium (no default CNI)
3. Configure control plane endpoint (VIP or first CP)
4. Generate separate configs for CP and workers

Phase 3: Bootstrap Scripts
1. `local-up.sh`: Vagrant up + apply configs + bootstrap
2. `local-down.sh`: Vagrant destroy + cleanup
3. Idempotent operations with status checks

Phase 4: Documentation
1. Prerequisites
2. Usage instructions
3. Troubleshooting common issues
```

---

## Reference Materials

| Type | Link | Notes |
|------|------|-------|
| Talos Getting Started | https://www.talos.dev/v1.6/introduction/getting-started/ | |
| Talos VirtualBox Guide | https://www.talos.dev/v1.6/talos-guides/install/virtualized-platforms/virtualbox/ | |
| talosctl Reference | https://www.talos.dev/v1.6/reference/cli/ | |
| Talos Machine Config | https://www.talos.dev/v1.6/reference/configuration/ | |

---

## Risks & Rollback

### Potential Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Host resources insufficient | Med | High | Document minimums, allow single-node option |
| VirtualBox networking issues | Med | Med | Fallback to NAT-only + port forwards |
| Talos ISO download fails | Low | Low | Cache ISO locally |
| Bootstrap timeout | Med | Med | Retry logic, increase timeouts |

### Rollback Plan

```
Local dev environment is disposable:
1. vagrant destroy -f
2. rm -rf .secrets/
3. Start fresh with `local-up.sh`
```

---

## Validation Steps

### Manual Verification

```bash
# Check VMs are running
vagrant status

# Check Talos API reachable
talosctl --talosconfig=.secrets/talosconfig health

# Check cluster bootstrapped
talosctl --talosconfig=.secrets/talosconfig cluster show

# Get kubeconfig
talosctl --talosconfig=.secrets/talosconfig kubeconfig .secrets/kubeconfig

# Check nodes
kubectl --kubeconfig=.secrets/kubeconfig get nodes
# Expected: All nodes in NotReady (no CNI yet)
```

### Automated Tests

```
- [ ] All VMs boot within 5 minutes
- [ ] Talos API responds on all nodes
- [ ] Control plane bootstrap completes
- [ ] kubeconfig is generated and valid
- [ ] kubectl can list nodes
```

---

## Notes

```
- Talos boots with NO CNI by default — nodes will be NotReady until Cilium is deployed (Task 03)
- Control plane VIP requires a load balancer or we use first CP node directly
- Consider MetalLB or kube-vip for VIP in later task
- For faster iteration, can start with 1 CP + 1 Worker config
- talosctl gen config creates both controlplane.yaml and worker.yaml

Minimum viable test:
- 1 control plane (192.168.56.10)
- 1 worker (192.168.56.20)
- Skip VIP, use CP IP directly
```

---

*Created: 2024-01-20*
*Status: Draft*
