# MySQL Helm chart

The WProofreader stack uses this chart for its MySQL server.
To install the full stack (MySQL, WProofreader Server, and Admin-panel) step by step
with Helm commands, follow the [Kubernetes installation guide](https://docs.wproofreader.com/deployment/installation/kubernetes).
To install the same stack with Argo CD or Flux, use the [WProofreader GitOps examples](https://github.com/WebSpellChecker/wproofreader-gitops).

## Table of contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Installation](#installation)
4. [Configuration reference](#configuration-reference)
5. [Networking and services](#networking-and-services)
6. [Security](#security)
7. [Monitoring and observability](#monitoring-and-observability)
8. [Operations and maintenance](#operations-and-maintenance)
9. [Troubleshooting](#troubleshooting)
10. [Migration guide](#migration-guide)
11. [API reference](#api-reference)
12. [Examples](#examples)

---

## Overview

### Chart information

- **Chart name**: mysql
- **Chart version**: 1.0.0
- **App version**: 9.4.0
- **Maintainer**: Infrastructure Team (support@webspellchecker.net)
- **Repository**: [GitHub](https://github.com/WebSpellChecker/mysql-server-helm)

### Key features

| Feature | Description | Status |
|---------|-------------|--------|
| **Official MySQL images** | Uses official Docker Hub MySQL images | ✅ Stable |
| **Architecture modes** | Standalone and Primary-Secondary replication | ✅ Stable |
| **GTID replication** | Global Transaction ID based replication | ✅ Stable |
| **StatefulSet deployment** | Stable network identity and persistent storage | ✅ Stable |
| **Prometheus monitoring** | Built-in metrics exporter with ServiceMonitor | ✅ Stable |
| **Network policies** | Traffic isolation and security | ✅ Stable |
| **RBAC** | Role-based access control | ✅ Stable |
| **Init scripts** | Database initialization support | ✅ Stable |
| **Volume permissions** | Automatic permission fixes for non-root | ✅ Stable |
| **Pod disruption budgets** | High availability safeguards | ✅ Stable |
| **Password update job** | Safe password rotation during upgrades | ✅ Stable |

### Compatibility matrix

| Component           | Minimum version | Recommended version | Maximum tested |
|---------------------|-----------------|---------------------|----------------|
| Kubernetes          | 1.25            | 1.34+               | 1.35           |
| Helm                | v3.2.0          | v3.13+              | v3.15.1        |
| MySQL               | v9.0.0          | v9.4.0              | v9.4.0         |
| Prometheus operator | v0.80.0         | v0.84.0+            | v0.84.1        |

---

## Architecture

### Deployment architectures

#### Standalone mode

```yaml
architecture: standalone
```

```
┌─────────────────────────┐
│    MySQL Primary Pod    │
│      (Read/Write)       │
│                         │
│  ┌─────────────────┐    │
│  │  MySQL Server   │    │
│  └─────────────────┘    │
│  ┌─────────────────┐    │
│  │ Metrics Exporter│    │
│  └─────────────────┘    │
└─────────────────────────┘
           │
           ▼
      ┌──────────┐
      │   PVC    │
      └──────────┘
```

#### Replication mode

```yaml
architecture: replication
```

```
     ┌─────────────────────────┐      ┌─────────────────────────┐
     │    MySQL Primary Pod    │      │   MySQL Secondary Pod   │
     │      (Read/Write)       │─────▶│       (Read-Only)       │
     │                         │      │                         │
     │  ┌─────────────────┐    │      │  ┌─────────────────┐    │
     │  │  MySQL Server   │    │      │  │  MySQL Server   │    │
     │  └─────────────────┘    │      │  └─────────────────┘    │
     │  ┌─────────────────┐    │      │  ┌─────────────────┐    │
     │  │ Metrics Exporter│    │      │  │ Metrics Exporter│    │
     │  └─────────────────┘    │      │  └─────────────────┘    │
     └─────────────────────────┘      └─────────────────────────┘
                │                                 │
                ▼                                 ▼
           ┌──────────┐                      ┌──────────┐
           │   PVC    │                      │   PVC    │
           └──────────┘                      └──────────┘
```

### Component structure

```
mysql-server-helm/
├── mysql/                           # The chart
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl               # Template helper functions
│       ├── NOTES.txt                  # Post-installation notes
│       ├── secrets.yaml               # Password management
│       ├── serviceaccount.yaml        # Service account
│       ├── role.yaml                  # RBAC role
│       ├── rolebinding.yaml           # RBAC binding
│       ├── networkpolicy.yaml         # Network policies
│       ├── poddisruptionbudget.yaml   # PDB configuration
│       ├── servicemonitor.yaml        # Prometheus monitoring
│       ├── prometheusrule.yaml        # Alert rules
│       ├── primary/
│       │   ├── statefulset.yaml       # Primary MySQL StatefulSet
│       │   ├── svc.yaml               # Primary services
│       │   └── configmap.yaml         # Primary configuration
│       ├── secondary/
│       │   ├── statefulset.yaml       # Secondary MySQL StatefulSet
│       │   ├── svc.yaml               # Secondary services
│       │   └── configmap.yaml         # Secondary configuration
│       ├── update-password/
│       │   ├── job.yaml               # Password update Job
│       │   ├── new-secret.yaml
│       │   └── previous-secret.yaml
│       └── tests/
│           └── test-connection.yaml    # Helm test
├── docs/                            # Guides
├── scripts/                         # Validation scripts
└── README.md
```

---

## Installation

### Prerequisites

#### System requirements

| Resource | Minimum       | Recommended   | Notes                             |
|----------|---------------|---------------|-----------------------------------|
| CPU      | 250m per pod  | 1000m per pod | Increase for production workloads |
| Memory   | 256Mi per pod | 1Gi per pod   | Based on buffer pool size         |
| Storage  | 1Gi           | 10Gi+         | Depends on data size              |
| Network  | 1Gbps         | 10Gbps        | For replication traffic           |


### Quick start
> **📖 Complete guide available**: For detailed Chart setup, advanced configurations, and troubleshooting, see [docs/mysql-install-guide.md](docs/mysql-install-guide.md)

#### Minimal installation

```bash
# Standalone mode with defaults
helm install my-mysql ./mysql-server-helm/mysql

# Get root password
kubectl get secret my-mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d
```

#### Production installation
> **📖 Complete replication guide**: For detailed replication Chart setup and troubleshooting, see [docs/mysql-replication-guide.md](docs/mysql-replication-guide.md)

```bash
# Create namespace
kubectl create namespace production

# Create secret with passwords
kubectl create secret generic mysql-prod-credentials -n production \
  --from-literal=mysql-root-password="$(openssl rand -base64 32)" \
  --from-literal=mysql-replication-password="$(openssl rand -base64 32)"

# Create values file
cat > production-values.yaml <<EOF
architecture: replication
auth:
  existingSecret: mysql-prod-credentials
  database: myapp
  
primary:
  persistence:
    size: 50Gi
    storageClass: standard
  resources:
    requests:
      memory: 2Gi
      cpu: 1000m
    limits:
      memory: 4Gi
      cpu: 2000m
  
secondary:
  replicaCount: 2
  persistence:
    size: 50Gi
    storageClass: standard
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 2Gi
      cpu: 1000m

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

volumePermissions:
  enabled: true
EOF

# Install
helm install mysql-prod ./mysql-server-helm/mysql \
  --namespace production \
  -f production-values.yaml
```

### Verification

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/instance=mysql-prod -n production

# Verify all pods are ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=mysql-prod -n production --timeout=300s

# Run Helm test
helm test mysql-prod -n production

# Get root password for verification
export MYSQL_ROOT_PASSWORD=$(kubectl get secret mysql-prod -n production -o jsonpath="{.data.mysql-root-password}" | base64 -d)

# Test direct pod connections
kubectl exec mysql-prod-primary-0 -n production -- \
  mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT VERSION();"

# Test service connections (primary)
kubectl run mysql-client --image=mysql:9.4.0 --rm -it --restart=Never -n production -- \
  mysql -h mysql-prod-primary -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 'Primary connection successful';"

# Check replication status (MySQL 9.4.0 compatible)
kubectl exec mysql-prod-secondary-0 -n production -- \
  mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "SHOW REPLICA STATUS"

# Alternative replication status check
kubectl exec mysql-prod-secondary-0 -n production -- \
  mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT * FROM performance_schema.replication_connection_status"

# Verify GTID replication is working
kubectl exec mysql-prod-secondary-0 -n production -- \
  mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "SHOW VARIABLES LIKE 'gtid_mode';"

# Test read from secondary (if using replication)
kubectl run mysql-client-read --image=mysql:9.4.0 --rm -it --restart=Never -n production -- \
  mysql -h mysql-prod-secondary -uroot -p${MYSQL_ROOT_PASSWORD} -e "SELECT 'Secondary read successful';"
```

## Configuration reference

### Global parameters

| Parameter                  | Description                         | Default         | Required |
|----------------------------|-------------------------------------|-----------------|----------|
| `global.imageRegistry`     | Global Docker image registry        | `""`            | No       |
| `global.imagePullSecrets`  | Global Docker registry secret names | `[]`            | No       |
| `global.storageClass`      | Global storage class for PVCs       | `""`            | No       |

### Architecture configuration

| Parameter     | Description             | Default      | Options                     |
|---------------|-------------------------|--------------|----------------------------|
| `architecture`| MySQL architecture mode | `standalone` | `standalone`, `replication`|

### Image configuration

| Parameter           | Description            | Default       |
|---------------------|------------------------|---------------|
| `image.registry`    | MySQL image registry   | `docker.io`   |
| `image.repository`  | MySQL image repository | `mysql`       |
| `image.tag`         | MySQL image tag        | `9.4.0`       |
| `image.pullPolicy`  | Image pull policy      | `IfNotPresent`|
| `image.pullSecrets` | Image pull secrets     | `[]`          |

### Authentication

| Parameter                    | Description              | Default               | Security notes                        |
|------------------------------|--------------------------|----------------------|---------------------------------------|
| `auth.rootPassword`          | MySQL root password      | `""` (auto-generated) | Use strong password or existing secret|
| `auth.database`              | Database to create       | `mydb`               | -                                     |
| `auth.username`              | Database user            | `""`                 | Optional                              |
| `auth.password`              | User password            | `""` (auto-generated) | Use strong password                   |
| `auth.replicationUser`       | Replication username     | `replicator`         | Only for replication mode             |
| `auth.replicationPassword`   | Replication password     | `""` (auto-generated) | Use strong password                   |
| `auth.existingSecret`        | Use existing secret      | `""`                 | Recommended for production            |
| `auth.usePasswordFiles`      | Mount passwords as files | `false`              | Currently not fully supported         |

### Password update job

| Parameter                                                  | Description                                | Default         | Notes                           |
|------------------------------------------------------------|--------------------------------------------|-----------------|---------------------------------|
| `passwordUpdateJob.enabled`                                | Enable password update job                 | `false`         | For safe password rotation      |
| `passwordUpdateJob.previousPasswords.rootPassword`        | Previous root password                     | `""`            | Required for validation         |
| `passwordUpdateJob.previousPasswords.password`            | Previous user password                     | `""`            | Required if user exists         |
| `passwordUpdateJob.previousPasswords.replicationPassword` | Previous replication password              | `""`            | Required for replication        |
| `passwordUpdateJob.previousPasswords.existingSecret`      | Use existing secret for previous passwords | `""`            | Alternative to inline passwords |
| `passwordUpdateJob.resources`                              | Job resource limits                        | See values.yaml | CPU and memory constraints      |

### Primary configuration

<details>
<summary>Click to expand primary configuration parameters</summary>

| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.name`  | Primary instance name | `primary` |
| `primary.configuration` | MySQL configuration (my.cnf) | See values.yaml |
| `primary.existingConfigmap` | Use existing ConfigMap | `""` |
| `primary.containerPorts.mysql` | MySQL port | `3306` |
| `primary.updateStrategy.type` | Update strategy | `RollingUpdate` |
| `primary.podManagementPolicy` | Pod management policy | `OrderedReady` |
| `primary.podAnnotations` | Pod annotations | `{}` |
| `primary.podLabels` | Pod labels | `{}` |
| `primary.podSecurityContext.enabled` | Enable pod security context | `true` |
| `primary.podSecurityContext.fsGroup` | Pod fsGroup | `999` |
| `primary.podSecurityContext.runAsUser` | Pod runAsUser | `999` |
| `primary.containerSecurityContext.enabled` | Enable container security context | `true` |
| `primary.containerSecurityContext.runAsUser` | Container runAsUser | `999` |
| `primary.containerSecurityContext.runAsNonRoot` | Run as non-root | `true` |
| `primary.containerSecurityContext.allowPrivilegeEscalation` | Allow privilege escalation | `false` |
| `primary.containerSecurityContext.capabilities.drop` | Dropped capabilities | `["ALL"]` |
| `primary.resources.limits.cpu` | CPU limit | `1000m` |
| `primary.resources.limits.memory` | Memory limit | `1Gi` |
| `primary.resources.requests.cpu` | CPU request | `250m` |
| `primary.resources.requests.memory` | Memory request | `256Mi` |

**Probe Configuration**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.livenessProbe.enabled` | Enable liveness probe | `true` |
| `primary.livenessProbe.initialDelaySeconds` | Initial delay for liveness probe | `30` |
| `primary.livenessProbe.periodSeconds` | Period for liveness probe | `10` |
| `primary.livenessProbe.timeoutSeconds` | Timeout for liveness probe | `5` |
| `primary.livenessProbe.successThreshold` | Success threshold for liveness probe | `1` |
| `primary.livenessProbe.failureThreshold` | Failure threshold for liveness probe | `3` |
| `primary.readinessProbe.enabled` | Enable readiness probe | `true` |
| `primary.readinessProbe.initialDelaySeconds` | Initial delay for readiness probe | `5` |
| `primary.readinessProbe.periodSeconds` | Period for readiness probe | `10` |
| `primary.readinessProbe.timeoutSeconds` | Timeout for readiness probe | `5` |
| `primary.readinessProbe.successThreshold` | Success threshold for readiness probe | `1` |
| `primary.readinessProbe.failureThreshold` | Failure threshold for readiness probe | `3` |
| `primary.startupProbe.enabled` | Enable startup probe | `true` |
| `primary.startupProbe.initialDelaySeconds` | Initial delay for startup probe | `0` |
| `primary.startupProbe.periodSeconds` | Period for startup probe | `10` |
| `primary.startupProbe.timeoutSeconds` | Timeout for startup probe | `5` |
| `primary.startupProbe.successThreshold` | Success threshold for startup probe | `1` |
| `primary.startupProbe.failureThreshold` | Failure threshold for startup probe | `30` |

**Persistence Configuration**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.persistence.enabled` | Enable persistence | `true` |
| `primary.persistence.storageClass` | Storage class | `""` |
| `primary.persistence.accessModes` | Access modes | `["ReadWriteOnce"]` |
| `primary.persistence.size` | PVC size | `8Gi` |
| `primary.persistence.annotations` | PVC annotations | `{}` |
| `primary.persistence.selector` | PVC selector | `{}` |
| `primary.persistence.existingClaim` | Use existing PVC | `""` |

**Service Configuration**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.service.type` | Service type | `ClusterIP` |
| `primary.service.ports.mysql` | MySQL service port | `3306` |
| `primary.service.nodePorts.mysql` | NodePort (if applicable) | `""` |
| `primary.service.clusterIP` | Cluster IP | `""` |
| `primary.service.loadBalancerIP` | LoadBalancer IP | `""` |
| `primary.service.loadBalancerSourceRanges` | LoadBalancer source ranges | `[]` |
| `primary.service.annotations` | Service annotations | `{}` |
| `primary.service.sessionAffinity` | Session affinity | `None` |

**Pod Disruption Budget**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.pdb.create` | Create PodDisruptionBudget | `false` |
| `primary.pdb.minAvailable` | Minimum available pods | `1` |
| `primary.pdb.maxUnavailable` | Maximum unavailable pods | `""` |

**Environment Variables and Commands**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.extraEnvVars` | Extra environment variables as array | `[]` |
| `primary.extraEnvVarsCM` | ConfigMap containing extra environment variables | `""` |
| `primary.extraEnvVarsSecret` | Secret containing extra environment variables | `""` |
| `primary.command` | Override default container command | `[]` |
| `primary.args` | Override default container args | `[]` |

**Lifecycle and Hooks**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.lifecycleHooks` | Lifecycle hooks for the container | `{}` |

**Additional Volumes and Containers**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.extraVolumes` | Extra volumes for the pod | `[]` |
| `primary.extraVolumeMounts` | Extra volume mounts for the container | `[]` |
| `primary.initContainers` | Init containers for the pod | `[]` |
| `primary.sidecars` | Sidecar containers for the pod | `[]` |

**Node Selection and Scheduling**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `primary.nodeSelector` | Node labels for pod assignment | `{}` |
| `primary.tolerations` | Tolerations for pod assignment | `[]` |
| `primary.affinity` | Affinity settings for pod assignment | `{}` |
| `primary.topologySpreadConstraints` | Topology spread constraints | `[]` |
| `primary.priorityClassName` | Priority class name for the pod | `""` |
| `primary.schedulerName` | Name of the k8s scheduler | `""` |
| `primary.terminationGracePeriodSeconds` | Grace period for pod termination | `30` |
| `primary.hostAliases` | Host aliases for the pod | `[]` |

</details>

### Secondary configuration

<details>
<summary>Click to expand secondary configuration parameters</summary>

| Parameter         | Description                 | Default     |
|-------------------|-----------------------------|-------------|
| `secondary.name`  | Secondary instance name     | `secondary` |
| `secondary.replicaCount` | Number of secondary replicas | `1` |
| `secondary.configuration` | MySQL configuration (my.cnf) | See values.yaml |
| `secondary.existingConfigmap` | Use existing ConfigMap | `""` |
| `secondary.containerPorts.mysql` | MySQL port | `3306` |
| `secondary.updateStrategy.type` | Update strategy | `RollingUpdate` |
| `secondary.podManagementPolicy` | Pod management policy | `OrderedReady` |
| `secondary.podAnnotations` | Pod annotations | `{}` |
| `secondary.podLabels` | Pod labels | `{}` |
| `secondary.podSecurityContext.enabled` | Enable pod security context | `true` |
| `secondary.podSecurityContext.fsGroup` | Pod fsGroup | `999` |
| `secondary.podSecurityContext.runAsUser` | Pod runAsUser | `999` |
| `secondary.containerSecurityContext.enabled` | Enable container security context | `true` |
| `secondary.containerSecurityContext.runAsUser` | Container runAsUser | `999` |
| `secondary.containerSecurityContext.runAsNonRoot` | Run as non-root | `true` |
| `secondary.containerSecurityContext.allowPrivilegeEscalation` | Allow privilege escalation | `false` |
| `secondary.containerSecurityContext.capabilities.drop` | Dropped capabilities | `["ALL"]` |
| `secondary.resources.limits.cpu` | CPU limit | `1000m` |
| `secondary.resources.limits.memory` | Memory limit | `1Gi` |
| `secondary.resources.requests.cpu` | CPU request | `250m` |
| `secondary.resources.requests.memory` | Memory request | `256Mi` |

**Probe Configuration**
| Parameter         | Description                 | Default     |
|-------------------|-----------------------------|-------------|
| `secondary.livenessProbe.enabled` | Enable liveness probe | `true` |
| `secondary.livenessProbe.initialDelaySeconds` | Initial delay for liveness probe | `30` |
| `secondary.livenessProbe.periodSeconds` | Period for liveness probe | `10` |
| `secondary.livenessProbe.timeoutSeconds` | Timeout for liveness probe | `5` |
| `secondary.livenessProbe.successThreshold` | Success threshold for liveness probe | `1` |
| `secondary.livenessProbe.failureThreshold` | Failure threshold for liveness probe | `3` |
| `secondary.readinessProbe.enabled` | Enable readiness probe | `true` |
| `secondary.readinessProbe.initialDelaySeconds` | Initial delay for readiness probe | `5` |
| `secondary.readinessProbe.periodSeconds` | Period for readiness probe | `10` |
| `secondary.readinessProbe.timeoutSeconds` | Timeout for readiness probe | `5` |
| `secondary.readinessProbe.successThreshold` | Success threshold for readiness probe | `1` |
| `secondary.readinessProbe.failureThreshold` | Failure threshold for readiness probe | `3` |
| `secondary.startupProbe.enabled` | Enable startup probe | `true` |
| `secondary.startupProbe.initialDelaySeconds` | Initial delay for startup probe | `0` |
| `secondary.startupProbe.periodSeconds` | Period for startup probe | `10` |
| `secondary.startupProbe.timeoutSeconds` | Timeout for startup probe | `5` |
| `secondary.startupProbe.successThreshold` | Success threshold for startup probe | `1` |
| `secondary.startupProbe.failureThreshold` | Failure threshold for startup probe | `30` |

**Persistence Configuration**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `secondary.persistence.enabled` | Enable persistence | `true` |
| `secondary.persistence.storageClass` | Storage class | `""` |
| `secondary.persistence.accessModes` | Access modes | `["ReadWriteOnce"]` |
| `secondary.persistence.size` | PVC size | `8Gi` |
| `secondary.persistence.annotations` | PVC annotations | `{}` |
| `secondary.persistence.selector` | PVC selector | `{}` |

**Service Configuration**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `secondary.service.type` | Service type | `ClusterIP` |
| `secondary.service.ports.mysql` | MySQL service port | `3306` |
| `secondary.service.nodePorts.mysql` | NodePort (if applicable) | `""` |
| `secondary.service.clusterIP` | Cluster IP | `""` |
| `secondary.service.loadBalancerIP` | LoadBalancer IP | `""` |
| `secondary.service.loadBalancerSourceRanges` | LoadBalancer source ranges | `[]` |
| `secondary.service.annotations` | Service annotations | `{}` |
| `secondary.service.sessionAffinity` | Session affinity | `None` |

**Pod Disruption Budget**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `secondary.pdb.create` | Create PodDisruptionBudget | `false` |
| `secondary.pdb.minAvailable` | Minimum available pods | `1` |
| `secondary.pdb.maxUnavailable` | Maximum unavailable pods | `""` |

**Environment Variables and Commands**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `secondary.extraEnvVars` | Extra environment variables as array | `[]` |
| `secondary.extraEnvVarsCM` | ConfigMap containing extra environment variables | `""` |
| `secondary.extraEnvVarsSecret` | Secret containing extra environment variables | `""` |
| `secondary.command` | Override default container command | `[]` |
| `secondary.args` | Override default container args | `[]` |

**Lifecycle and Hooks**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `secondary.lifecycleHooks` | Lifecycle hooks for the container | `{}` |

**Additional Volumes and Containers**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `secondary.extraVolumes` | Extra volumes for the pod | `[]` |
| `secondary.extraVolumeMounts` | Extra volume mounts for the container | `[]` |
| `secondary.initContainers` | Init containers for the pod | `[]` |
| `secondary.sidecars` | Sidecar containers for the pod | `[]` |

**Node Selection and Scheduling**
| Parameter       | Description           | Default   |
|-----------------|----------------------|-----------|
| `secondary.nodeSelector` | Node labels for pod assignment | `{}` |
| `secondary.tolerations` | Tolerations for pod assignment | `[]` |
| `secondary.affinity` | Affinity settings for pod assignment | `{}` |
| `secondary.topologySpreadConstraints` | Topology spread constraints | `[]` |
| `secondary.priorityClassName` | Priority class name for the pod | `""` |
| `secondary.schedulerName` | Name of the k8s scheduler | `""` |
| `secondary.terminationGracePeriodSeconds` | Grace period for pod termination | `30` |
| `secondary.hostAliases` | Host aliases for the pod | `[]` |

</details>

### Monitoring configuration

| Parameter         | Description             | Default |
|-------------------|-------------------------|---------|
| `metrics.enabled` | Enable metrics exporter | `false` |
| `metrics.image.registry` | Exporter image registry | `docker.io` |
| `metrics.image.repository` | Exporter image repository | `prom/mysqld-exporter` |
| `metrics.image.tag` | Exporter image tag | `v0.17.2` |
| `metrics.image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `metrics.resources.limits.cpu` | CPU limit | `100m` |
| `metrics.resources.limits.memory` | Memory limit | `128Mi` |
| `metrics.resources.requests.cpu` | CPU request | `50m` |
| `metrics.resources.requests.memory` | Memory request | `64Mi` |

**Probe Configuration for Metrics**
| Parameter         | Description             | Default |
|-------------------|-------------------------|---------|
| `metrics.livenessProbe.enabled` | Enable liveness probe for metrics | `true` |
| `metrics.livenessProbe.initialDelaySeconds` | Initial delay for metrics liveness probe | `15` |
| `metrics.livenessProbe.periodSeconds` | Period for metrics liveness probe | `10` |
| `metrics.livenessProbe.timeoutSeconds` | Timeout for metrics liveness probe | `5` |
| `metrics.livenessProbe.successThreshold` | Success threshold for metrics liveness probe | `1` |
| `metrics.livenessProbe.failureThreshold` | Failure threshold for metrics liveness probe | `3` |
| `metrics.readinessProbe.enabled` | Enable readiness probe for metrics | `true` |
| `metrics.readinessProbe.initialDelaySeconds` | Initial delay for metrics readiness probe | `5` |
| `metrics.readinessProbe.periodSeconds` | Period for metrics readiness probe | `10` |
| `metrics.readinessProbe.timeoutSeconds` | Timeout for metrics readiness probe | `5` |
| `metrics.readinessProbe.successThreshold` | Success threshold for metrics readiness probe | `1` |
| `metrics.readinessProbe.failureThreshold` | Failure threshold for metrics readiness probe | `3` |

**Exporter Arguments**
| Parameter         | Description             | Default |
|-------------------|-------------------------|---------|
| `metrics.extraArgs.primary` | Extra args for primary exporter | `[]` |
| `metrics.extraArgs.secondary` | Extra args for secondary exporter | `[]` |

**Prometheus ServiceMonitor**
| Parameter         | Description             | Default |
|-------------------|-------------------------|---------|
| `metrics.serviceMonitor.enabled` | Create ServiceMonitor | `false` |
| `metrics.serviceMonitor.namespace` | ServiceMonitor namespace | `""` |
| `metrics.serviceMonitor.interval` | Scrape interval | `30s` |
| `metrics.serviceMonitor.scrapeTimeout` | Scrape timeout | `10s` |
| `metrics.serviceMonitor.labels` | ServiceMonitor labels | `{}` |
| `metrics.serviceMonitor.selector` | ServiceMonitor selector | `{}` |
| `metrics.serviceMonitor.relabelings` | ServiceMonitor relabelings | `[]` |
| `metrics.serviceMonitor.metricRelabelings` | ServiceMonitor metric relabelings | `[]` |
| `metrics.serviceMonitor.honorLabels` | Honor labels from targets | `false` |
| `metrics.serviceMonitor.jobLabel` | Job label for ServiceMonitor | `""` |

**Prometheus Rules**
| Parameter         | Description             | Default |
|-------------------|-------------------------|---------|
| `metrics.prometheusRule.enabled` | Create PrometheusRule | `false` |
| `metrics.prometheusRule.namespace` | PrometheusRule namespace | `""` |
| `metrics.prometheusRule.labels` | PrometheusRule labels | `{}` |
| `metrics.prometheusRule.groups` | Alert rule groups | `[]` |

### Security configuration

| Parameter                | Description            | Default |
|--------------------------|------------------------|---------|
| `serviceAccount.create`  | Create service account | `true`  |
| `serviceAccount.name` | Service account name | `""` |
| `serviceAccount.annotations` | Service account annotations | `{}` |
| `serviceAccount.automountServiceAccountToken` | Automount token | `false` |
| `rbac.create` | Create RBAC resources | `true` |
| `rbac.rules` | Custom RBAC rules | `[]` |
| `networkPolicy.enabled` | Enable network policy | `false` |
| `networkPolicy.allowExternal` | Allow external traffic | `true` |
| `networkPolicy.explicitNamespacesSelector` | Namespace selector | `{}` |
| `networkPolicy.ingressRules.primaryAccessOnlyFrom.enabled` | Restrict primary access | `false` |
| `networkPolicy.ingressRules.primaryAccessOnlyFrom.namespaceSelector` | Namespace selector | `{}` |
| `networkPolicy.ingressRules.primaryAccessOnlyFrom.podSelector` | Pod selector | `{}` |

### Volume permissions

| Parameter                      | Description                               | Default |
|--------------------------------|-------------------------------------------|---------|
| `volumePermissions.enabled`    | Enable init container to fix permissions | `true`  |
| `volumePermissions.image.registry` | Init container registry | `docker.io` |
| `volumePermissions.image.repository` | Init container repository | `busybox` |
| `volumePermissions.image.tag` | Init container tag | `1.36` |
| `volumePermissions.image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `volumePermissions.resources.limits.cpu` | CPU limit | `100m` |
| `volumePermissions.resources.limits.memory` | Memory limit | `128Mi` |
| `volumePermissions.resources.requests.cpu` | CPU request | `50m` |
| `volumePermissions.resources.requests.memory` | Memory request | `64Mi` |

### Advanced configuration

| Parameter            | Description                 | Default |
|----------------------|-----------------------------|---------|
| `initdbScripts`      | Dictionary of init scripts  | `{}`    |
| `initdbScriptsConfigMap` | ConfigMap with init scripts | `""` |
| `diagnosticMode.enabled` | Enable diagnostic mode | `false` |
| `diagnosticMode.command` | Diagnostic command | `["sleep"]` |
| `diagnosticMode.args` | Diagnostic args | `["infinity"]` |
| `commonAnnotations` | Common annotations | `{}` |
| `commonLabels` | Common labels | `{}` |
| `extraDeploy` | Extra Kubernetes objects | `[]` |
| `nameOverride` | Override chart name | `""` |
| `fullnameOverride` | Override full name | `""` |
| `namespaceOverride` | Override namespace | `""` |
| `clusterDomain` | Kubernetes cluster domain | `cluster.local` |

---

## Networking and services

### Service architecture

#### Standalone mode services

```
<release-name>-primary (ClusterIP)
├── Port: 3306 (MySQL)
└── Port: 9104 (Metrics, if enabled)

<release-name>-primary-headless (Headless)
└── Port: 3306 (MySQL)
```

#### Replication mode services

```
<release-name>-primary (ClusterIP)
├── Port: 3306 (MySQL)
└── Port: 9104 (Metrics, if enabled)

<release-name>-primary-headless (Headless)
└── Port: 3306 (MySQL)

<release-name>-secondary (ClusterIP)
├── Port: 3306 (MySQL)
└── Port: 9104 (Metrics, if enabled)

<release-name>-secondary-headless (Headless)
└── Port: 3306 (MySQL)
```

### DNS resolution

```bash
# Primary service
<release-name>-primary.<namespace>.svc.cluster.local

# Secondary service (replication mode)
<release-name>-secondary.<namespace>.svc.cluster.local

# Individual pod DNS
<release-name>-primary-0.<release-name>-primary-headless.<namespace>.svc.cluster.local
<release-name>-secondary-0.<release-name>-secondary-headless.<namespace>.svc.cluster.local
```

### Network policies

When enabled, network policies control traffic:

```yaml
networkPolicy:
  enabled: true
  allowExternal: false
  ingressRules:
    primaryAccessOnlyFrom:
      enabled: true
      namespaceSelector:
        matchLabels:
          name: allowed-namespace
      podSelector:
        matchLabels:
          app: allowed-app
```

### Service types

#### ClusterIP (Default)
```yaml
primary:
  service:
    type: ClusterIP
```

#### NodePort
```yaml
primary:
  service:
    type: NodePort
    nodePorts:
      mysql: 30306
```

#### LoadBalancer
```yaml
primary:
  service:
    type: LoadBalancer
    loadBalancerIP: "10.0.0.100"
    loadBalancerSourceRanges:
      - 10.0.0.0/8
```

---

## Security

### Password management

#### Auto-generated passwords
```bash
# Passwords are auto-generated if not specified
helm install my-mysql ./mysql-server-helm/mysql

# Retrieve auto-generated password
kubectl get secret my-mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d
```

#### Using existing secret
```bash
# Create secret
kubectl create secret generic mysql-passwords \
  --from-literal=mysql-root-password=myRootPassword \
  --from-literal=mysql-password=myUserPassword \
  --from-literal=mysql-replication-password=myReplPassword

# Use in installation
helm install my-mysql ./mysql-server-helm/mysql \
  --set auth.existingSecret=mysql-passwords
```

#### Password update during upgrades

> **📖 Complete Testing Guide Available**: For comprehensive manual testing procedures and validation steps for password changes in both replication and standalone modes, see [MySQL Password Change Testing Guide](docs/mysql-password-change-testing-guide.md)

```bash
# Create secret with previous passwords for validation
kubectl create secret generic mysql-previous-passwords \
  --from-literal=mysql-root-password=oldRootPassword \
  --from-literal=mysql-replication-password=oldReplPassword

# Update values to enable password update job
cat > password-update-values.yaml <<EOF
auth:
  rootPassword: newRootPassword
  replicationPassword: newReplPassword

passwordUpdateJob:
  enabled: true
  previousPasswords:
    existingSecret: mysql-previous-passwords
EOF

# Perform upgrade with new passwords
helm upgrade my-mysql ./mysql-server-helm/mysql \
  --reuse-values \
  -f password-update-values.yaml

# Or you can use --set to re-apply the changes
# architecture: standalone
helm upgrade --install my-mysql ./mysql-server-helm/mysql \
  --set architecture=standalone \
  --set passwordUpdateJob.enabled=true \
  --set auth.rootPassword=newpass456 \
  --set passwordUpdateJob.previousPasswords.rootPassword=testroot123

# architecture: replication
helm upgrade --install my-mysql ./mysql-server-helm/mysql \
  --set architecture=replication \
  --set passwordUpdateJob.enabled=true \
  --set auth.replicationPassword=replicapass123 \
  --set auth.rootPassword=newpass456 \
  --set passwordUpdateJob.previousPasswords.rootPassword=testroot123
```

### RBAC configuration

The chart creates:
- ServiceAccount
- Role with necessary permissions
- RoleBinding

Default permissions:
```yaml
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims", "services", "endpoints"]
    verbs: ["get", "list", "watch"]
```

### Security contexts

#### Pod security context
```yaml
podSecurityContext:
  enabled: true
  fsGroup: 999
  runAsUser: 999
```

#### Container security context
```yaml
containerSecurityContext:
  enabled: true
  runAsUser: 999
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

### Network security

#### Enable network policies
```yaml
networkPolicy:
  enabled: true
  allowExternal: false
  ingressRules:
    primaryAccessOnlyFrom:
      enabled: true
      namespaceSelector:
        matchLabels:
          purpose: production
```

### TLS/SSL configuration

> **Note**: TLS configuration can be implemented by mounting custom certificates and configuring MySQL through the configuration section. For production deployments, consider using network-level encryption (service mesh, IPSec) or configure MySQL SSL manually.

---

## Monitoring and observability

> **📖 Complete Guide Available**: For detailed Prometheus setup, advanced configurations, and troubleshooting, see [docs/prometheus-monitoring-guide.md](docs/prometheus-monitoring-guide.md)

### Quick start - metrics collection

#### Enable metrics
```yaml
metrics:
  enabled: true
  image:
    tag: "v0.17.2"
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
```

#### Available metrics

| Metric                                                | Description         | Type    |
|-------------------------------------------------------|---------------------|---------|
| `mysql_up`                                            | MySQL server status | Gauge   |
| `mysql_global_status_connections`                     | Total connections   | Counter |
| `mysql_global_status_threads_connected`               | Current connections | Gauge   |
| `mysql_global_status_queries`                         | Total queries       | Counter |
| `mysql_global_status_slow_queries`                    | Slow queries        | Counter |
| `mysql_global_status_innodb_row_lock_waits`           | InnoDB lock waits   | Counter |
| `mysql_global_status_innodb_buffer_pool_pages_*`      | Buffer pool metrics | Gauge   |
| `mysql_slave_lag_seconds`                             | Replication lag     | Gauge   |

### Prometheus integration

#### ServiceMonitor configuration
```yaml
metrics:
  serviceMonitor:
    enabled: true
    namespace: monitoring
    interval: 30s
    scrapeTimeout: 10s
    labels:
      prometheus: kube-prometheus
```

#### PrometheusRule configuration
```yaml
metrics:
  prometheusRule:
    enabled: true
    namespace: monitoring
    groups:
      - name: mysql.rules
        rules:
          - alert: MySQLDown
            expr: mysql_up == 0
            for: 5m
            labels:
              severity: critical
          - alert: MySQLTooManyConnections
            expr: mysql_global_status_threads_connected > mysql_global_variables_max_connections * 0.8
            for: 5m
            labels:
              severity: warning
```

### Grafana dashboards

Recommended dashboard IDs:

- **7362**: MySQL Overview
- **14057**: MySQL Exporter Quickstart
- **6239**: MySQL Performance
- **7991**: MySQL Replication

### Complete monitoring setup

The [Prometheus Monitoring Guide](docs/prometheus-monitoring-guide.md) provides comprehensive instructions for:

- **Full Prometheus stack setup** - Complete kube-prometheus-stack installation
- **Advanced configurations** - Replication monitoring, custom alerts, and performance tuning
- **Grafana integration** - Dashboard setup with custom panels and business metrics
- **Production examples** - Real-world configuration files for various deployment scenarios
- **Troubleshooting** - Common issues and solutions for metrics collection
- **Security best practices** - Network policies and access controls for monitoring

### Logging

MySQL logs are available via:
```bash
# Primary logs
kubectl logs my-mysql-primary-0

# Secondary logs
kubectl logs my-mysql-secondary-0

# Metrics exporter logs
kubectl logs my-mysql-primary-0 -c metrics
```

Configure custom log levels:
```yaml
# Example:
primary:
  configuration: |
    [mysqld]
    log-error=/var/log/mysql/error.log
    general_log=1
    general_log_file=/var/log/mysql/general.log
    slow_query_log=1
    slow_query_log_file=/var/log/mysql/slow.log
    long_query_time=2
```

---

## Operations and maintenance

### Backup strategies

#### Manual backup
```bash
# Create backup
kubectl exec my-mysql-primary-0 -- \
  mysqldump -uroot -p${MYSQL_ROOT_PASSWORD} \
  --all-databases --single-transaction \
  --flush-logs --master-data=2 > backup.sql

# Compress backup
gzip backup.sql
```

#### Automated backup (CronJob)
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mysql-backup
            image: mysql:9.4.0
            command:
            - /bin/bash
            - -c
            - |
              mysqldump -h my-mysql-primary \
                -uroot -p${MYSQL_ROOT_PASSWORD} \
                --all-databases > /backup/backup-$(date +%Y%m%d).sql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-mysql
                  key: mysql-root-password
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: mysql-backup-pvc
```

### Restore procedures

```bash
# Restore from backup
kubectl exec -i my-mysql-primary-0 -- \
  mysql -uroot -p${MYSQL_ROOT_PASSWORD} < backup.sql
```

### Scaling operations

#### Scale secondary replicas
```bash
# Scale up
helm upgrade my-mysql ./mysql-server-helm/mysql \
  --reuse-values \
  --set secondary.replicaCount=3

# Scale down
helm upgrade my-mysql ./mysql-server-helm/mysql \
  --reuse-values \
  --set secondary.replicaCount=1
```

#### Minor version upgrade
```bash
# Backup first
kubectl exec my-mysql-primary-0 -- mysqldump -uroot -p${MYSQL_ROOT_PASSWORD} --all-databases > pre-upgrade-backup.sql

# Upgrade
helm upgrade my-mysql ./mysql-server-helm/mysql \
  --reuse-values \
  --set image.tag=9.4.1
```

#### Major version upgrade
```bash
# 1. Create full backup
kubectl exec my-mysql-primary-0 -- mysqldump -uroot -p${MYSQL_ROOT_PASSWORD} \
  --all-databases --routines --triggers --events > full-backup.sql

# 2. Stop replication (if applicable)
kubectl exec my-mysql-secondary-0 -- mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "STOP SLAVE"

# 3. Delete StatefulSets (keep PVCs)
kubectl delete statefulset my-mysql-primary my-mysql-secondary --cascade=orphan

# 4. Upgrade with new version
helm upgrade my-mysql ./mysql-server-helm/mysql \
  --set image.tag=9.4.1 \
  --reuse-values

# 5. Verify and restart replication
kubectl exec my-mysql-secondary-0 -- mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "START SLAVE"
```

### Performance tuning

#### Buffer pool optimization
```yaml
primary:
  configuration: |
    [mysqld]
    # Set to 70-80% of available memory (MySQL 9.x compatible)
    innodb_buffer_pool_size=6G
    innodb_buffer_pool_instances=4
```

#### Connection pool tuning
```yaml
primary:
  configuration: |
    [mysqld]
    max_connections=500
    max_connect_errors=1000000
    thread_cache_size=128
    thread_stack=256K
```

#### Query optimization
```yaml
primary:
  configuration: |
    [mysqld]
    # Query cache removed in MySQL 8.0+
    slow_query_log=1
    long_query_time=1
    log_queries_not_using_indexes=1
    # Optimizer settings for MySQL 9.x
    optimizer_switch='index_merge=on,index_merge_union=on'
```

### Maintenance tasks

#### Analyze tables
```bash
kubectl exec my-mysql-primary-0 -- mysqlcheck -uroot -p${MYSQL_ROOT_PASSWORD} --analyze --all-databases
```

#### Optimize tables
```bash
kubectl exec my-mysql-primary-0 -- mysqlcheck -uroot -p${MYSQL_ROOT_PASSWORD} --optimize --all-databases
```

#### Check table integrity
```bash
kubectl exec my-mysql-primary-0 -- mysqlcheck -uroot -p${MYSQL_ROOT_PASSWORD} --check --all-databases
```

---

## Troubleshooting

### Common issues and solutions

#### Pod stuck in pending state

**Symptom**: Pod shows `Pending` status
```bash
kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
my-mysql-primary-0      0/1     Pending   0          5m
```

**Diagnosis**:
```bash
kubectl describe pod my-mysql-primary-0
```

**Common causes and solutions**:

| Cause                   | Solution                                  |
|-------------------------|-------------------------------------------|
| Insufficient resources  | Scale cluster or reduce resource requests |
| PVC not bound           | Check storage class and available PVs     |
| Node selector mismatch  | Verify node labels match selector         |
| Taints/tolerations      | Add appropriate tolerations               |

#### Authentication failures

**Symptom**: Access denied errors
```
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)
```

**Solutions**:
```bash
# Verify password is correct
kubectl get secret my-mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d

# Reset root password if needed
kubectl exec my-mysql-primary-0 -- mysql -uroot -e "ALTER USER 'root'@'%' IDENTIFIED BY 'NewPassword123!'"
```

#### Replication not working

**Symptom**: Secondary not replicating
```bash
kubectl exec my-mysql-secondary-0 -- mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "SHOW REPLICA STATUS" | grep Running
Slave_IO_Running: No
Slave_SQL_Running: No
```

**Diagnosis & Fix**:
```bash
# Check error
kubectl exec my-mysql-secondary-0 -- mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "SHOW REPLICA STATUS" | grep Last_Error

# Reset replication
kubectl exec my-mysql-secondary-0 -- mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "
STOP SLAVE;
RESET SLAVE ALL;
CHANGE MASTER TO
  MASTER_HOST='my-mysql-primary.default.svc.cluster.local',
  MASTER_PORT=3306,
  MASTER_USER='replicator',
  MASTER_PASSWORD='${MYSQL_REPLICATION_PASSWORD}',
  MASTER_AUTO_POSITION=1;
START SLAVE;"
```

#### High memory usage

**Symptom**: OOMKilled pods
```bash
kubectl describe pod my-mysql-primary-0 | grep -A 5 "Last State"
    Last State:     Terminated
      Reason:       OOMKilled
```

**Solution**:
```yaml
# Increase memory limits
primary:
  resources:
    limits:
      memory: 2Gi
  configuration: |
    [mysqld]
    innodb_buffer_pool_size=1G  # Set to 50-70% of container memory
```

#### Slow queries

**Diagnosis**:
```bash
# Enable slow query log
kubectl exec my-mysql-primary-0 -- mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "
SET GLOBAL slow_query_log = 1;
SET GLOBAL long_query_time = 1;"

# Check slow queries
kubectl exec my-mysql-primary-0 -- tail -f /var/log/mysql/slow.log
```

**Optimization**:
```bash
# Analyze query execution plan
kubectl exec my-mysql-primary-0 -- mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "EXPLAIN SELECT ..."

# Add indexes as needed
kubectl exec my-mysql-primary-0 -- mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "CREATE INDEX idx_name ON table(column)"
```

### Debug mode

Enable diagnostic mode for troubleshooting:
```yaml
diagnosticMode:
  enabled: true
  command:
    - sleep
  args:
    - infinity
```

Then exec into pod:
```bash
kubectl exec -it my-mysql-primary-0 -- /bin/bash
```

### Log analysis

#### Check MySQL error log
```bash
kubectl logs my-mysql-primary-0 | grep ERROR
```

#### Check metrics exporter logs
```bash
kubectl logs my-mysql-primary-0 -c metrics
```

#### Enable verbose logging
```yaml
primary:
  configuration: |
    [mysqld]
    log_error_verbosity=3
    general_log=1
```

---

## Migration guide

### From standalone MySQL

#### Using mysqldump

```bash
# 1. Create backup from source
mysqldump -h source-mysql -uroot -p --all-databases > backup.sql

# 2. Install chart
helm install my-mysql ./mysql-server-helm/mysql --set auth.rootPassword=MyNewPassword

# 3. Import data
kubectl exec -i my-mysql-primary-0 -- mysql -uroot -pMyNewPassword < backup.sql
```

#### Using PVC migration

```bash
# 1. Create PVC from existing data
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-migration
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF

# 2. Copy data to PVC (using a temporary pod)
kubectl run mysql-migrate --image=mysql:9.4.0 --rm -it --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"mysql-migrate","volumeMounts":[{"name":"data","mountPath":"/var/lib/mysql"}]}],"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"mysql-data-migration"}}]}}' \
  -- bash -c "cp -r /source/data/* /var/lib/mysql/"

# 3. Install chart with existing PVC
helm install my-mysql ./mysql-server-helm/mysql \
  --set primary.persistence.existingClaim=mysql-data-migration
```

---

## API reference

### Template functions

| Function                      | Description              | Example                                     |
|-------------------------------|--------------------------|---------------------------------------------|
| `mysql.name`                  | Chart name               | `mysql`                                     |
| `mysql.fullname`              | Full release name        | `my-mysql`                                  |
| `mysql.chart`                 | Chart name and version   | `mysql-1.0.0`                              |
| `mysql.labels`                | Standard labels          | See _helpers.tpl                            |
| `mysql.selectorLabels`        | Selector labels          | See _helpers.tpl                            |
| `mysql.serviceAccountName`    | Service account name     | `my-mysql`                                  |
| `mysql.image`                 | MySQL image              | `docker.io/mysql:9.4.0`                    |
| `mysql.metrics.image`         | Metrics image            | `docker.io/prom/mysqld-exporter:v0.17.2`   |
| `mysql.secretName`            | Secret name              | `my-mysql`                                  |
| `mysql.primary.fullname`      | Primary fullname         | `my-mysql-primary`                          |
| `mysql.secondary.fullname`    | Secondary fullname       | `my-mysql-secondary`                        |

### Kubernetes resources created

| Resource type           | Name pattern                              | Condition                         |
|-------------------------|-------------------------------------------|-----------------------------------|
| **Secret**              | `{release-name}`                          | `!auth.existingSecret`            |
| **ServiceAccount**      | `{release-name}`                          | `serviceAccount.create`           |
| **Role**                | `{release-name}`                          | `rbac.create`                     |
| **RoleBinding**         | `{release-name}`                          | `rbac.create`                     |
| **ConfigMap**           | `{release-name}-primary`                  | Primary configuration             |
| **ConfigMap**           | `{release-name}-primary-replication`     | Replication mode                  |
| **ConfigMap**           | `{release-name}-secondary`                | Secondary configuration           |
| **ConfigMap**           | `{release-name}-secondary-replication`   | Replication mode                  |
| **Service**             | `{release-name}-primary`                  | Always                            |
| **Service**             | `{release-name}-primary-headless`        | Always                            |
| **Service**             | `{release-name}-secondary`                | Replication mode                  |
| **Service**             | `{release-name}-secondary-headless`      | Replication mode                  |
| **StatefulSet**         | `{release-name}-primary`                  | Always                            |
| **StatefulSet**         | `{release-name}-secondary`                | Replication mode                  |
| **PVC**                 | `data-{release-name}-primary-0`          | `primary.persistence.enabled`     |
| **PVC**                 | `data-{release-name}-secondary-{n}`      | `secondary.persistence.enabled`   |
| **NetworkPolicy**       | `{release-name}`                          | `networkPolicy.enabled`           |
| **PodDisruptionBudget** | `{release-name}-primary`                  | `primary.pdb.create`              |
| **PodDisruptionBudget** | `{release-name}-secondary`                | `secondary.pdb.create`            |
| **ServiceMonitor**      | `{release-name}`                          | `metrics.serviceMonitor.enabled`  |
| **PrometheusRule**      | `{release-name}`                          | `metrics.prometheusRule.enabled`  |

### Labels

Standard labels applied to all resources:
```yaml
app.kubernetes.io/name: mysql
app.kubernetes.io/instance: {release-name}
app.kubernetes.io/version: "9.4.0"
app.kubernetes.io/managed-by: Helm
app.kubernetes.io/component: {primary|secondary}
helm.sh/chart: mysql-1.0.0
```

### Annotations

Configurable annotations:
- `commonAnnotations`: Applied to all resources
- `primary.podAnnotations`: Applied to primary pods
- `secondary.podAnnotations`: Applied to secondary pods
- `primary.service.annotations`: Applied to primary service
- `secondary.service.annotations`: Applied to secondary service

---

## Examples

### Development environment

```yaml
# dev-values.yaml
architecture: standalone
auth:
  rootPassword: "dev123"
  database: "devdb"
  
primary:
  persistence:
    size: 1Gi
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"

metrics:
  enabled: false
```

```bash
helm install mysql-dev ./mysql-server-helm/mysql -f dev-values.yaml
```

### Production with high availability

```yaml
# prod-ha-values.yaml
architecture: replication

auth:
  existingSecret: mysql-prod-secret
  database: myapp
  
primary:
  persistence:
    size: 50Gi
    storageClass: standard
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  pdb:
    create: true
    minAvailable: 1
  
secondary:
  replicaCount: 2
  persistence:
    size: 50Gi
    storageClass: standard
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1000m"
  pdb:
    create: true
    minAvailable: 1

metrics:
  enabled: true
  serviceMonitor:
    enabled: true

networkPolicy:
  enabled: true

volumePermissions:
  enabled: true
```

### Multi-tenant setup

```yaml
# multi-tenant-values.yaml
architecture: standalone

auth:
  rootPassword: "admin-secret"
  
initdbScripts:
  01-create-databases.sql: |
    CREATE DATABASE IF NOT EXISTS tenant1;
    CREATE DATABASE IF NOT EXISTS tenant2;
    CREATE DATABASE IF NOT EXISTS tenant3;
    
  02-create-users.sql: |
    CREATE USER 'tenant1'@'%' IDENTIFIED BY 'tenant1pass';
    GRANT ALL PRIVILEGES ON tenant1.* TO 'tenant1'@'%';
    
    CREATE USER 'tenant2'@'%' IDENTIFIED BY 'tenant2pass';
    GRANT ALL PRIVILEGES ON tenant2.* TO 'tenant2'@'%';
    
    CREATE USER 'tenant3'@'%' IDENTIFIED BY 'tenant3pass';
    GRANT ALL PRIVILEGES ON tenant3.* TO 'tenant3'@'%';
    
    FLUSH PRIVILEGES;

primary:
  configuration: |
    [mysqld]
    max_connections=1000
    max_user_connections=100
```

### Read-heavy workload

```yaml
# read-heavy-values.yaml
architecture: replication

auth:
  rootPassword: "secure-password"
  replicationPassword: "repl-password"

primary:
  persistence:
    size: 50Gi
  resources:
    requests:
      memory: "2Gi"
      cpu: "1"
  configuration: |
    [mysqld]
    # Optimize for writes
    innodb_flush_log_at_trx_commit=2
    sync_binlog=0
    innodb_buffer_pool_size=1500M

secondary:
  replicaCount: 5  # Multiple read replicas
  persistence:
    size: 50Gi
  resources:
    requests:
      memory: "4Gi"
      cpu: "2"
  configuration: |
    [mysqld]
    # Optimize for reads (MySQL 9.x compatible)
    innodb_buffer_pool_size=3G
    # Query cache removed in MySQL 8.0+, use result cache instead
    read_buffer_size=2M
    read_rnd_buffer_size=8M
    # Adaptive hash index for better read performance
    innodb_adaptive_hash_index=ON
```

### Disaster recovery setup

```yaml
# dr-values.yaml
architecture: replication

auth:
  existingSecret: mysql-dr-secret

primary:
  persistence:
    size: 200Gi
    storageClass: replicated-storage  # Storage with built-in replication
  configuration: |
    [mysqld]
    # Binary logging for point-in-time recovery
    log-bin=mysql-bin
    binlog_format=ROW
    expire_logs_days=7
    max_binlog_size=100M
    
    # Backup-friendly settings
    innodb_flush_log_at_trx_commit=1
    sync_binlog=1

secondary:
  replicaCount: 2
  persistence:
    size: 200Gi
    storageClass: replicated-storage

# Backup CronJob (deploy separately)
backup:
  schedule: "0 */6 * * *"  # Every 6 hours
  retention: 30  # days
  destination: s3://my-bucket/mysql-backups/
```

---

## Appendix

### Environment variables reference

| Variable                       | Description           | Used by             |
|--------------------------------|-----------------------|---------------------|
| `MYSQL_ROOT_PASSWORD`          | Root password         | Primary & Secondary |
| `MYSQL_DATABASE`               | Database to create    | Primary             |
| `MYSQL_USER`                   | User to create        | Primary             |
| `MYSQL_PASSWORD`               | User password         | Primary             |
| `MYSQL_REPLICATION_USER`       | Replication user      | Primary & Secondary |
| `MYSQL_REPLICATION_PASSWORD`   | Replication password  | Primary & Secondary |
| `MYSQL_MASTER_HOST`            | Primary hostname      | Secondary           |
| `MYSQL_MASTER_PORT`            | Primary port          | Secondary           |

### Port reference

| Port  | Protocol | Description                 |
|-------|----------|-----------------------------|
| 3306  | TCP      | MySQL server                |
| 9104  | TCP      | Metrics exporter            |
| 33060 | TCP      | MySQL X Protocol (not used) |

### File paths

| Path                            | Description             |
|---------------------------------|-------------------------|
| `/var/lib/mysql`                | Data directory          |
| `/etc/mysql/conf.d/`            | Configuration directory |
| `/docker-entrypoint-initdb.d/`  | Init scripts directory  |
| `/var/log/mysql/`               | Log directory           |
| `/var/run/mysqld/`              | Runtime directory       |

### Resource recommendations

| Workload type     | CPU request | CPU limit | Memory request | Memory limit | Storage |
|-------------------|-------------|-----------|----------------|--------------|---------|
| Development       | 100m        | 500m      | 256Mi          | 512Mi        | 1Gi     |
| Small production  | 500m        | 2         | 1Gi            | 2Gi          | 10Gi    |
| Medium production | 2           | 4         | 4Gi            | 8Gi          | 50Gi    |
| Large production  | 4           | 8         | 16Gi           | 32Gi         | 200Gi   |
| Extra large       | 8           | 16        | 64Gi           | 128Gi        | 1Ti     |

### Version history

| Chart version | App version | Release date | Changes                           |
|---------------|-------------|--------------|-----------------------------------|
| 1.0.0         | 9.4.0       | 2025-08-28   | Initial release with MySQL 9.4.0 |

### Support matrix

| MySQL version | Chart support  | Notes                                    |
|---------------|----------------|------------------------------------------|
| 9.4.0         | ✅ Full        | Tested and supported                     |
| 9.0.x - 9.3.x | ⚠️ Compatible  | Should work but not fully tested         |
| 8.x.x         | ❌ Not supported | Use older chart version or upgrade MySQL |

---

## Additional resources

### Documentation
- [MySQL official documentation](https://dev.mysql.com/doc/)
- [Kubernetes StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Prometheus operator](https://prometheus-operator.dev/)
- [Helm documentation](https://helm.sh/docs/)

### Related guides
- [Prometheus monitoring guide](docs/prometheus-monitoring-guide.md) - Complete Prometheus setup and monitoring
- [MySQL replication guide](docs/mysql-replication-guide.md)
- [Installation guide](docs/mysql-install-guide.md)
- [MySQL password change testing guide](docs/mysql-password-change-testing-guide.md) - Comprehensive manual testing procedures for password changes

### Community
- Report issues: [GitHub Issues](https://github.com/WebSpellChecker/mysql-server-helm/issues)
- Contribute: [Contributing guide](CONTRIBUTING.md)
- Discussions: [GitHub Discussions](https://github.com/WebSpellChecker/mysql-server-helm/discussions)
