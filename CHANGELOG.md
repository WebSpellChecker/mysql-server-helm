# Changelog

This file records changes to the MySQL Helm chart.
The chart follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html):
`version` tracks the chart, while `appVersion` tracks the default MySQL image.

## [1.0.0]

Initial release with the official MySQL images (default MySQL 9.4.0).

### Added

- Standalone and primary-replica (replication) architectures.
- Support for a chart-managed Secret or an existing Secret.
- A password update Job that changes the passwords during an upgrade.
- Init scripts from values or from an existing ConfigMap.
- Optional metrics exporter, ServiceMonitor, and PrometheusRule.
- Optional NetworkPolicy and PodDisruptionBudget, and a Helm test.
