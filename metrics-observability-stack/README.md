## Overall Layout of Metrics Observability Stack

```
metrics-observability-stack/
│
├── prometheus/
│   ├── binary-installation/
│   ├── docker-compose/
│   ├── configuration/
│   └── alert-rules/
│
├── grafana/
│   ├── binary-installation/
│   ├── docker-compose/
│   └── dashboards/
│
├── alertmanager/
│   ├── binary-installation/
│   ├── docker-compose/
│   ├── configuration/
│   ├── receivers/
│   ├── routing/
│   └── templates/
│
├── exporters/
│   ├── node-exporter/
│   ├── blackbox-exporter/
│   └── mysqld-exporter/
│
├── docker/
├── scripts/
├── images/
├── architecture/
└── README.md

```