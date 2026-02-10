# Task 03: Prometheus Operator Stack — Metrics & Alerting

## Objective

```
GOAL:         Deploy kube-prometheus-stack (Prometheus Operator, Prometheus,
              Alertmanager, Grafana, kube-state-metrics, node-exporter)
SCOPE:        kubernetes/observability/prometheus-stack/
OUT OF SCOPE: Loki logging (Task 04), custom dashboards (Task 05)
```

---

## Context

```
WHY:          Core observability stack for metrics collection, alerting,
              and visualization. Foundation for SRE practices.
TICKET:       SRE-003
URGENCY:      High
DEADLINE:     Week 2
```

### Background

The kube-prometheus-stack Helm chart deploys:
- **Prometheus Operator**: Manages Prometheus instances via CRDs
- **Prometheus**: Time-series metrics collection and storage
- **Alertmanager**: Alert routing and notification
- **Grafana**: Visualization dashboards
- **kube-state-metrics**: Kubernetes object metrics
- **node-exporter**: Host-level metrics

All components expose metrics that Prometheus scrapes via ServiceMonitors.

### Blocked By

- [x] Task 01: Cluster running
- [x] Task 02: Cilium CNI deployed, nodes Ready

---

## Acceptance Criteria

- [ ] kube-prometheus-stack deployed to `monitoring` namespace
- [ ] Prometheus Operator running and managing Prometheus CRs
- [ ] Prometheus instance running with persistent storage
- [ ] Alertmanager running (basic config)
- [ ] Grafana running with default dashboards
- [ ] kube-state-metrics collecting K8s object metrics
- [ ] node-exporter running on all nodes (DaemonSet)
- [ ] ServiceMonitors auto-discovering targets (including Cilium from Task 02)
- [ ] Basic alerting rules deployed (node down, pod crash, etc.)
- [ ] Grafana accessible via Ingress/NodePort
- [ ] Works for both local and cloud with value overlays
- [ ] Prometheus retention configured (2d local, 15d cloud)

### Definition of Done

- [ ] All pods in `monitoring` namespace Running
- [ ] Prometheus targets page shows all expected targets
- [ ] Grafana dashboards load with data
- [ ] Test alert fires and routes correctly

---

## Constraints

```
MUST:
  - Use kube-prometheus-stack Helm chart
  - Create `monitoring` namespace
  - Enable persistent storage for Prometheus (PVC)
  - Configure Grafana admin password via Secret
  - Include default K8s dashboards
  - Include Cilium/Hubble dashboards
  - ServiceMonitors for: Cilium, node-exporter, kube-state-metrics, kubelet
  - Basic alert rules for: NodeDown, PodCrashLooping, HighMemory, HighCPU

MUST NOT:
  - Use default Grafana password (admin/admin)
  - Deploy without persistence (data loss on restart)
  - Enable unnecessary exporters (keep minimal)
  - Expose Prometheus/Alertmanager externally without auth

PREFER:
  - Use Grafana sidecar for dashboard provisioning
  - Alertmanager config via Secret (not inline)
  - Separate values files for local vs cloud
  - Resource limits appropriate to environment
```

---

## Technical Details

### Current State

```
- Cluster running with Cilium CNI
- No observability stack deployed
- Cilium metrics available but not scraped
```

### Desired State

```
kubernetes/observability/prometheus-stack/
├── kustomization.yaml
├── namespace.yaml
├── helm-release.yaml           # If using Flux
├── values.yaml                 # Base values
├── values-local.yaml           # Local (small resources, short retention)
├── values-cloud.yaml           # Cloud (larger resources, longer retention)
├── alertmanager-config.yaml    # Alert routing config
├── additional-rules.yaml       # Custom PrometheusRules
└── dashboards/
    ├── configmap-dashboards.yaml
    └── *.json                  # Dashboard JSON files

Deployed Resources (monitoring namespace):
├── prometheus-operator (Deployment)
├── prometheus-kube-prometheus-prometheus (StatefulSet)
├── alertmanager-kube-prometheus-alertmanager (StatefulSet)
├── kube-prometheus-grafana (Deployment)
├── kube-prometheus-kube-state-metrics (Deployment)
├── kube-prometheus-prometheus-node-exporter (DaemonSet)
├── ServiceMonitors (multiple)
└── PrometheusRules (alerting rules)

Endpoints:
├── Grafana:      :3000 (NodePort 30300 or Ingress)
├── Prometheus:   :9090 (NodePort 30900 or internal)
├── Alertmanager: :9093 (NodePort 30903 or internal)
```

### Storage Requirements

| Component | Local | Cloud | Storage Class |
|-----------|-------|-------|---------------|
| Prometheus | 10Gi | 50Gi | local-path / gp3 |
| Alertmanager | 1Gi | 5Gi | local-path / gp3 |
| Grafana | (stateless) | (stateless) | N/A |

### Files / Resources in Scope

| File/Resource | Action | Notes |
|---------------|--------|-------|
| `kubernetes/observability/prometheus-stack/namespace.yaml` | Create | monitoring namespace |
| `kubernetes/observability/prometheus-stack/values.yaml` | Create | Base Helm values |
| `kubernetes/observability/prometheus-stack/values-local.yaml` | Create | Local overrides |
| `kubernetes/observability/prometheus-stack/values-cloud.yaml` | Create | Cloud overrides |
| `kubernetes/observability/prometheus-stack/kustomization.yaml` | Create | Kustomize wrapper |
| `kubernetes/observability/prometheus-stack/alertmanager-config.yaml` | Create | Alert routing |
| `kubernetes/observability/prometheus-stack/additional-rules.yaml` | Create | Custom alerts |
| `kubernetes/observability/prometheus-stack/dashboards/` | Create | Custom dashboards |
| `scripts/deploy-prometheus.sh` | Create | Deployment script |

### Key Helm Values

```yaml
# Base values.yaml
namespaceOverride: monitoring

prometheus:
  prometheusSpec:
    retention: 2d  # Override per environment
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi  # Override per environment
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    # Discover all ServiceMonitors in all namespaces
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 1Gi

grafana:
  adminPassword: ""  # Set via separate secret
  persistence:
    enabled: false  # Stateless, dashboards via ConfigMaps
  sidecar:
    dashboards:
      enabled: true
      searchNamespace: ALL
  service:
    type: NodePort
    nodePort: 30300

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true

# Default alerting rules
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: false  # Talos manages etcd differently
    kubeApiserver: true
    kubeApiserverAvailability: true
    kubeApiserverSlos: true
    kubelet: true
    kubeScheduler: false  # Talos specific
    kubeControllerManager: false  # Talos specific
    node: true
    nodeExporterAlerting: true
    prometheus: true
```

---

## Implementation Approach

```
Phase 1: Namespace & Prerequisites
1. Create monitoring namespace
2. Create Grafana admin secret
3. Ensure storage class available

Phase 2: Helm Installation
1. Add prometheus-community Helm repo
2. Create values files
3. Install kube-prometheus-stack
4. Wait for all pods Ready

Phase 3: Verification
1. Check all targets in Prometheus
2. Verify Grafana dashboards
3. Test alert routing (fire test alert)

Phase 4: Cilium Integration
1. Verify Cilium ServiceMonitors discovered
2. Import Cilium dashboards to Grafana
3. Confirm Hubble metrics in Prometheus
```

---

## Reference Materials

| Type | Link | Notes |
|------|------|-------|
| kube-prometheus-stack | https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack | Helm chart |
| Values Reference | https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml | All options |
| Prometheus Operator | https://prometheus-operator.dev/ | CRD documentation |
| Cilium Dashboards | https://github.com/cilium/cilium/tree/main/examples/kubernetes/addons/prometheus | |

---

## Risks & Rollback

### Potential Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| PVC not provisioned (no storage class) | Med | High | Verify storage class first |
| Resource constraints in local | Med | Med | Use minimal values-local.yaml |
| CRD conflicts with existing | Low | High | Check for existing prometheus CRDs |
| Grafana exposed without auth | Med | Med | Use strong password, internal only |

### Rollback Plan

```
1. helm uninstall kube-prometheus-stack -n monitoring
2. kubectl delete namespace monitoring
3. kubectl delete crd prometheuses.monitoring.coreos.com (etc.)
4. Re-deploy with corrected values
```

---

## Validation Steps

### Manual Verification

```bash
# Check all pods
kubectl get pods -n monitoring

# Check Prometheus targets
kubectl port-forward -n monitoring svc/kube-prometheus-prometheus 9090:9090
open http://localhost:9090/targets
# Expected: All targets UP (green)

# Check Alertmanager
kubectl port-forward -n monitoring svc/kube-prometheus-alertmanager 9093:9093
open http://localhost:9093

# Access Grafana
# NodePort:
open http://<node-ip>:30300
# Or port-forward:
kubectl port-forward -n monitoring svc/kube-prometheus-grafana 3000:80
open http://localhost:3000

# Check Cilium metrics in Prometheus
# Query: cilium_endpoint_state
# Query: hubble_flows_processed_total

# Fire test alert (optional)
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: test-alert
  namespace: monitoring
spec:
  groups:
  - name: test
    rules:
    - alert: TestAlert
      expr: vector(1)
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Test alert"
EOF
```

### Automated Tests

```
- [ ] All pods in monitoring namespace Running
- [ ] Prometheus has >0 active targets
- [ ] Grafana responds on service port
- [ ] kube-state-metrics producing metrics
- [ ] node-exporter running on all nodes
- [ ] Cilium metrics present in Prometheus
```

---

## Notes

```
- Talos doesn't expose kube-scheduler/kube-controller-manager metrics the standard way
  — disable those default rules or configure Talos to expose them
- Grafana dashboards can be added via ConfigMaps with specific labels
- For production, consider:
  - Remote write to long-term storage (Thanos, Mimir, VictoriaMetrics)
  - Alertmanager integration with PagerDuty/Slack/etc.
  - Grafana OIDC authentication
- Local environment: 10Gi Prometheus storage sufficient for ~2 days
- Cloud environment: Size based on cardinality and retention needs

Next task (Task 04): Add Loki for log aggregation
```

---

*Created: 2024-01-20*
*Status: Draft*
*Depends on: Task 01, Task 02*
