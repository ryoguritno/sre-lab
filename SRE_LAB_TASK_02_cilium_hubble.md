# Task 02: Cilium CNI + Hubble — Network & eBPF Observability

## Objective

```
GOAL:         Deploy Cilium as CNI with Hubble enabled for eBPF-based 
              network observability and metrics
SCOPE:        kubernetes/infrastructure/cilium/
              infrastructure/talos/patches/cilium.yaml
OUT OF SCOPE: Ingress controller, other networking (separate tasks)
```

---

## Context

```
WHY:          Nodes are NotReady without CNI; Cilium provides modern eBPF-based
              networking with built-in observability via Hubble
TICKET:       SRE-002
URGENCY:      High (blocks all other K8s workloads)
DEADLINE:     Week 1 (after Task 01)
```

### Background

Cilium is the CNI for this SRE Lab because:
- eBPF-based networking (efficient, kernel-level)
- Hubble provides network flow observability
- Native Prometheus metrics endpoint
- Can replace kube-proxy (eBPF mode)
- Strong security policies (NetworkPolicy + CiliumNetworkPolicy)

### Blocked By

- [x] Task 01: Local Vagrant setup complete
- [x] Kubernetes cluster bootstrapped (nodes in NotReady state)

---

## Acceptance Criteria

- [ ] Cilium deployed via Helm to kube-system namespace
- [ ] All nodes transition to Ready state
- [ ] Cilium pods running on every node (DaemonSet)
- [ ] Hubble enabled with UI and metrics
- [ ] `cilium status` shows healthy cluster
- [ ] `hubble status` shows flows being observed
- [ ] Prometheus metrics endpoint exposed (/metrics)
- [ ] Hubble UI accessible via port-forward (and later Ingress)
- [ ] Talos machine config patched to skip default CNI
- [ ] Works for both local (Vagrant) and cloud deployments
- [ ] ServiceMonitor created for Prometheus scraping

### Definition of Done

- [ ] Nodes Ready
- [ ] `cilium connectivity test` passes
- [ ] Hubble UI shows network flows
- [ ] Metrics visible in Prometheus (after Task 04)

---

## Constraints

```
MUST:
  - Use Cilium Helm chart (official)
  - Enable Hubble with UI, relay, and metrics
  - Enable Prometheus metrics on all components
  - Replace kube-proxy with eBPF (Talos supports this)
  - Configure IPAM mode appropriate for environment
  - Work with Talos Linux (no iptables legacy mode)

MUST NOT:
  - Use Cilium CLI for installation (use Helm for GitOps compatibility)
  - Enable features that require kernel modules Talos doesn't have
  - Hard-code cluster-specific values in chart

PREFER:
  - Use Cilium's native Ingress if simpler than nginx
  - Enable L7 visibility where useful
  - Use Helm values overlay for local vs cloud differences
```

---

## Technical Details

### Current State

```
- Kubernetes cluster running (from Task 01)
- All nodes in NotReady state (no CNI)
- No pod networking available
- CoreDNS pending (waiting for CNI)
```

### Desired State

```
kubernetes/infrastructure/cilium/
├── kustomization.yaml
├── namespace.yaml
├── helm-release.yaml       # If using Flux
├── values.yaml             # Base values
├── values-local.yaml       # Local overrides
├── values-cloud.yaml       # Cloud overrides
└── servicemonitor.yaml     # Prometheus scraping

Deployed Resources:
├── cilium-agent (DaemonSet on all nodes)
├── cilium-operator (Deployment)
├── hubble-relay (Deployment)
├── hubble-ui (Deployment)
└── ServiceMonitor (for Prometheus)

Endpoints:
├── Hubble UI:      port-forward → http://localhost:12000
├── Hubble Relay:   port 4245
├── Cilium Metrics: :9962/metrics, :9963/metrics, :9965/metrics
└── Hubble Metrics: :9966/metrics
```

### Talos Configuration Patch

```yaml
# infrastructure/talos/patches/cilium.yaml
# Patch to disable default CNI and kube-proxy for Cilium
cluster:
  network:
    cni:
      name: none  # Cilium will be installed separately
  proxy:
    disabled: true  # Cilium replaces kube-proxy
```

### Files / Resources in Scope

| File/Resource | Action | Notes |
|---------------|--------|-------|
| `kubernetes/infrastructure/cilium/values.yaml` | Create | Base Helm values |
| `kubernetes/infrastructure/cilium/values-local.yaml` | Create | Local overrides |
| `kubernetes/infrastructure/cilium/values-cloud.yaml` | Create | Cloud overrides |
| `kubernetes/infrastructure/cilium/kustomization.yaml` | Create | Kustomize wrapper |
| `kubernetes/infrastructure/cilium/servicemonitor.yaml` | Create | Prometheus integration |
| `infrastructure/talos/patches/cilium.yaml` | Create | Talos CNI patch |
| `scripts/deploy-cilium.sh` | Create | Deployment script |

### Key Helm Values

```yaml
# Base values (values.yaml)
cluster:
  name: sre-lab
  id: 1

ipam:
  mode: kubernetes  # or cluster-pool for cloud

kubeProxyReplacement: true  # eBPF kube-proxy replacement

hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
  metrics:
    enabled:
      - dns
      - drop
      - tcp
      - flow
      - icmp
      - http
    serviceMonitor:
      enabled: true

prometheus:
  enabled: true
  serviceMonitor:
    enabled: true

operator:
  prometheus:
    enabled: true
    serviceMonitor:
      enabled: true
```

---

## Implementation Approach

```
Phase 1: Talos Configuration
1. Create Cilium patch for Talos machine config
2. Regenerate Talos configs with patch applied
3. Apply configs to nodes (or rebuild cluster)

Phase 2: Cilium Installation
1. Add Cilium Helm repo
2. Create values files (base + environment overlays)
3. Install via Helm
4. Wait for DaemonSet rollout

Phase 3: Verification
1. Check cilium status
2. Run connectivity test
3. Verify Hubble flows
4. Confirm Prometheus metrics

Phase 4: Documentation
1. Access instructions (port-forward, etc.)
2. Troubleshooting guide
3. Metric reference
```

---

## Reference Materials

| Type | Link | Notes |
|------|------|-------|
| Cilium on Talos | https://www.talos.dev/v1.6/kubernetes-guides/network/deploying-cilium/ | Official guide |
| Cilium Helm Values | https://github.com/cilium/cilium/blob/main/install/kubernetes/cilium/values.yaml | All options |
| Hubble Docs | https://docs.cilium.io/en/stable/gettingstarted/hubble/ | |
| Cilium Metrics | https://docs.cilium.io/en/stable/observability/metrics/ | |

---

## Risks & Rollback

### Potential Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Cilium incompatible with Talos kernel | Low | High | Use tested version combinations |
| IPAM exhaustion | Low | Med | Size pool appropriately |
| Hubble overhead in local env | Med | Low | Can disable in resource-constrained setups |

### Rollback Plan

```
If Cilium fails:
1. helm uninstall cilium -n kube-system
2. Nodes return to NotReady
3. Debug and re-deploy

Nuclear option:
1. Rebuild cluster from Task 01
2. Ensure Talos patch is correct
3. Re-deploy Cilium
```

---

## Validation Steps

### Manual Verification

```bash
# Check Cilium status
cilium status --wait

# Verify all nodes have Cilium agent
kubectl get pods -n kube-system -l k8s-app=cilium -o wide

# Check nodes are Ready
kubectl get nodes
# Expected: All nodes Ready

# Run connectivity test
cilium connectivity test

# Check Hubble
hubble status
hubble observe --follow

# Access Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
open http://localhost:12000

# Check metrics endpoint
kubectl port-forward -n kube-system ds/cilium 9962:9962
curl http://localhost:9962/metrics
```

### Automated Tests

```
- [ ] All Cilium pods Running
- [ ] All nodes Ready
- [ ] cilium status shows OK
- [ ] hubble status shows flows
- [ ] Metrics endpoints respond
- [ ] Basic connectivity test passes
```

---

## Notes

```
- Cilium on Talos requires kube-proxy disabled (handled by patch)
- First Cilium install may take 2-3 minutes for agents to be ready
- Hubble UI is resource-light but can be disabled for minimal setups
- Metrics cardinality can be high — tune scrape interval in production
- Consider enabling Cilium Ingress Controller instead of nginx (Task evaluation)

Local resource optimization:
- Can disable Hubble UI in local if RAM constrained
- Reduce metrics retention
- Use single replica for hubble-relay
```

---

*Created: 2024-01-20*
*Status: Draft*
*Depends on: Task 01*
