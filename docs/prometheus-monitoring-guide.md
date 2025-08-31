# Prometheus Monitoring Guide for MySQL Helm Chart

## Table of Contents
1. [Quick Start](#quick-start)
2. [Installing Prometheus in Minikube](#installing-prometheus-in-minikube)
3. [MySQL with Metrics Configuration](#mysql-with-metrics-configuration)
4. [Verifying the Setup](#verifying-the-setup)
5. [Grafana Dashboard Setup](#grafana-dashboard-setup)
6. [Troubleshooting](#troubleshooting)

## Quick Start

### Prerequisites
```bash
# Ensure Minikube is running with sufficient resources
minikube start --driver=docker --memory=4096 --cpus=2

# Enable metrics-server addon (optional but recommended)
minikube addons enable metrics-server
```

## Installing Prometheus in Minikube

### Option 1: Using Prometheus Operator (Recommended)

```bash
# Add prometheus-community helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack (includes Prometheus, Grafana, and Alertmanager)
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=prometheus -n monitoring --timeout=300s
```

### Option 2: Standalone Prometheus (Minimal)

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install standalone Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.enabled=false
```

## MySQL with Metrics Configuration

### 1. Create MySQL Values File with Metrics

Create `mysql-metrics-values.yaml`:

```yaml
# Minimal configuration with metrics
architecture: standalone

auth:
  rootPassword: "mysqlroot123"
  database: "testdb"

primary:
  persistence:
    enabled: true
    size: 2Gi
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"

# Metrics configuration
metrics:
  enabled: true
  image:
    registry: docker.io
    repository: prom/mysqld-exporter
    tag: "v0.15.1"
    pullPolicy: IfNotPresent
  
  resources:
    limits:
      cpu: 100m
      memory: 128Mi
    requests:
      cpu: 50m
      memory: 64Mi
  
  livenessProbe:
    enabled: true
    initialDelaySeconds: 15
    periodSeconds: 10
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 3
  
  readinessProbe:
    enabled: true
    initialDelaySeconds: 5
    periodSeconds: 10
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 3
  
  # ServiceMonitor for Prometheus Operator
  serviceMonitor:
    enabled: true
    namespace: ""  # Uses release namespace if empty
    interval: 30s
    scrapeTimeout: 10s
    labels: {}
    selector: {}
    relabelings: []
    metricRelabelings: []
    honorLabels: false
    jobLabel: ""
  
  # PrometheusRule for alerting
  prometheusRule:
    enabled: true
    namespace: ""
    labels: {}
    groups: []  # Will use default rules from template
```

### 2. Install MySQL with Metrics

```bash
# Install MySQL with metrics enabled
helm install my-mysql ./mysql-server-helm \
  -f mysql-metrics-values.yaml

# Verify pods are running
kubectl get pods -l app.kubernetes.io/instance=my-mysql
```

### 3. Advanced Configuration with Replication

Create `mysql-metrics-replication.yaml`:

```yaml
# Full configuration with replication and metrics
architecture: replication

auth:
  rootPassword: "mysqlroot123"
  database: "proddb"
  username: "appuser"
  password: "apppass123"
  replicationUser: "replicator"
  replicationPassword: "replpass123"

primary:
  persistence:
    enabled: true
    size: 10Gi
    storageClass: standard
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  service:
    type: ClusterIP
  pdb:
    create: true
    minAvailable: 1

secondary:
  replicaCount: 2
  persistence:
    enabled: true
    size: 10Gi
    storageClass: standard
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  pdb:
    create: true
    minAvailable: 1

# Advanced metrics configuration
metrics:
  enabled: true
  image:
    registry: docker.io
    repository: prom/mysqld-exporter
    tag: "v0.15.1"
    pullPolicy: IfNotPresent
  
  resources:
    limits:
      cpu: 150m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi
  
  # Custom exporter arguments
  extraArgs:
    primary:
      - --collect.info_schema.innodb_metrics
      - --collect.info_schema.processlist
      - --collect.info_schema.query_response_time
    secondary:
      - --collect.info_schema.processlist
      - --collect.slave_status
  
  serviceMonitor:
    enabled: true
    namespace: monitoring  # Specify monitoring namespace
    interval: 15s
    scrapeTimeout: 10s
    labels:
      prometheus: kube-prometheus  # Match Prometheus operator selector
    selector: {}
    relabelings:
      - sourceLabels: [__meta_kubernetes_pod_name]
        targetLabel: pod
      - sourceLabels: [__meta_kubernetes_pod_node_name]
        targetLabel: node
    metricRelabelings:
      - sourceLabels: [__name__]
        regex: 'mysql_global_status_slow_queries'
        targetLabel: __tmp_slow_queries
        replacement: true
    honorLabels: true
    jobLabel: "mysql-monitoring"
  
  prometheusRule:
    enabled: true
    namespace: monitoring
    labels:
      prometheus: kube-prometheus
    groups:
      - name: mysql.rules
        interval: 30s
        rules:
          - alert: MySQLDown
            expr: mysql_up == 0
            for: 5m
            labels:
              severity: critical
            annotations:
              summary: "MySQL instance {{ $labels.instance }} is down"
              description: "MySQL has been down for more than 5 minutes."
          
          - alert: MySQLTooManyConnections
            expr: mysql_global_status_threads_connected > mysql_global_variables_max_connections * 0.8
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "MySQL too many connections on {{ $labels.instance }}"
              description: "More than 80% of MySQL connections are in use."
          
          - alert: MySQLSlowQueries
            expr: rate(mysql_global_status_slow_queries[5m]) > 0.05
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "MySQL slow queries on {{ $labels.instance }}"
              description: "MySQL is having {{ $value }} slow queries per second."

# Network Policy
networkPolicy:
  enabled: true
  allowExternal: true
  ingressRules:
    primaryAccessOnlyFrom:
      enabled: false

# Volume permissions for non-root containers
volumePermissions:
  enabled: true
```

### 4. Deploy MySQL with Advanced Configuration

```bash
# Deploy with advanced configuration
helm install my-mysql-prod ./mysql-server-helm \
  -f mysql-metrics-replication.yaml \
  --namespace production \
  --create-namespace

# Check deployment status
kubectl get all -n production -l app.kubernetes.io/instance=my-mysql-prod
```

## Verifying the Setup

### 1. Check Metrics Endpoint

```bash
# Port-forward to metrics port
kubectl port-forward svc/my-mysql-primary 9104:9104

# In another terminal, check metrics
curl http://localhost:9104/metrics | grep mysql_up

# Expected output:
# mysql_up 1
```

### 2. Verify ServiceMonitor is Discovered

```bash
# Check if ServiceMonitor is created
kubectl get servicemonitor -A | grep mysql

# Check Prometheus targets (port-forward to Prometheus)
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Open browser: http://localhost:9090/targets
# Look for mysql targets
```

### 3. Check Metrics in Prometheus

Access Prometheus UI and run queries:

```promql
# Check if MySQL is up
mysql_up

# Check connections
mysql_global_status_threads_connected

# Check queries per second
rate(mysql_global_status_queries[5m])

# Check slow queries
rate(mysql_global_status_slow_queries[5m])

# Replication lag (for replication mode)
mysql_slave_lag_seconds
```

## Grafana Dashboard Setup

### 1. Access Grafana

```bash
# Get Grafana password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d

# Port-forward Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Access: http://localhost:3000
# Login: admin / <password from above>
```

### 2. Import MySQL Dashboard

1. Go to Dashboards → Import
2. Enter Dashboard ID: `7362` (MySQL Overview)
3. Select Prometheus data source
4. Click Import

Alternative dashboards:
- `14057` - MySQL Exporter Quickstart
- `6239` - MySQL Performance
- `7991` - MySQL Replication

### 3. Create Custom Dashboard

Create a custom dashboard with key panels:

```json
{
  "dashboard": {
    "title": "MySQL Custom Metrics",
    "panels": [
      {
        "title": "MySQL Status",
        "targets": [
          {
            "expr": "mysql_up",
            "legendFormat": "{{ instance }}"
          }
        ]
      },
      {
        "title": "Connections",
        "targets": [
          {
            "expr": "mysql_global_status_threads_connected",
            "legendFormat": "Connected"
          },
          {
            "expr": "mysql_global_variables_max_connections",
            "legendFormat": "Max"
          }
        ]
      },
      {
        "title": "Queries Per Second",
        "targets": [
          {
            "expr": "rate(mysql_global_status_queries[5m])",
            "legendFormat": "QPS"
          }
        ]
      }
    ]
  }
}
```

## Troubleshooting

### Issue: ServiceMonitor Not Discovered

```bash
# Check if Prometheus Operator is configured to discover ServiceMonitors
kubectl get prometheus -n monitoring -o yaml | grep serviceMonitorSelector

# Solution: Update Prometheus to discover all ServiceMonitors
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### Issue: No Metrics Data

```bash
# Check if metrics container is running
kubectl logs my-mysql-primary-0 -c metrics

# Check service endpoints
kubectl get endpoints my-mysql-primary

# Test metrics directly
kubectl exec my-mysql-primary-0 -c metrics -- wget -qO- localhost:9104/metrics
```

### Issue: Permission Denied Errors

```bash
# Check mysqld-exporter logs
kubectl logs my-mysql-primary-0 -c metrics

# If permission errors, verify MySQL user has required permissions
kubectl exec -it my-mysql-primary-0 -- mysql -uroot -p -e "
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'root'@'localhost';
FLUSH PRIVILEGES;"
```

### Issue: High Cardinality Metrics

If experiencing performance issues with too many metrics:

```yaml
metrics:
  extraArgs:
    primary:
      - --no-collect.info_schema.tables
      - --no-collect.info_schema.innodb_metrics
```

## Monitoring Best Practices

### 1. Resource Allocation

```yaml
metrics:
  resources:
    requests:
      memory: "128Mi"  # Minimum for basic metrics
      cpu: "100m"
    limits:
      memory: "256Mi"  # Increase for large databases
      cpu: "200m"
```

### 2. Scrape Configuration

```yaml
metrics:
  serviceMonitor:
    interval: 30s  # Balance between data granularity and load
    scrapeTimeout: 10s  # Must be less than interval
```

### 3. Essential Alerts

Always configure these critical alerts:

```yaml
- MySQL instance down
- Connection pool exhaustion
- Replication lag (if using replication)
- Slow query rate increase
- Disk space usage
```

### 4. Security Considerations

```yaml
# Use network policies to restrict metrics access
networkPolicy:
  enabled: true
  allowExternal: false
  ingressRules:
    primaryAccessOnlyFrom:
      enabled: true
      namespaceSelector:
        matchLabels:
          name: monitoring
```

## Testing Metrics Load

```bash
# Generate load to see metrics change
kubectl run mysql-client --rm -it --image=mysql:8.0.36 --restart=Never -- \
  mysql -hmy-mysql-primary -uroot -pmysqlroot123 -e "
  CREATE DATABASE IF NOT EXISTS loadtest;
  USE loadtest;
  CREATE TABLE IF NOT EXISTS test (id INT PRIMARY KEY AUTO_INCREMENT, data VARCHAR(255));
  INSERT INTO test (data) SELECT CONCAT('data-', RAND()) FROM information_schema.tables LIMIT 1000;"

# Watch metrics change in Grafana
```

## Cleanup

```bash
# Remove MySQL installation
helm uninstall my-mysql
kubectl delete pvc -l app.kubernetes.io/instance=my-mysql

# Remove Prometheus stack (optional)
helm uninstall prometheus -n monitoring
kubectl delete namespace monitoring
```

## Next Steps

1. Configure long-term metrics storage (Thanos, Cortex)
2. Set up alerting channels (PagerDuty, Slack)
3. Implement custom business metrics
4. Configure metric retention policies
5. Set up metric aggregation rules for performance