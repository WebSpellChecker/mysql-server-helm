# MySQL password change testing guide

This guide provides comprehensive manual testing procedures for validating MySQL password changes in both replication and standalone modes using the MySQL Helm chart.

## Table of contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Architecture modes](#architecture-modes)
- [Manual testing procedures](#manual-testing-procedures)
  - [Replication mode testing](#replication-mode-testing)
  - [Standalone mode testing](#standalone-mode-testing)
- [Troubleshooting](#troubleshooting)
- [Best practices](#best-practices)
- [Security considerations](#security-considerations)

## Overview

The MySQL chart includes a password update job feature that safely changes passwords during Helm upgrades. This guide demonstrates how to manually test this functionality to ensure it works correctly for both deployment architectures.

## Prerequisites

Before starting the testing procedures, ensure you have:

- Kubernetes cluster with `kubectl` access
- Helm 3.x installed and configured
- MySQL Helm chart available locally (in `./mysql-server-helm` directory)
- Basic understanding of MySQL replication concepts
- Administrative access to the Kubernetes namespace

## Architecture modes

The MySQL Helm chart supports two deployment architectures, each with different password requirements:

### Standalone mode

- **Primary pod**: Single MySQL instance (server-id: 1)
- **No replication**: Single database instance
- **Password requirements**: Only root password needed
- **Use case**: Development environments or single-instance deployments

### Replication mode

- **Primary pod**: Read-write MySQL instance (server-id: 1)
- **Secondary pod**: Read-only MySQL replica (server-id: 2)
- **Replication**: Data automatically syncs from primary to secondary
- **Password requirements**: Both root password and replication user password
- **Use case**: Production environments requiring high availability

## Manual testing procedures

This section provides step-by-step procedures for testing password changes in both replication and standalone modes.

### Replication mode testing

> **Note**: If `mysql-server-helm` has already been deployed, then start from step 3.

#### Step 1: Deploy initial replication setup
```bash
# Deploy with initial password
helm upgrade --install my-mysql ./mysql-server-helm \
  --set architecture=replication \
  --set auth.rootPassword=initialpass123 \
  --set auth.replicationPassword=replicapass123
```

#### Step 2: Wait and test initial connectivity
```bash
# Wait for both pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=mysql --timeout=120s

# Check pods status (should show primary-0 and secondary-0)
kubectl get pods -l app.kubernetes.io/name=mysql

# Test connection to primary
kubectl exec my-mysql-primary-0 -- mysql -uroot -pinitialpass123 \
  -e "SELECT 'Primary connection successful' as status, @@server_id as server_id;"

# Test connection to secondary  
kubectl exec my-mysql-secondary-0 -- mysql -uroot -pinitialpass123 \
  -e "SELECT 'Secondary connection successful' as status, @@server_id as server_id;"
```

**Expected results:**

- Both pods show `READY` status (1/1)
- Primary shows `server_id = 1`
- Secondary shows `server_id = 2`
- Both connections succeed with initial password

#### Step 3: Verify replication functionality
```bash
# Create test data on primary
kubectl exec my-mysql-primary-0 -- mysql -uroot -pinitialpass123 -e "
CREATE DATABASE IF NOT EXISTS replication_test; 
USE replication_test; 
CREATE TABLE IF NOT EXISTS test_data (
  id INT PRIMARY KEY, 
  message VARCHAR(100), 
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
); 
INSERT INTO test_data (id, message) VALUES (1, 'Initial replication test');"

# Wait for replication (typically 1-3 seconds)
sleep 3

# Verify data replicated to secondary
kubectl exec my-mysql-secondary-0 -- mysql -uroot -pinitialpass123 \
  -e "USE replication_test; SELECT * FROM test_data;"
```

**Expected results:**

- Data appears on secondary replica
- Timestamps should match between primary and secondary

#### Step 4: Execute password change
```bash
# Change password using password update job
helm upgrade --install my-mysql ./mysql-server-helm \
  --set architecture=replication \
  --set auth.rootPassword=newpass456 \
  --set auth.replicationPassword=replicapass123 \
  --set passwordUpdateJob.enabled=true \
  --set passwordUpdateJob.previousPasswords.rootPassword=initialpass123
```

#### Step 5: Monitor password update job

```bash
# Check job status
kubectl get jobs -l app.kubernetes.io/name=mysql

# View job logs (replace with actual job name)
kubectl logs job/$(kubectl get jobs -l app.kubernetes.io/name=mysql -o name | tail -1 | cut -d'/' -f2)
```

**Expected log contents:**
```
===== MySQL Password Update Job Started =====
✓ Primary is ready and accepting connections
✓ Connected successfully with previous password
Password update is required. Proceeding...
✓ Root passwords updated successfully.
✓ Privileges flushed with new password
✓ New password is active
✓ Replication password updated
===== Password Update Completed Successfully =====
```

#### Step 6: Test new password on both replicas
```bash
# Wait for pods to restart after password change
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=mysql --timeout=120s

# Test primary with new password
kubectl exec my-mysql-primary-0 -- mysql -uroot -pnewpass456 \
  -e "SELECT 'Primary with NEW password' as status, @@server_id as server_id;"

# Test secondary with new password
kubectl exec my-mysql-secondary-0 -- mysql -uroot -pnewpass456 \
  -e "SELECT 'Secondary with NEW password' as status, @@server_id as server_id;"
```

**Expected results:**

- Both connections succeed with new password
- Server IDs remain consistent (1 for primary, 2 for secondary)

#### Step 7: Verify security and replication
```bash
# Test that old password is rejected on both replicas
kubectl exec my-mysql-primary-0 -- mysql -uroot -pinitialpass123 \
  -e "SELECT 'This should fail';" 2>&1 || echo "✓ Old password rejected on primary"

kubectl exec my-mysql-secondary-0 -- mysql -uroot -pinitialpass123 \
  -e "SELECT 'This should fail';" 2>&1 || echo "✓ Old password rejected on secondary"

# Test replication still works after password change
kubectl exec my-mysql-primary-0 -- mysql -uroot -pnewpass456 -e "
USE replication_test; 
INSERT INTO test_data (id, message) VALUES (2, 'Post password-change replication test');"

sleep 3

kubectl exec my-mysql-secondary-0 -- mysql -uroot -pnewpass456 \
  -e "USE replication_test; SELECT * FROM test_data WHERE id = 2;"
```

**Expected results:**

- Old password returns "Access denied" error on both replicas
- New data replicates successfully from primary to secondary

### Standalone mode testing

#### Step 1: Deploy or switch to standalone mode
```bash
# Deploy or switch to standalone mode with password change
helm upgrade --install my-mysql ./mysql-server-helm \
  --set architecture=standalone \
  --set auth.rootPassword=standalonepass123 \
  --set passwordUpdateJob.enabled=true \
  --set passwordUpdateJob.previousPasswords.rootPassword=previouspass456
```

#### Step 2: Monitor password update job

```bash
# Check job status
kubectl get jobs -l app.kubernetes.io/name=mysql

# View job logs
kubectl logs job/$(kubectl get jobs -l app.kubernetes.io/name=mysql -o name | tail -1 | cut -d'/' -f2)
```

**Expected log contents (standalone mode):**
```
===== MySQL Password Update Job Started =====
✓ Primary is ready and accepting connections
✓ Connected successfully with previous password
✓ Root passwords updated successfully.
✓ Privileges flushed with new password
✓ New password is active
===== Password Update Completed Successfully =====
```

> **Note**: No replication user password update is performed in standalone mode.

#### Step 3: Test standalone mode after password change
```bash
# Wait for pod to restart
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=mysql,app.kubernetes.io/component=primary --timeout=120s

# Check only primary pod exists
kubectl get pods -l app.kubernetes.io/name=mysql

# Test connection with new password
kubectl exec my-mysql-primary-0 -- mysql -uroot -pstandalonepass123 \
  -e "SELECT 'Standalone with NEW password' as status, @@server_id as server_id;"

# Test old password rejection
kubectl exec my-mysql-primary-0 -- mysql -uroot -ppreviouspass456 \
  -e "SELECT 'This should fail';" 2>&1 || echo "✓ Old password rejected in standalone mode"

# Test database functionality
kubectl exec my-mysql-primary-0 -- mysql -uroot -pstandalonepass123 -e "
CREATE DATABASE IF NOT EXISTS standalone_test; 
USE standalone_test; 
CREATE TABLE IF NOT EXISTS test_data (
  id INT PRIMARY KEY, 
  message VARCHAR(100), 
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
); 
INSERT INTO test_data (id, message) VALUES (1, 'Standalone mode test after password change');
SELECT * FROM test_data;"
```

**Expected results:**

- Only one pod exists (`my-mysql-primary-0`)
- New password works correctly
- Old password is rejected
- Database operations function normally

## Troubleshooting

This section covers common issues that may occur during password change testing and their solutions.

### Common issues

#### Password update job hangs

**Symptoms:**

- Job shows "Running" status for extended period
- Job logs show "Waiting for MySQL primary to be ready..."

**Causes:**

- Primary pod not fully started
- Network connectivity issues
- Resource constraints

**Solutions:**
```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=mysql

# Check pod logs
kubectl logs my-mysql-primary-0

# Delete stuck job and retry
kubectl delete job $(kubectl get jobs -l app.kubernetes.io/name=mysql -o name | tail -1 | cut -d'/' -f2)
```

#### "Access denied" after password change

**Symptoms:**

- Cannot connect with new password
- Password update job completed successfully

**Causes:**

- Pod didn't restart properly
- Secret not updated
- Multiple password changes in sequence

**Solutions:**
```bash
# Verify secret contains new password
kubectl get secret my-mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d && echo

# Restart pods manually if needed
kubectl delete pod my-mysql-primary-0
kubectl delete pod my-mysql-secondary-0  # only if using replication

# Wait for pods to restart
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=mysql --timeout=120s
```

#### Replication broken after password change

**Symptoms:**

- Secondary shows replication errors
- Data not syncing between primary and secondary

**Causes:**

- Replication user password mismatch
- Network issues during password update

**Solutions:**
```bash
# Check replication status on secondary
kubectl exec my-mysql-secondary-0 -- mysql -uroot -p[PASSWORD] -e "SHOW REPLICA STATUS"

# Look for specific error messages in status output
# Common fixes: restart secondary pod or re-run password update job
```

#### Architecture switch issues

**Symptoms:**

- Expected pods not created when switching between standalone/replication
- Password doesn't change during architecture switch

**Causes:**

- StatefulSets from previous architecture still exist
- Password update job not enabled during architecture change

**Solutions:**
```bash
# Always use password update job when switching architectures with different passwords
helm upgrade --install my-mysql ./mysql-server-helm \
  --set architecture=[standalone|replication] \
  --set auth.rootPassword=newpassword \
  --set passwordUpdateJob.enabled=true \
  --set passwordUpdateJob.previousPasswords.rootPassword=oldpassword

# If persistent volumes have old data, you may need to delete PVCs
kubectl get pvc -l app.kubernetes.io/name=mysql
# kubectl delete pvc [pvc-name] # Use with caution - this deletes data!
```

### Verification commands

#### Quick health check
```bash
# Check all MySQL resources
kubectl get all -l app.kubernetes.io/name=mysql

# Check recent password update jobs
kubectl get jobs -l app.kubernetes.io/name=mysql --sort-by=.metadata.creationTimestamp

# Check current password in secret
kubectl get secret my-mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d && echo
```

#### Test script template
```bash
#!/bin/bash
# MySQL password change test script

set -e

CURRENT_PASSWORD="$1"
NEW_PASSWORD="$2"
ARCHITECTURE="${3:-replication}"  # replication or standalone

echo "Testing password change from $CURRENT_PASSWORD to $NEW_PASSWORD in $ARCHITECTURE mode"

# Test current password works
kubectl exec my-mysql-primary-0 -- mysql -uroot -p$CURRENT_PASSWORD -e "SELECT 'Current password works' as status;"

# Perform password change
helm upgrade --install my-mysql ./mysql-server-helm \
  --set architecture=$ARCHITECTURE \
  --set auth.rootPassword=$NEW_PASSWORD \
  --set auth.replicationPassword=replicapass123 \
  --set passwordUpdateJob.enabled=true \
  --set passwordUpdateJob.previousPasswords.rootPassword=$CURRENT_PASSWORD

# Wait and test new password
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=mysql --timeout=120s
kubectl exec my-mysql-primary-0 -- mysql -uroot -p$NEW_PASSWORD -e "SELECT 'New password works' as status;"

# Test old password rejected
kubectl exec my-mysql-primary-0 -- mysql -uroot -p$CURRENT_PASSWORD -e "SELECT 1;" 2>&1 || echo "✓ Old password correctly rejected"

echo "Password change test completed successfully!"
```

## Best practices

Follow these best practices to ensure successful and secure password changes:

1. **Always use the password update job** when changing passwords through Helm upgrades
2. **Test in non-production environments first** before applying to production
3. **Monitor job logs** during password changes to catch issues early
4. **Verify both old password rejection and new password acceptance** 
5. **Test replication functionality** after password changes in replication mode
6. **Keep backup of previous passwords** until changes are confirmed working
7. **Use strong passwords** and rotate them regularly
8. **Document password changes** and maintain change logs

## Security considerations

Important security aspects to consider during password changes:

- The password update job uses the previous password to authenticate and change to the new password
- During the update process, both old and new passwords may be temporarily valid
- Always verify that old passwords are rejected after the update completes
- Monitor MySQL logs for any unauthorized access attempts during password changes
- Use Kubernetes secrets properly - avoid exposing passwords in logs or environment variables

## Additional resources

This testing guide ensures reliable password changes across different MySQL deployment architectures. Regular testing of these procedures helps maintain security and operational reliability of MySQL deployments.

For additional help, refer to:

- MySQL chart documentation in `mysql-server-helm/docs/`
- Kubernetes troubleshooting guides  
- MySQL official documentation for replication troubleshooting