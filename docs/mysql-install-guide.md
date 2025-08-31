# MySQL Helm Chart Installation Guide

## Quick Start

### Prerequisites

1. Kubernetes cluster (1.19+)
2. Helm 3.2.0+
3. kubectl configured to access your cluster

### Basic Installation

#### Standalone Mode (Default)

```bash
# Install with auto-generated passwords
helm install my-mysql ./mysql-server-helm

# Install with custom passwords
helm install my-mysql ./mysql-server-helm \
  --set auth.rootPassword=myRootPassword \
  --set auth.database=myapp \
  --set auth.username=myuser \
  --set auth.password=mypassword
```

#### Replication Mode

```bash
helm install my-mysql ./mysql-server-helm \
  --set architecture=replication \
  --set auth.rootPassword=myRootPassword \
  --set auth.replicationPassword=myReplPassword \
  --set secondary.replicaCount=1
```

## Validation Steps

### 1. Check Pod Status

```bash
# For standalone
kubectl get pods -l app.kubernetes.io/instance=my-mysql

# For replication
kubectl get pods -l app.kubernetes.io/instance=my-mysql
# Should see: my-mysql-primary-0, my-mysql-secondary-0, my-mysql-secondary-1
# Note: Secondary pods may take 2-5 minutes to initialize and reach Running state

# Wait for all pods to be ready (timeout after 5 minutes)
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=my-mysql --timeout=300s
```

### 2. Run Helm Test

```bash
helm test my-mysql
```

### 3. Connect to MySQL

```bash
# Get root password
export MYSQL_ROOT_PASSWORD=$(kubectl get secret my-mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d)

# Interactive connection (for manual testing)
kubectl run mysql-client --rm --tty -i --restart='Never' \
  --image mysql:9.4.0 \
  --env MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
  --command -- mysql -hmy-mysql-primary -uroot -p"$MYSQL_ROOT_PASSWORD"

# Non-interactive connection (for scripts/CI)
kubectl run mysql-client --rm --restart='Never' \
  --image mysql:9.4.0 \
  --env MYSQL_ROOT_PASSWORD="$MYSQL_ROOT_PASSWORD" \
  --command -- mysql -hmy-mysql-primary -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 1 as 'Connection Test';"
```

### 4. Verify Replication (if using replication mode)

```bash
# Check replication status on secondary
kubectl exec -it my-mysql-secondary-0 -- mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS"

# Non-interactive version for scripts
kubectl exec my-mysql-secondary-0 -- mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS"
```

## Common Installation Scenarios

### Production Installation

```bash
helm install production-mysql ./mysql-server-helm \
  --namespace production \
  --create-namespace \
  --set architecture=replication \
  --set auth.existingSecret=mysql-credentials \
  --set primary.persistence.size=100Gi \
  --set primary.persistence.storageClass=fast-ssd \
  --set primary.resources.requests.memory=4Gi \
  --set primary.resources.limits.memory=8Gi \
  --set secondary.replicaCount=3 \
  --set secondary.persistence.size=100Gi \
  --set secondary.resources.requests.memory=2Gi \
  --set secondary.resources.limits.memory=4Gi \
  --set metrics.enabled=true \
  --set metrics.serviceMonitor.enabled=true \
  --set networkPolicy.enabled=true \
  --set primary.pdb.create=true \
  --set secondary.pdb.create=true
```

### Development Installation

```bash
helm install dev-mysql ./mysql-server-helm \
  --namespace development \
  --create-namespace \
  --set architecture=standalone \
  --set auth.rootPassword=devpass \
  --set primary.persistence.size=10Gi \
  --set primary.resources.requests.memory=512Mi \
  --set primary.resources.limits.memory=1Gi
```

### Using Init Scripts

```bash
# Create a ConfigMap with init scripts
kubectl create configmap mysql-init-scripts \
  --from-file=init.sql=./my-init.sql \
  --from-file=setup.sh=./my-setup.sh

# Install with init scripts
helm install my-mysql ./mysql-server-helm \
  --set initdbScriptsConfigMap=mysql-init-scripts
```

### Using Existing PVC

```bash
helm install my-mysql ./mysql-server-helm \
  --set primary.persistence.existingClaim=my-existing-pvc
```

## Troubleshooting Installation

### Issue: Pods stuck in Pending state

**Cause**: PVC cannot be bound
**Solution**:
```bash
# Check PVC status
kubectl get pvc

# Check available storage classes
kubectl get storageclass

# Install with specific storage class
helm install my-mysql ./mysql-server-helm \
  --set primary.persistence.storageClass=standard
```

### Issue: Connection refused

**Cause**: MySQL not ready
**Solution**:
```bash
# Check pod logs
kubectl logs my-mysql-primary-0

# Check if MySQL is running
kubectl exec my-mysql-primary-0 -- mysqladmin ping -uroot -p"$MYSQL_ROOT_PASSWORD"
```

### Issue: Secondary pods stuck in init phase

**Cause**: Replication initialization takes time
**Solution**:
```bash
# Check secondary pod status (may show Init:0/1 for several minutes)
kubectl get pods -l app.kubernetes.io/component=secondary

# Check init container logs
kubectl logs my-mysql-secondary-0 -c clone-mysql

# Wait longer for initialization (up to 5 minutes)
kubectl wait --for=condition=Ready pod my-mysql-secondary-0 --timeout=300s
```

### Issue: Replication not working

**Cause**: Secondary cannot connect to primary or replication misconfiguration
**Solution**:
```bash
# Check secondary logs
kubectl logs my-mysql-secondary-0

# Verify primary is accessible from secondary
kubectl exec my-mysql-secondary-0 -- mysql -hmy-mysql-primary -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 1"

# Check replication status (using MySQL 9.4.0 syntax)
kubectl exec my-mysql-secondary-0 -- mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW REPLICA STATUS"

# Check primary binary log status
kubectl exec my-mysql-primary-0 -- mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW MASTER STATUS"
```

### Issue: Network connectivity between primary and secondary

**Cause**: Network policies or DNS issues
**Solution**:
```bash
# Test DNS resolution from secondary
kubectl exec my-mysql-secondary-0 -- nslookup my-mysql-primary

# Test port connectivity
kubectl exec my-mysql-secondary-0 -- nc -zv my-mysql-primary 3306

# Check network policies
kubectl get networkpolicy -o yaml
```

### Issue: Test fails

**Cause**: Various
**Solution**:
```bash
# Check test pod logs
kubectl logs my-mysql-test-connection

# Run test manually (non-interactive for scripts)
kubectl run test --rm --image=mysql:9.4.0 --restart=Never -- \
  mysql -hmy-mysql-primary.default.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT 1 as 'Manual Test';"

# Interactive test (for manual debugging)
kubectl run test --rm -it --image=mysql:9.4.0 --restart=Never -- \
  mysql -hmy-mysql-primary.default.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

## Uninstallation

```bash
# Uninstall the release
helm uninstall my-mysql

# Delete PVCs (data will be lost!)
kubectl delete pvc -l app.kubernetes.io/instance=my-mysql

# If using custom namespace
kubectl delete namespace <namespace-name>
```

## Upgrade from Previous Version

```bash
# Check current values
helm get values my-mysql > current-values.yaml

# Perform upgrade
helm upgrade my-mysql ./mysql-server-helm \
  -f current-values.yaml \
  --set image.tag=9.3.0

# For major version upgrades, backup first
kubectl exec my-mysql-primary-0 -- mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" --all-databases > backup.sql
```

## Validation Checklist

- [ ] All pods are Running
- [ ] PVCs are Bound
- [ ] Services are created
- [ ] Can connect to primary
- [ ] Can connect to secondary (if replication)
- [ ] Replication is working (if enabled)
- [ ] Metrics are exposed (if enabled)
- [ ] Network policies are applied (if enabled)
- [ ] Helm test passes

## Next Steps

1. Configure backups
2. Set up monitoring dashboards
3. Configure alerting rules
4. Plan disaster recovery
5. Document connection strings for applications