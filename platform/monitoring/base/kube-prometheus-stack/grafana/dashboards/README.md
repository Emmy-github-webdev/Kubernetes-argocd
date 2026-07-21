# Grafana Dashboards

This directory contains custom Grafana dashboards provisioned through
the kube-prometheus-stack Grafana sidecar.

## Dashboards

- kubernetes.json
- nodes.json
- ingress-nginx.json
- springboot.json
- jvm.json
- postgres.json
- redis.json

Each dashboard is packaged into a ConfigMap by Kustomize and
automatically imported into Grafana.