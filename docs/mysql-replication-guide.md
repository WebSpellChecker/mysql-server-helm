# MySQL replication setup guide

A comprehensive guide for deploying and managing MySQL primary-secondary replication using the MySQL Helm chart, including architecture overview, configuration parameters, troubleshooting procedures, and recovery scenarios based on extensive testing.

## Table of contents

- [Architecture overview](#architecture-overview)
- [Configuration parameters](#configuration-parameters)
- [Installation guide](#installation-guide)
- [Verification procedures](#verification-procedures)
- [Manual replication configuration](#manual-replication-configuration)
- [Troubleshooting scenarios](#troubleshooting-scenarios)
- [Recovery procedures](#recovery-procedures)
- [Monitoring and maintenance](#monitoring-and-maintenance)

## Architecture overview

### Primary-secondary replication

The MySQL Helm chart supports primary-secondary replication architecture with the following components:

```
┌─────────────────┐    ┌─────────────────┐
│   Primary Pod   │    │  Secondary Pod  │
│  (Read/Write)   │───▶│   (Read-Only)   │
│                 │    │                 │
│ my-mysql-       │    │ my-mysql-       │
│ primary-0       │    │ secondary-0     │
└─────────────────┘    └─────────────────┘
        │                       │
        ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│   Primary PVC   │    │  Secondary PVC  │
│                 │    │                 │
│ data-my-mysql-  │    │ data-my-mysql-  │
│ primary-0       │    │ secondary-0     │
└─────────────────┘    └─────────────────┘
```

### Services created

When deploying in replication mode, two services are automatically created:

- **Primary Service**: `RELEASE_NAME-primary.NAMESPACE.svc.cluster.local`
  - Purpose: Read-write operations
  - Port: 3306
  - Target: Primary pod only

- **Secondary Service**: `RELEASE_NAME-secondary.NAMESPACE.svc.cluster.local`
  - Purpose: Read-only operations
  - Port: 3306
  - Target: Secondary pods (load balanced)

### StatefulSet architecture

Both primary and secondary deployments use StatefulSets to ensure:
- Stable network identity
- Ordered pod creation and deletion
- Persistent storage attachment
- Predictable pod names for replication configuration

## Configuration parameters

### Essential replication parameters

| Parameter | Description | Default | Required for Replication |
|-----------|-------------|---------|-------------------------|
| `architecture` | MySQL architecture mode | `standalone` | Yes (set to `replication`) |
| `auth.rootPassword` | Root password for both pods | `""` | Yes |
| `auth.replicationPassword` | Password for replication user | `""` | Yes |
| `auth.replicationUser` | Username for replication | `replicator` | No |
| `secondary.replicaCount` | Number of secondary replicas | `1` | No |
| `volumePermissions.enabled` | Enable volume permission fixes | `false` | Recommended |

### Working installation command

Based on extensive testing, this command consistently works:

```bash
helm install my-mysql ./mysql-server-helm/mysql \
  --set architecture=replication \
  --set auth.rootPassword=testroot123 \
  --set auth.replicationPassword=testrepl123 \
  --set volumePermissions.enabled=true \
  --set secondary.replicaCount=1
```

### Advanced configuration

For production environments, consider these additional parameters:

```yaml
architecture: replication
auth:
  rootPassword: "your-secure-root-password"
  replicationPassword: "your-secure-replication-password"
  
primary:
  persistence:
    enabled: true
    size: 100Gi
    storageClass: "fast-ssd"
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  configuration: |
    [mysqld]
    log-bin=mysql-bin
    server-id=1
    gtid-mode=ON
    enforce-gtid-consistency=ON
    log-slave-updates=ON
    binlog-format=ROW
    max_connections=500
    innodb_buffer_pool_size=2G

secondary:
  replicaCount: 2
  persistence:
    enabled: true
    size: 100Gi
    storageClass: "fast-ssd"
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1000m"
  configuration: |
    [mysqld]
    server-id=2
    gtid-mode=ON
    enforce-gtid-consistency=ON
    log-slave-updates=ON
    read_only=ON
    super_read_only=ON
    max_connections=200
    innodb_buffer_pool_size=1G

volumePermissions:
  enabled: true

networkPolicy:
  enabled: true
  
metrics:
  enabled: true
  serviceMonitor:
    enabled: true
```

## Installation guide

### Prerequisites

1. Kubernetes cluster 1.19+
2. Helm 3.2.0+
3. kubectl configured to access your cluster
4. Persistent volume provisioner configured
5. Sufficient resources (minimum 2 CPU, 4Gi memory total)

### Step 1: Prepare the environment

```bash
# Create namespace (optional)
kubectl create namespace mysql-replication

# Verify storage classes
kubectl get storageclass
```

### Step 2: Install with replication

```bash
# Basic replication installation
helm install my-mysql ./mysql-server-helm/mysql \
  --namespace mysql-replication \
  --set architecture=replication \
  --set auth.rootPassword=testroot123 \
  --set auth.replicationPassword=testrepl123 \
  --set volumePermissions.enabled=true \
  --set secondary.replicaCount=1
```

### Step 3: Wait for deployment

```bash
# Watch pod creation
kubectl get pods -n mysql-replication -w

# Wait for all pods to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/instance=my-mysql \
  -n mysql-replication \
  --timeout=300s
```

## Verification procedures

### Basic health checks

```bash
# 1. Check pod status
kubectl get pods -l app.kubernetes.io/instance=my-mysql -n mysql-replication

# Expected output:
# NAME                     READY   STATUS    RESTARTS   AGE
# my-mysql-primary-0       1/1     Running   0          5m
# my-mysql-secondary-0     1/1     Running   0          5m

# 2. Check services
kubectl get svc -l app.kubernetes.io/instance=my-mysql -n mysql-replication

# Expected output:
# NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
# my-mysql-primary      ClusterIP   10.96.XX.XX     <none>        3306/TCP   5m
# my-mysql-secondary    ClusterIP   10.96.XX.XX     <none>        3306/TCP   5m

# 3. Check persistent volumes
kubectl get pvc -l app.kubernetes.io/instance=my-mysql -n mysql-replication
```

### Database connectivity tests

```bash
# Get passwords
export MYSQL_ROOT_PASSWORD=$(kubectl get secret my-mysql -n mysql-replication -o jsonpath="{.data.mysql-root-password}" | base64 -d)

# Test primary connection
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 'Primary connected successfully' as status;"

# Test secondary connection
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 'Secondary connected successfully' as status;"
```

### Replication status verification

```bash
# Check replication status on secondary
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS"

# or

kubectl exec my-mysql-secondary-0 -- mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS" 2>/dev/null | awk -F'\t' 'NR==1{for(i=1;i<=NF;i++){if($i=="Replica_IO_Running" || $i=="Replica_SQL_Running"){print "Field " i ": " $i; pos[i]=i}}} NR==2{for(i in pos){print "Value " i ": " $i}}'

# Expected output:
# Field 11: Replica_IO_Running
# Field 12: Replica_SQL_Running
# Value 11: Yes
# Value 12: Yes

# Key indicators for healthy replication:
# Slave_IO_Running: Yes
# Slave_SQL_Running: Yes
# Seconds_Behind_Master: 0 (or low number)
# Last_Error: (empty)
```

### Replication functionality test

```bash
# 1. Create test data on primary
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    CREATE DATABASE IF NOT EXISTS replication_test;
    USE replication_test;
    CREATE TABLE test_table (id INT AUTO_INCREMENT PRIMARY KEY, data VARCHAR(100), created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
    INSERT INTO test_table (data) VALUES ('Test data 1'), ('Test data 2'), ('Test data 3');
  "

# 2. Verify data on secondary (wait a few seconds)
sleep 5
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    USE replication_test;
    SELECT COUNT(*) as replicated_records FROM test_table;
    SELECT * FROM test_table;
  "

# 3. Clean up test data
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "DROP DATABASE replication_test;"
```

### Master status verification

```bash
# Check master status on primary
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW BINARY LOG STATUS"

# Verify replication user exists
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT User, Host FROM mysql.user WHERE User='replicator';"
```

## Manual replication configuration

When automatic replication setup fails, use this manual configuration process:

### Step 1: Get master status

```bash
# Connect to primary and get current master status
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW BINARY LOG STATUS"

# Note the File and Position values from output
# Example output:
# File: mysql-bin.000003
# Position: 857
```

### Step 2: Configure replication on secondary

```bash
# Stop any existing replication
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "STOP SLAVE; RESET SLAVE ALL;"

# Configure replication with master position (replace values from Step 1)
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    CHANGE MASTER TO 
      MASTER_HOST='my-mysql-primary.mysql-replication.svc.cluster.local',
      MASTER_PORT=3306,
      MASTER_USER='replicator',
      MASTER_PASSWORD='testrepl123',
      MASTER_LOG_FILE='mysql-bin.000003',
      MASTER_LOG_POS=857;
    START SLAVE;
  "
```

### Step 3: Verify manual configuration

```bash
# Check replication status
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS"

# Look for:
# Slave_IO_Running: Yes
# Slave_SQL_Running: Yes
# Seconds_Behind_Master: 0
# Last_Error: (should be empty)
```

### Alternative: GTID-based replication

If GTID is enabled (recommended for production):

```bash
# Configure GTID-based replication (easier maintenance)
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    STOP SLAVE;
    RESET SLAVE ALL;
    CHANGE MASTER TO 
      MASTER_HOST='my-mysql-primary.mysql-replication.svc.cluster.local',
      MASTER_PORT=3306,
      MASTER_USER='replicator',
      MASTER_PASSWORD='testrepl123',
      MASTER_AUTO_POSITION=1;
    START SLAVE;
  "
```

## Troubleshooting scenarios

Based on extensive testing, here are the most common issues and their solutions:

### 1. Helm lint parse errors

**Symptoms:**
- `helm lint` fails with parse errors
- Error: `parse error at (mysql/templates/secondary/statefulset.yaml:779): unexpected {{end}}`

**Root cause:**
Duplicate content and corrupted template structure in secondary StatefulSet template.

**Solution:**
```bash
# Check template structure
helm template my-mysql ./mysql-server-helm/mysql --debug

# Look for duplicate {{end}} statements or malformed template syntax
# Fix the template files by removing duplicate content
```

**Prevention:**
- Always run `helm lint` before installation
- Use `helm template --debug` to validate template structure

### 2. Missing replication ConfigMap

**Symptoms:**
- Secondary pods fail to start
- Error: `configmap "my-mysql-secondary-replication" not found`

**Root cause:**
Secondary replication ConfigMap is not being created by the chart.

**Diagnosis:**
```bash
# Check if ConfigMaps exist
kubectl get configmap -l app.kubernetes.io/instance=my-mysql -n mysql-replication

# Check template rendering
helm template my-mysql ./mysql-server-helm/mysql | grep -A 20 -B 5 "secondary.*configmap"
```

**Solution:**
Manually create the missing ConfigMap:
```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-mysql-secondary-replication
  namespace: mysql-replication
  labels:
    app.kubernetes.io/instance: my-mysql
    app.kubernetes.io/component: secondary
data:
  setup-replication.sh: |
    #!/bin/bash
    set -e
    echo "Waiting for primary to be ready..."
    until mysql -h my-mysql-primary.mysql-replication.svc.cluster.local -u root -p"\$MYSQL_ROOT_PASSWORD" -e "SELECT 1" > /dev/null 2>&1; do
      echo "Primary not ready, waiting..."
      sleep 5
    done
    echo "Primary is ready, configuring replication..."
    # Additional replication setup scripts here
EOF
```

### 3. Primary init script authentication issues

**Symptoms:**
- Primary pod logs show: `ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)`
- Pod keeps restarting during initialization

**Root cause:**
MySQL temporary server during initialization requires socket connection, not TCP.

**Diagnosis:**
```bash
# Check primary pod logs
kubectl logs -n mysql-replication my-mysql-primary-0 --previous

# Look for authentication errors during init
```

**Solution:**
Update init scripts to use socket connection:
```bash
# Example init script fix
kubectl patch configmap my-mysql-primary -n mysql-replication --patch '
data:
  init-replication.sh: |
    #!/bin/bash
    set -e
    # Use socket connection during init
    mysql --socket=/var/run/mysqld/mysqld.sock -e "
      CREATE USER IF NOT EXISTS '\''replicator'\''@'\''%'\'' IDENTIFIED BY '\''$MYSQL_REPLICATION_PASSWORD'\'';
      GRANT REPLICATION SLAVE ON *.* TO '\''replicator'\''@'\''%'\'';
      FLUSH PRIVILEGES;
    "
'
```

### 4. Secondary replication setup timing issues

**Symptoms:**
- Secondary pod starts successfully but replication is not configured
- `SHOW SLAVE STATUS` returns empty result
- No replication errors in logs

**Root cause:**
Init scripts fail due to timing issues or primary not being ready.

**Diagnosis:**
```bash
# Check secondary logs for init script execution
kubectl logs -n mysql-replication my-mysql-secondary-0 -c mysql

# Check if replication user exists on primary
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT User FROM mysql.user WHERE User='replicator';"
```

**Solution:**
Implement retry logic in init scripts:
```bash
# Create improved init script with retries
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-mysql-secondary-init
  namespace: mysql-replication
data:
  init-replication.sh: |
    #!/bin/bash
    set -e
    RETRY_COUNT=30
    RETRY_DELAY=10
    
    echo "Waiting for primary to be ready..."
    for i in \$(seq 1 \$RETRY_COUNT); do
      if mysql -h my-mysql-primary.mysql-replication.svc.cluster.local -u root -p"\$MYSQL_ROOT_PASSWORD" -e "SELECT 1" > /dev/null 2>&1; then
        echo "Primary is ready!"
        break
      fi
      if [ \$i -eq \$RETRY_COUNT ]; then
        echo "Primary not ready after \$((RETRY_COUNT * RETRY_DELAY)) seconds"
        exit 1
      fi
      echo "Attempt \$i/\$RETRY_COUNT failed, retrying in \$RETRY_DELAY seconds..."
      sleep \$RETRY_DELAY
    done
    
    # Configure replication
    mysql -e "
      CHANGE MASTER TO 
        MASTER_HOST='my-mysql-primary.mysql-replication.svc.cluster.local',
        MASTER_PORT=3306,
        MASTER_USER='replicator',
        MASTER_PASSWORD='\$MYSQL_REPLICATION_PASSWORD',
        MASTER_AUTO_POSITION=1;
      START SLAVE;
    "
EOF
```

### 5. Replication error recovery

**Symptoms:**
- `Slave_SQL_Running: No` with coordinator errors
- Error: `Coordinator stopped because there were error(s) in the worker(s)`
- Replication lag increasing

**Diagnosis:**
```bash
# Check detailed slave status
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS" | grep -E "Last_.*Error|Seconds_Behind_Master"

# Check for conflicting transactions
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS" | grep -E "Last_SQL_Errno|Last_SQL_Error"
```

**Solution:**
Reset and restart replication with current master position:
```bash
# Get current master status
MASTER_STATUS=$(kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW BINARY LOG STATUS")

echo "$MASTER_STATUS"

# Extract values and restart replication
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    STOP SLAVE;
    RESET SLAVE ALL;
    CHANGE MASTER TO 
      MASTER_HOST='my-mysql-primary.mysql-replication.svc.cluster.local',
      MASTER_PORT=3306,
      MASTER_USER='replicator',
      MASTER_PASSWORD='testrepl123',
      MASTER_AUTO_POSITION=1;
    START SLAVE;
  "

# Verify recovery
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS" | grep -E "Slave_.*_Running"
```

### 6. Pod/PVC deletion timing issues

**Symptoms:**
- Error: `Unable to attach or mount volumes: error processing PVC: PVC is being deleted`
- Pod stuck in `ContainerCreating` status
- Multiple pod recreation attempts fail

**Root cause:**
Pod is recreated while PVC is still being deleted, causing a race condition.

**Diagnosis:**
```bash
# Check PVC status
kubectl get pvc -n mysql-replication

# Check pod events
kubectl describe pod my-mysql-secondary-0 -n mysql-replication

# Look for volume attachment errors in events
kubectl get events -n mysql-replication --sort-by='.lastTimestamp' | grep -i volume
```

**Solution:**
Ensure proper deletion order:
```bash
# Method 1: Delete pod first, then wait for PVC deletion
kubectl delete pod my-mysql-secondary-0 -n mysql-replication
kubectl wait --for=delete pvc/data-my-mysql-secondary-0 -n mysql-replication --timeout=300s

# Method 2: Force delete stuck resources
kubectl delete pod my-mysql-secondary-0 -n mysql-replication --grace-period=0 --force
kubectl patch pvc data-my-mysql-secondary-0 -n mysql-replication -p '{"metadata":{"finalizers":null}}'

# Verify clean state before proceeding
kubectl get pods,pvc -n mysql-replication
```

**Prevention:**
```bash
# Scale down StatefulSet before maintenance
kubectl scale statefulset my-mysql-secondary -n mysql-replication --replicas=0
kubectl wait --for=delete pod/my-mysql-secondary-0 -n mysql-replication --timeout=300s

# Perform maintenance operations
# ...

# Scale back up
kubectl scale statefulset my-mysql-secondary -n mysql-replication --replicas=1
```

### 7. Network connectivity issues

**Symptoms:**
- Secondary cannot connect to primary
- DNS resolution failures
- Connection timeouts

**Diagnosis:**
```bash
# Test DNS resolution from secondary pod
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  nslookup my-mysql-primary.mysql-replication.svc.cluster.local

# Test network connectivity
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  telnet my-mysql-primary.mysql-replication.svc.cluster.local 3306

# Check service endpoints
kubectl get endpoints -n mysql-replication my-mysql-primary
```

**Solution:**
```bash
# Verify service labels and selectors
kubectl get svc my-mysql-primary -n mysql-replication -o yaml

# Check pod labels
kubectl get pod my-mysql-primary-0 -n mysql-replication --show-labels

# If labels don't match, patch the service
kubectl patch svc my-mysql-primary -n mysql-replication --patch '
spec:
  selector:
    app.kubernetes.io/instance: my-mysql
    app.kubernetes.io/component: primary
'
```

## Recovery procedures

### Complete secondary pod recreation

When secondary pod is corrupted or stuck:

```bash
# 1. Scale down secondary StatefulSet
kubectl scale statefulset my-mysql-secondary -n mysql-replication --replicas=0

# 2. Wait for pod deletion
kubectl wait --for=delete pod/my-mysql-secondary-0 -n mysql-replication --timeout=300s

# 3. Delete PVC to start fresh (data will be lost!)
kubectl delete pvc data-my-mysql-secondary-0 -n mysql-replication

# 4. Verify clean state
kubectl get pods,pvc -l app.kubernetes.io/instance=my-mysql -n mysql-replication

# 5. Scale back up
kubectl scale statefulset my-mysql-secondary -n mysql-replication --replicas=1

# 6. Wait for new pod to be ready
kubectl wait --for=condition=ready pod/my-mysql-secondary-0 -n mysql-replication --timeout=300s

# 7. Configure replication manually (see manual configuration section)
```

### Secondary PVC recreation

When secondary PVC is corrupted:

```bash
# 1. Stop secondary pod
kubectl delete pod my-mysql-secondary-0 -n mysql-replication

# 2. Wait for pod deletion
kubectl wait --for=delete pod/my-mysql-secondary-0 -n mysql-replication --timeout=120s

# 3. Delete corrupted PVC
kubectl delete pvc data-my-mysql-secondary-0 -n mysql-replication

# 4. Verify PVC is deleted
kubectl wait --for=delete pvc/data-my-mysql-secondary-0 -n mysql-replication --timeout=300s

# 5. Pod will be recreated automatically with new PVC
kubectl wait --for=condition=ready pod/my-mysql-secondary-0 -n mysql-replication --timeout=300s

# 6. Manual replication configuration required
```

### Replication lag recovery

When replication falls significantly behind:

```bash
# 1. Check current lag
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS" | grep Seconds_Behind_Master

# 2. If lag is excessive (>300 seconds), consider forced resync
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    STOP SLAVE;
    RESET SLAVE ALL;
    CHANGE MASTER TO 
      MASTER_HOST='my-mysql-primary.mysql-replication.svc.cluster.local',
      MASTER_PORT=3306,
      MASTER_USER='replicator',
      MASTER_PASSWORD='testrepl123',
      MASTER_AUTO_POSITION=1;
    START SLAVE;
  "

# 3. Monitor recovery progress
watch "kubectl exec -n mysql-replication my-mysql-secondary-0 -- mysql -uroot -p'$MYSQL_ROOT_PASSWORD' -e 'SHOW REPLICA STATUS' | grep -E 'Slave_.*_Running|Seconds_Behind_Master'"
```

### Connection issues recovery

When authentication or connection problems occur:

```bash
# 1. Verify replication user exists and has correct privileges
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    SELECT User, Host FROM mysql.user WHERE User='replicator';
    SHOW GRANTS FOR 'replicator'@'%';
  "

# 2. Recreate replication user if missing
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    DROP USER IF EXISTS 'replicator'@'%';
    CREATE USER 'replicator'@'%' IDENTIFIED BY 'testrepl123';
    GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
    FLUSH PRIVILEGES;
  "

# 3. Test connection from secondary
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -h my-mysql-primary.mysql-replication.svc.cluster.local -u replicator -p"testrepl123" -e "SELECT 1"

# 4. Reconfigure replication with correct credentials
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    STOP SLAVE;
    CHANGE MASTER TO MASTER_PASSWORD='testrepl123';
    START SLAVE;
  "
```

### Backup and restore method

For complete data recovery:

```bash
# 1. Create backup on primary
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" \
    --all-databases \
    --single-transaction \
    --source-data=2 \
    --flush-logs \
    --events \
    --routines \
    --triggers > /tmp/primary-backup.sql

# 2. Copy backup to secondary pod
kubectl cp /tmp/primary-backup.sql mysql-replication/my-mysql-secondary-0:/tmp/

# 3. Restore on secondary
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "STOP SLAVE; RESET SLAVE ALL;"

kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" < /tmp/primary-backup.sql

# 4. Extract master position from backup file
MASTER_INFO=$(kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  grep "CHANGE MASTER TO" /tmp/primary-backup.sql)
echo "Master info from backup: $MASTER_INFO"

# 5. Configure replication with extracted position
# (Use the CHANGE MASTER TO command from backup file)

# 6. Clean up
kubectl exec -n mysql-replication my-mysql-secondary-0 -- rm /tmp/primary-backup.sql
rm /tmp/primary-backup.sql
```

## Monitoring and maintenance

### Continuous monitoring commands

```bash
# Real-time replication status monitoring
watch -n 5 "kubectl exec -n mysql-replication my-mysql-secondary-0 -- mysql -uroot -p'$MYSQL_ROOT_PASSWORD' -e 'SHOW REPLICA STATUS' | grep -E 'Slave_IO_State|Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master|Last_Error'"

# Monitor both primary and secondary status
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW BINARY LOG STATUS"

kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS"
```

### Health indicators

**Healthy replication indicators:**
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Seconds_Behind_Master: 0` (or consistently low)
- `Last_Error:` (empty)
- `Last_IO_Error:` (empty)
- `Last_SQL_Error:` (empty)

**Warning indicators:**
- `Seconds_Behind_Master: 30-300` (moderate lag)
- Occasional connection errors (recoverable)
- High but stable replication lag

**Critical indicators:**
- `Slave_IO_Running: No`
- `Slave_SQL_Running: No`
- `Seconds_Behind_Master: >300` (severe lag)
- Persistent errors in `Last_Error` fields

### Regular maintenance tasks

```bash
# Weekly replication health check
kubectl exec -n mysql-replication my-mysql-secondary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "
    SELECT 
      CASE 
        WHEN Slave_IO_Running = 'Yes' AND Slave_SQL_Running = 'Yes' AND Seconds_Behind_Master < 30 
        THEN 'HEALTHY'
        WHEN Slave_IO_Running = 'Yes' AND Slave_SQL_Running = 'Yes' AND Seconds_Behind_Master < 300 
        THEN 'WARNING'
        ELSE 'CRITICAL'
      END as replication_status,
      Slave_IO_Running,
      Slave_SQL_Running,
      Seconds_Behind_Master,
      Last_Error
    FROM 
      (SELECT * FROM INFORMATION_SCHEMA.REPLICA_HOST_STATUS 
       UNION ALL 
       SELECT NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL,NULL) t 
    LIMIT 1;
  " 2>/dev/null || \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS" | grep -E "Slave_IO_Running|Slave_SQL_Running|Seconds_Behind_Master|Last_Error"

# Monthly performance review
kubectl top pods -n mysql-replication
kubectl exec -n mysql-replication my-mysql-primary-0 -- \
  mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW GLOBAL STATUS LIKE 'Binlog_%'"
```

### Alerting rules

Consider setting up monitoring alerts for:

1. **Replication failure**: `Slave_IO_Running = No OR Slave_SQL_Running = No`
2. **High replication lag**: `Seconds_Behind_Master > 300`
3. **Pod restart frequency**: More than 3 restarts in 1 hour
4. **PVC space usage**: >80% full
5. **Connection errors**: Persistent authentication failures

This comprehensive guide covers all the scenarios encountered during testing and provides solutions for successful MySQL replication deployment and management.