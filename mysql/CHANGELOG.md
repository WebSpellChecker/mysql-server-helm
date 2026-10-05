# Changelog

## [1.0.1](https://github.com/WebSpellChecker/mysql-server-helm/compare/v1.0.0...v1.0.1) (2026-10-05)


### Bug fixes

* **chart:** list the infrastructure team as maintainer ([a41f789](https://github.com/WebSpellChecker/mysql-server-helm/commit/a41f789275cb3370a4f18791da0435a0e811abbb))
* **chart:** list the infrastructure team as maintainer ([82d17fe](https://github.com/WebSpellChecker/mysql-server-helm/commit/82d17fec053242a5134e1f56e8297dbe5a6a9cf5))

## [1.0.0](https://github.com/WebSpellChecker/mysql-server-helm/releases/tag/v1.0.0) (2026-10-05)


### Features

* first release of the chart with the official MySQL images (default MySQL 9.4.0)
* standalone and primary-replica (replication) architectures
* chart-managed Secret or an existing Secret
* password update Job that changes the passwords during an upgrade
* init scripts from values or from an existing ConfigMap
* optional metrics exporter, ServiceMonitor, and PrometheusRule
* optional NetworkPolicy and PodDisruptionBudget, and a Helm test
