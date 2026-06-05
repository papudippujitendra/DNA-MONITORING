This is a more detailed and better-scoped prompt than the previous one. Let me deliver against it precisely.

---

## UAT AKS Monitoring Stack — Dedicated Node Pool Redesign

---

### 1. `prometheus-values.yaml`

```yaml
## ── PROMETHEUS OPERATOR ──────────────────────────────────────────────────────
prometheusOperator:
  nodeSelector:
    node-role: monitoring
  tolerations:
    - key: node-role
      operator: Equal
      value: monitoring
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role
                operator: In
                values:
                  - monitoring

## ── PROMETHEUS ───────────────────────────────────────────────────────────────
prometheus:
  prometheusSpec:
    retention: 15d                          # UNCHANGED
    retentionSize: "12GB"                   # ADDED — prevents disk exhaustion at 15Gi cap
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: managed-csi
          resources:
            requests:
              storage: 15Gi               # +5Gi — provides headroom for retentionSize guard

    serviceMonitorSelectorNilUsesHelmValues: false   # UNCHANGED
    podMonitorSelectorNilUsesHelmValues: false        # UNCHANGED

    # ADDED — explicit cluster-wide scraping scope (see note below)
    serviceMonitorNamespaceSelector: {}
    podMonitorNamespaceSelector: {}
    ruleNamespaceSelector: {}

    # ADDED — resource requests/limits for UAT sizing
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 1500m
        memory: 3Gi

    # MANDATORY — schedule on monitoring node pool only
    nodeSelector:
      node-role: monitoring
    tolerations:
      - key: node-role
        operator: Equal
        value: monitoring
        effect: NoSchedule
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: node-role
                  operator: In
                  values:
                    - monitoring

## ── ALERTMANAGER ─────────────────────────────────────────────────────────────
alertmanager:
  enabled: true
  alertmanagerSpec:
    replicas: 1                             # UNCHANGED — single replica acceptable for UAT

    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi

    storage:
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          storageClassName: managed-csi
          resources:
            requests:
              storage: 2Gi                  # UNCHANGED

    # MANDATORY
    nodeSelector:
      node-role: monitoring
    tolerations:
      - key: node-role
        operator: Equal
        value: monitoring
        effect: NoSchedule
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
            - matchExpressions:
                - key: node-role
                  operator: In
                  values:
                    - monitoring

  config:
    global:
      resolve_timeout: 5m                   # UNCHANGED
    route:
      receiver: default
      group_by:
        - namespace
        - alertname                         # ADDED alertname — improves alert grouping clarity
      group_wait: 30s                       # UNCHANGED
      group_interval: 5m                    # UNCHANGED
      repeat_interval: 12h                  # UNCHANGED
      routes:
        - receiver: "null"
          matchers:
            - alertname =~ "Watchdog|InfoInhibitor"   # ADDED — suppresses constant noise alerts
    receivers:
      - name: default
      - name: "null"
    # Outlook SMTP — add when SMTP relay details are confirmed:
    # global:
    #   smtp_smarthost: 'smtp.office365.com:587'
    #   smtp_from: 'alerts@yourdomain.com'
    #   smtp_auth_username: 'alerts@yourdomain.com'
    #   smtp_auth_password: '<app-password>'
    #   smtp_require_tls: true
    # receivers:
    #   - name: default
    #     email_configs:
    #       - to: 'team@yourdomain.com'
    #         send_resolved: true

## ── GRAFANA ──────────────────────────────────────────────────────────────────
grafana:
  enabled: true
  adminPassword: "admin@123"                # UNCHANGED — see Optional Improvements

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 300m
      memory: 256Mi

  persistence:
    enabled: true
    storageClassName: managed-csi
    size: 5Gi                               # UNCHANGED — adequate for UAT

  service:
    type: LoadBalancer                      # UNCHANGED

  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki.monitoring.svc.cluster.local:3100   # UPDATED — FQDN is safer (see note)
      access: proxy
      isDefault: false

  # MANDATORY
  nodeSelector:
    node-role: monitoring
  tolerations:
    - key: node-role
      operator: Equal
      value: monitoring
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role
                operator: In
                values:
                  - monitoring

## ── KUBE-STATE-METRICS ───────────────────────────────────────────────────────
kubeStateMetrics:
  enabled: true                             # UNCHANGED

kube-state-metrics:                         # Sub-chart key — hyphenated, not camelCase
  nodeSelector:
    node-role: monitoring
  tolerations:
    - key: node-role
      operator: Equal
      value: monitoring
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role
                operator: In
                values:
                  - monitoring

## ── NODE EXPORTER ────────────────────────────────────────────────────────────
## Must run on ALL nodes — no nodeSelector, only universal toleration
nodeExporter:
  enabled: true                             # UNCHANGED

prometheus-node-exporter:                   # Sub-chart key
  tolerations:
    - operator: Exists                      # Tolerates all taints — ensures full node coverage

## ── KUBELET ──────────────────────────────────────────────────────────────────
kubelet:
  enabled: true                             # UNCHANGED
```

**`serviceMonitorNamespaceSelector: {}`  — why add it:**
Without this, the chart default scopes ServiceMonitor discovery to the release namespace only when `serviceMonitorSelectorNilUsesHelmValues: false` is combined with certain chart versions. Setting it explicitly to `{}` (empty = all namespaces) is a one-line safeguard that guarantees cross-namespace scraping regardless of chart version behaviour.

---

### 2. `loki-values.yaml`

```yaml
loki:
  auth_enabled: false                       # UNCHANGED
  commonConfig:
    replication_factor: 1                   # UNCHANGED
  storage:
    type: filesystem                        # UNCHANGED
  schema_config:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: filesystem
        schema: v13
        index:
          prefix: index_
          period: 24h
  limits_config:
    retention_period: 168h                  # UNCHANGED — 7 days is fine for UAT

deploymentMode: SingleBinary               # UNCHANGED

singleBinary:
  replicas: 1                               # UNCHANGED

  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

  persistence:
    enabled: true
    storageClass: managed-csi
    size: 20Gi                              # UNCHANGED — 20Gi is sufficient for 7d UAT retention

  # MANDATORY
  nodeSelector:
    node-role: monitoring
  tolerations:
    - key: node-role
      operator: Equal
      value: monitoring
      effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role
                operator: In
                values:
                  - monitoring

read:
  replicas: 0
write:
  replicas: 0
backend:
  replicas: 0

monitoring:
  selfMonitoring:
    enabled: false
    grafanaAgent:
      installOperator: false
  lokiCanary:
    enabled: false
test:
  enabled: false
```

**No Loki config changes required beyond node scheduling.** 20Gi with 7-day retention is appropriate for UAT. Only increase to 25Gi if log ingestion rate consistently exceeds ~3GB/day.

---

### 3. `promtail-values.yaml`

```yaml
## Promtail — DaemonSet, runs on ALL nodes, NO nodeSelector

tolerations:
  - key: node-role.kubernetes.io/master
    operator: Exists
    effect: NoSchedule
  - operator: "Exists"                      # UNCHANGED — already covers node-role=monitoring:NoSchedule
    effect: "NoSchedule"

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 150m
    memory: 128Mi

config:
  clients:
    - url: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push  # UPDATED to FQDN
  snippets:
    pipelineStages:
      - cri: {}
    scrapeConfigs: |                        # UNCHANGED
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name,
              __meta_kubernetes_pod_label_app, __tmp_controller_name,
              __meta_kubernetes_pod_name]
            regex: ^;*([^;]+)(;.*)?$
            action: replace
            target_label: app
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: namespace
          - action: replace
            replacement: $1
            separator: /
            source_labels: [namespace, app]
            target_label: job
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: pod
          - source_labels: [__meta_kubernetes_pod_container_name]
            action: replace
            target_label: container
          - action: replace
            replacement: /var/log/pods/*$1/*.log
            separator: /
            source_labels: [__meta_kubernetes_pod_uid,
              __meta_kubernetes_pod_container_name]
            target_label: __path__
```

**Promtail toleration note:** The existing `operator: Exists / effect: NoSchedule` already tolerates `node-role=monitoring:NoSchedule`. No new toleration entry is required. The only actual change is the Loki URL to FQDN.

---

### 4. Deployment Commands

```bash
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml \
  --version 58.7.2 \
  --wait --timeout 10m

helm upgrade --install loki \
  grafana/loki \
  --namespace monitoring \
  --values loki-values.yaml \
  --version 6.6.4 \
  --wait --timeout 10m

helm upgrade --install promtail \
  grafana/promtail \
  --namespace monitoring \
  --values promtail-values.yaml \
  --version 6.15.5 \
  --wait --timeout 5m
```

---

### 5. Verification Commands

```bash
# All pods Running — check NODE column for placement
kubectl get pods -n monitoring -o wide

# Confirm stateful components are on monitoring nodes only
kubectl get pods -n monitoring -o wide \
  | grep -E "prometheus|alertmanager|grafana|loki|state-metrics|operator"

# Promtail DaemonSet: DESIRED must equal cluster node count
kubectl get daemonset -n monitoring promtail

# node-exporter: also must equal cluster node count
kubectl get daemonset -n monitoring kube-prometheus-stack-prometheus-node-exporter

# PVCs all Bound
kubectl get pvc -n monitoring

# Prometheus scrape targets — all namespaces/nodes should appear
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Open: http://localhost:9090/targets

# Confirm Prometheus node scheduling
kubectl get pod -n monitoring -l app.kubernetes.io/name=prometheus \
  -o jsonpath='{.items[0].spec.nodeName}'

# Loki receiving logs from Promtail
kubectl logs -n monitoring -l app.kubernetes.io/name=promtail --tail=20 | grep "msg="
```

---

### 6. Mandatory Changes

| # | Component | Change | Reason |
|---|---|---|---|
| 1 | `prometheusOperator` | Add nodeSelector + tolerations + affinity | Operator must run on monitoring nodes |
| 2 | `prometheus.prometheusSpec` | Add nodeSelector + tolerations + affinity | Core requirement |
| 3 | `alertmanager.alertmanagerSpec` | Add nodeSelector + tolerations + affinity | Core requirement |
| 4 | `grafana` | Add nodeSelector + tolerations + affinity | Core requirement |
| 5 | `kube-state-metrics` (sub-chart) | Add nodeSelector + tolerations + affinity under hyphenated key | Core requirement — wrong key = silently ignored |
| 6 | `prometheus-node-exporter` (sub-chart) | Add `tolerations: - operator: Exists` only | Must run on all nodes including monitoring-tainted ones |
| 7 | `loki.singleBinary` | Add nodeSelector + tolerations + affinity | Core requirement |
| 8 | Prometheus `storage` | 10Gi → 15Gi | Required to safely accommodate `retentionSize: 12GB` guard |
| 9 | Prometheus `retentionSize` | Not set → `12GB` | Prevents TSDB from consuming 100% of 15Gi and crashing |
| 10 | Promtail | No scheduling change needed | Existing `operator: Exists` already covers monitoring taint |

---

### 7. Optional Improvements

| # | Component | Suggestion | Detail |
|---|---|---|---|
| 1 | All components | Add resource requests/limits | Prevents eviction and noisy-neighbour issues on shared monitoring node; UAT-sized values included in values above |
| 2 | Grafana `adminPassword` | Move to Kubernetes secret | `admin@123` in plaintext values.yaml is a credential risk; use `--set grafana.adminPassword=$VAR` at deploy time |
| 3 | Grafana Loki datasource URL | `http://loki:3100` → FQDN | Short-form DNS only resolves within the same namespace; FQDN works if Grafana is ever moved |
| 4 | Alertmanager `routes` | Add null route for Watchdog/InfoInhibitor | These alerts fire constantly and are noise in UAT; suppress them with a null receiver route |
| 5 | Alertmanager `group_by` | Add `alertname` to `group_by` | Grouping only by `namespace` merges unrelated alerts into the same notification |
| 6 | Alertmanager SMTP | Add Outlook SMTP config when ready | Template included as comments in values above; requires Office 365 app password or relay |
| 7 | `serviceMonitorNamespaceSelector: {}` | Explicitly add | Guarantees cross-namespace scraping regardless of chart version behaviour |

---

### 8. Risks and Recommendations

**Single monitoring node = single point of failure.**
If the monitoring node is cordoned, drained, or fails, Prometheus, Alertmanager, Grafana, and Loki all go down simultaneously. You lose visibility precisely when you most need it.

For UAT this is acceptable if the risk is understood. The minimum mitigation is two monitoring nodes so Kubernetes can reschedule pods during node maintenance without a full monitoring blackout. With `managed-csi` (Azure Disk, `ReadWriteOnce`), the PVC must detach from the failed node before it can reattach on the second — this takes 2–6 minutes on AKS. That is the recovery window to plan for.

**Sub-chart key naming is a silent failure mode.** `kube-state-metrics:` (hyphenated) and `prometheus-node-exporter:` are sub-chart override keys. If you use the camelCase keys (`kubeStateMetrics:`, `nodeExporter:`) for scheduling config, Helm silently ignores those values and the pods schedule on app nodes with no error. Verify with `kubectl get pods -o wide` after deploy.

**`ReadWriteOnce` blocks HA upgrades.** All PVCs use `managed-csi` (Azure Disk), which is `ReadWriteOnce` — only one pod can mount at a time. If you later want to scale Prometheus or Loki replicas, you'll need storage migration. For UAT this is a non-issue; just worth noting if this config is ever promoted toward production.# DNA-MONITORING
