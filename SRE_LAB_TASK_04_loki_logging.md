# Task 04: Loki Stack — Log Aggregation & Collection

## Objective

```
GOAL:         Deploy Loki for log aggregation and Promtail for log collection,
              integrated with Grafana for unified observability
SCOPE:        kubernetes/observability/loki-stack/
OUT OF SCOPE: Application-specific logging, log-based alerting (future)
```

---

## Context

```
WHY:          Complete the observability stack with centralized logging.
              Enables correlation between metrics (Prometheus) and logs (Loki)
              in a single Grafana interface.
TICKET:       SRE-004
URGENCY:      Medium
DEADLINE:     Week 2
```

### Background

Loki is a horizontally-scalable, highly-available log aggregation system inspired by Prometheus. Key features:
- **Label-based indexing**: Only indexes metadata (labels), not log content
- **Cost-effective**: Much cheaper than Elasticsearch for log storage
- **Prometheus-native**: Uses same label concepts, integrates with Grafana
- **Promtail**: Agent that ships logs to Loki (similar to Prometheus node-exporter)

Architecture:
```
Nodes (Promtail) ──► Loki ──► Grafana
                      │
                      ▼
                   Storage
              (filesystem/S3/GCS)
```

### Blocked By

- [x] Task 01: Cluster running
- [x] Task 02: Cilium CNI deployed
- [x] Task 03: Prometheus stack (Grafana) deployed

---

## Acceptance Criteria

- [ ] Loki deployed to `logging` namespace (or `monitoring`)
- [ ] Promtail DaemonSet running on all nodes
- [ ] Promtail collecting logs from all pods and system components
- [ ] Loki accessible as Grafana datasource
- [ ] Logs queryable via LogQL in Grafana Explore
- [ ] Persistent storage configured for Loki
- [ ] Talos system logs collected (kubelet, containerd)
- [ ] Works for both local and cloud deployments
- [ ] Retention policy configured (3 days local, 7+ days cloud)
- [ ] ServiceMonitor for Loki/Promtail metrics

### Definition of Done

- [ ] Logs visible in Grafana Explore
- [ ] Can query by namespace, pod, container
- [ ] System logs (Talos) appearing
- [ ] No Promtail errors in logs

---

## Constraints

```
MUST:
  - Use Loki Helm chart (grafana/loki or grafana/loki-stack)
  - Deploy Promtail as DaemonSet
  - Configure Grafana datasource automatically
  - Collect logs from: pods, kubelet, containerd
  - Persistent storage for Loki index and chunks
  - Parse common log formats (JSON, logfmt)

MUST NOT:
  - Use single-binary mode in cloud (not scalable)
  - Store logs without retention limits (disk exhaustion)
  - Expose Loki externally without authentication
  - Collect logs from sensitive namespaces without redaction

PREFER:
  - Use loki-stack chart (includes Promtail)
  - Use filesystem storage for local, S3/GCS for cloud
  - Match Prometheus label conventions
  - Enable multi-tenancy for future use
```

---

## Technical Details

### Current State

```
- Prometheus stack deployed
- Grafana running
- No log aggregation
- Logs only accessible via kubectl logs
```

### Desired State

```
kubernetes/observability/loki-stack/
├── kustomization.yaml
├── namespace.yaml              # If separate from monitoring
├── helm-release.yaml
├── values.yaml                 # Base values
├── values-local.yaml           # Local (filesystem storage)
├── values-cloud.yaml           # Cloud (S3/GCS storage)
├── grafana-datasource.yaml     # Add Loki to Grafana
└── servicemonitor.yaml         # Prometheus scraping

Deployed Resources:
├── loki (StatefulSet or Deployment)
├── promtail (DaemonSet on all nodes)
├── loki-gateway (optional, for multi-tenant)
└── ServiceMonitor

Endpoints:
├── Loki:     :3100 (internal)
├── Promtail: :9080/metrics (per node)
└── Grafana:  Loki datasource added
```

### Storage Configuration

| Environment | Storage Backend | Retention | Size |
|-------------|-----------------|-----------|------|
| Local | Filesystem (PVC) | 72h | 10Gi |
| Cloud (AWS) | S3 bucket | 7d | N/A (object storage) |
| Cloud (GCP) | GCS bucket | 7d | N/A (object storage) |

### Files / Resources in Scope

| File/Resource | Action | Notes |
|---------------|--------|-------|
| `kubernetes/observability/loki-stack/namespace.yaml` | Create | logging namespace (optional) |
| `kubernetes/observability/loki-stack/values.yaml` | Create | Base Helm values |
| `kubernetes/observability/loki-stack/values-local.yaml` | Create | Filesystem storage |
| `kubernetes/observability/loki-stack/values-cloud.yaml` | Create | Object storage |
| `kubernetes/observability/loki-stack/kustomization.yaml` | Create | Kustomize wrapper |
| `kubernetes/observability/loki-stack/grafana-datasource.yaml` | Create | Datasource ConfigMap |
| `scripts/deploy-loki.sh` | Create | Deployment script |

### Key Helm Values

```yaml
# values.yaml (base)
loki:
  auth_enabled: false  # Enable for multi-tenant
  
  commonConfig:
    replication_factor: 1  # Increase for HA
    
  storage:
    type: filesystem  # Override for cloud
    
  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h
          
  limits_config:
    retention_period: 72h
    ingestion_rate_mb: 10
    ingestion_burst_size_mb: 20
    max_streams_per_user: 10000
    
  compactor:
    retention_enabled: true

promtail:
  enabled: true
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push
        
    snippets:
      pipelineStages:
        - cri: {}  # Parse CRI format (containerd)
        - json:
            expressions:
              level: level
              msg: msg
        - labels:
            level:
            
  # Collect from Talos system logs
  extraVolumes:
    - name: machine-logs
      hostPath:
        path: /var/log
  extraVolumeMounts:
    - name: machine-logs
      mountPath: /var/log/host
      readOnly: true

# Grafana datasource (if using loki-stack chart)
grafana:
  enabled: false  # Already deployed via prometheus-stack
  
# Add as sidecar datasource to existing Grafana
serviceMonitor:
  enabled: true
```

### Grafana Datasource ConfigMap

```yaml
# grafana-datasource.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-grafana-datasource
  namespace: monitoring
  labels:
    grafana_datasource: "1"  # Picked up by Grafana sidecar
data:
  loki-datasource.yaml: |
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki.logging.svc:3100
        jsonData:
          maxLines: 1000
```

---

## Implementation Approach

```
Phase 1: Namespace & Storage
1. Create logging namespace (or use monitoring)
2. Ensure storage class available
3. For cloud: create S3/GCS bucket + IAM

Phase 2: Loki Installation
1. Add grafana Helm repo
2. Create values files
3. Install Loki
4. Verify Loki pods running

Phase 3: Promtail Configuration
1. Verify DaemonSet on all nodes
2. Check Promtail can read container logs
3. Configure Talos system log collection
4. Verify logs flowing to Loki

Phase 4: Grafana Integration
1. Add Loki datasource ConfigMap
2. Restart Grafana or wait for sidecar sync
3. Test queries in Grafana Explore
4. Import useful dashboards
```

---

## Reference Materials

| Type | Link | Notes |
|------|------|-------|
| Loki Helm Chart | https://github.com/grafana/loki/tree/main/production/helm/loki | |
| Loki Documentation | https://grafana.com/docs/loki/latest/ | |
| Promtail Config | https://grafana.com/docs/loki/latest/clients/promtail/configuration/ | |
| LogQL | https://grafana.com/docs/loki/latest/logql/ | Query language |
| Talos Logging | https://www.talos.dev/v1.6/talos-guides/configuration/logging/ | System logs |

---

## Risks & Rollback

### Potential Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Promtail can't read CRI logs | Med | High | Verify socket path, test with kubectl logs |
| Storage fills up | Med | Med | Set retention, monitor disk usage |
| High cardinality labels | Med | Med | Limit labels, use relabeling |
| Talos restricts log paths | Med | Med | Check Talos machine config |

### Rollback Plan

```
1. helm uninstall loki -n logging
2. kubectl delete pvc -n logging -l app=loki
3. Remove Grafana datasource ConfigMap
4. Re-deploy with corrected config
```

---

## Validation Steps

### Manual Verification

```bash
# Check Loki pods
kubectl get pods -n logging -l app=loki

# Check Promtail DaemonSet
kubectl get pods -n logging -l app=promtail -o wide
# Expected: One pod per node

# Check Promtail logs for errors
kubectl logs -n logging -l app=promtail --tail=50

# Test Loki API
kubectl port-forward -n logging svc/loki 3100:3100
curl http://localhost:3100/ready
# Expected: ready

# Check log labels
curl http://localhost:3100/loki/api/v1/labels
# Expected: namespace, pod, container, etc.

# Query via Grafana
# Go to Explore → Select Loki datasource
# Query: {namespace="kube-system"}
# Query: {namespace="monitoring"} |= "error"

# Check Talos system logs
# Query: {job="machine-logs"}
```

### Automated Tests

```
- [ ] Loki pod Running and Ready
- [ ] Promtail pods on all nodes
- [ ] /ready endpoint returns 200
- [ ] Labels API returns expected labels
- [ ] Logs queryable from last hour
- [ ] Grafana datasource configured
```

---

## Useful LogQL Queries

```logql
# All logs from a namespace
{namespace="monitoring"}

# Logs from specific pod
{namespace="kube-system", pod=~"cilium.*"}

# Filter by content
{namespace="default"} |= "error"
{namespace="default"} |~ "error|warn"

# JSON parsing
{namespace="app"} | json | level="error"

# Rate of log lines
rate({namespace="monitoring"}[5m])

# Top namespaces by log volume
sum by (namespace) (rate({job="kubernetes-pods"}[5m]))
```

---

## Notes

```
- Loki is NOT a full-text search engine — use label queries first, then filter
- High cardinality labels (like user_id) will cause performance issues
- Promtail default config handles most Kubernetes log formats
- For Talos: system logs are in /var/log on host, need hostPath mount
- Consider Grafana OnCall for log-based alerting (future enhancement)

Local optimization:
- Use single replica Loki
- Short retention (72h)
- Filesystem storage

Cloud optimization:
- Use distributed mode (read/write separation)
- S3/GCS for cost-effective storage
- Longer retention (7-30 days)
```

---

*Created: 2024-01-20*
*Status: Draft*
*Depends on: Task 01, Task 02, Task 03*
