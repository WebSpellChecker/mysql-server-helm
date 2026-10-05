# Changelog

## [1.0.0](https://github.com/WebSpellChecker/mysql-server-helm/releases/tag/v1.0.0) - 2026-10-05

### Features

* first release of the chart with the official MySQL images (default MySQL 9.4.0)
* standalone and primary-replica (replication) architectures
* chart-managed Secret or an existing Secret
* password update Job that changes the passwords during an upgrade
* init scripts from values or from an existing ConfigMap
* optional metrics exporter, ServiceMonitor, and PrometheusRule
* optional NetworkPolicy and PodDisruptionBudget, and a Helm test
