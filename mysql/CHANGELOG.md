# Changelog

## [1.1.0](https://github.com/WebSpellChecker/mysql-server-helm/compare/v1.0.1...v1.1.0) - 2026-10-07

### Features

* check writes, the application user, and replication in helm test (#17) ([2933dfa](https://github.com/WebSpellChecker/mysql-server-helm/commit/2933dfa7af560b5b674882252bc29a84c0c58b95))

### Bug fixes

* create the chart Secret when auth.existingSecret is not set (#14) ([b4ddecd](https://github.com/WebSpellChecker/mysql-server-helm/commit/b4ddecd6e18531268d0c661953dd37c03fc2415f))
* set up replication on the secondary through the local socket (#15) ([01338c3](https://github.com/WebSpellChecker/mysql-server-helm/commit/01338c33ad751250d450daea09a21464e643f7a7))
* report readiness only when MySQL accepts TCP connections (#18) ([4afd88a](https://github.com/WebSpellChecker/mysql-server-helm/commit/4afd88a3c689c509fd5b864796566695793623f9))

## [1.0.1](https://github.com/WebSpellChecker/mysql-server-helm/compare/v1.0.0...v1.0.1) - 2026-10-05

### Bug fixes

* **chart:** list the infrastructure team as maintainer ([82d17fe](https://github.com/WebSpellChecker/mysql-server-helm/commit/82d17fec053242a5134e1f56e8297dbe5a6a9cf5))

## [1.0.0](https://github.com/WebSpellChecker/mysql-server-helm/releases/tag/v1.0.0) - 2026-10-05

### Features

* first release of the chart with the official MySQL images (default MySQL 9.4.0)
* standalone and primary-replica (replication) architectures
* chart-managed Secret or an existing Secret
* password update Job that changes the passwords during an upgrade
* init scripts from values or from an existing ConfigMap
* optional metrics exporter, ServiceMonitor, and PrometheusRule
* optional NetworkPolicy and PodDisruptionBudget, and a Helm test
