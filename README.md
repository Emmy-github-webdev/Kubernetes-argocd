# Kubernetes-argocd

### Architecture


### Action Runner Controller (ARC) - Self hosted runner

```
                      GitHub
                         │
                         │ Workflow Dispatch
                         ▼
               Actions Runner Controller
                      (EKS)
                         │
               Creates ephemeral runner Pods
                         │
                         ▼
               GitHub Runner Pod
               (private subnet)
                         │
       ┌─────────────────┴─────────────────┐
       │                                   │
       ▼                                   ▼
Terraform AWS Provider              PostgreSQL Provider
       │                                   │
       ▼                                   ▼
 AWS APIs                          Private RDS (5432)
```

### Steps

1. Create a GitHub App
- Go to settings
- Developer settings
- GitHub Apps
- New GitHub App
- Permissions
  - Repository
    - Actions - read/Write
    - Contents - Read
    - Metadata - Read

  - Organization
    - Self-hosted runners
    - Read/Write
- Download private-key.pem after creation - private-key.pem. You will need 
  - App ID
  - Installation ID
  - Private Key

2. Create namespace


Source Repo
├── user-service
├── order-service
├── payment-service
└── product-service

↓

GitHub Actions

↓

ECR

↓


Kubernetes-argocd Repo
├── apps
│   ├── user-service
│   │   ├── base
|   │   |   ├── deployment.yaml
|   │   |   ├── kustomization.yaml
|   │   |   └── poddistruption.yaml
|   │   |   └── service.yaml
│   │   └── overlays
|   │   |   ├── dev
|   |   |   |   ├── external-secret-patch.yaml
|   |   |   |   ├── Image-patch.yaml
|   |   |   |   ├── kustomization.yaml
|   │   |   ├── staging
|   │   |   └── prod
│   ├── order-service
│   ├── payment-service
│   └── product-service
|
├── platform
    │
    ├── monitoring
    │   ├── base
    │   │   ├── namespace.yaml
    │   │   ├── kustomization.yaml
    |   │   ├── grafana/
    │   |   │   ├── datasources.yaml
    │   │   |   ├── dashboardproviders.yaml
    │   │   |   └── dashboards/
    │   │   |        ├── kubernetes.json
    │   │   |        ├── nodes.json
    │   │   |        ├── ingress-nginx.json
    │   │   |        ├── springboot.json
    │   │   |        ├── jvm.json
    │   │   |        ├── postgres.json
    │   │   |        └── redis.json
    │   │   |
    │   |   |    ├── alertmanager/
    │   │   |    |     ├── config.yaml
    │   │   |    |     ├── receivers.yaml
    │   │   |    |     ├── routes.yaml
    │   │   |    |     ├── inhibit-rules.yaml
    │   │        |     └── templates/
    │   │                   ├── slack.tmpl
    │   │                   └── email.tmpl
    │   │   ├── servicemonitors/
    │   │   │   ├── order.yaml
    │   │   │   ├── user.yaml
    │   │   │   ├── payment.yaml
    │   │   │   ├── ingress-nginx.yaml
    │   │   │   ├── postgres.yaml
    │   │   │   ├── redis.yaml
    │   │   │   └── product.yaml
    │   │   │
    │   │   └── prometheusrules/
    │   │       ├── high-cpu.yaml
    │   │       ├── high-memory.yaml
    │   │       ├── pod-restarts.yaml
    │   │       ├── node-health.yaml
    │   │       ├── api-errors.yaml
    │   │       ├── latency.yaml
    │   │       ├── disk-space.yaml
    │   │       └── database-down.yaml
    │   │   ├── recording-rules/
    │   │   │   ├── cluster.yaml
    │   │   │   ├── nodes.yaml
    │   │   │   ├── workloads.yaml
    │   │   │   └── applications.yaml
    │   │   ├── exporters/
    │   │   │   ├── blackbox-exporter.yaml
    │   │   │   ├── postgres-exporter.yaml
    │   │   │   ├── Redis-exporter.yaml
    │   │   │   └── jmx-exporter.yaml
    │   │   ├── exporters/
    │   │   │   └── allow-prometheus.yaml
    │   │
    │   └── overlays
    │       ├── dev
    │       │   ├── kustomization.yaml
    │       │   └── values-patch.yaml
    │       │
    │       ├── staging
    │       └── prod
│
    logging/
    ├── base/
    │   ├── namespace.yaml
    │   ├── kustomization.yaml
    │   │
    │   ├── loki/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── promtail/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── fluent-bit/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── log-retention/
    │   │   └── retention.yaml
    │   │
    │   └── networkpolicy/
    │       └── allow-logging.yaml
    │
    └── overlays/
        ├── dev/
        ├── staging/
        └── prod/
|
|
    tracing/
    ├── base/
    │   ├── namespace.yaml
    │   ├── kustomization.yaml
    │   │
    │   ├── tempo/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── opentelemetry-collector/
    │   │   ├── helm-release.yaml
    │   │   ├── values.yaml
    │   │   └── pipelines.yaml
    │   │
    │   └── networkpolicy/
    │       └── allow-tracing.yaml
    │
    └── overlays/
        ├── dev/
        ├── staging/
        └── prod/
|
|
    ingress/
    ├── base/
    │   ├── namespace.yaml
    │   ├── kustomization.yaml
    │   │
    │   ├── aws-load-balancer-controller/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── ingress-nginx/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── external-dns/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   └── cert-manager/
    │       ├── helm-release.yaml
    │       ├── values.yaml
    │       └── clusterissuers/
    │           ├── letsencrypt-prod.yaml
    │           └── letsencrypt-staging.yaml
    │
    └── overlays/
        ├── dev/
        ├── staging/
        └── prod/
|
    security/
    ├── base/
    │   ├── namespace.yaml
    │   ├── kustomization.yaml
    │   │
    │   ├── kyverno/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── external-secrets/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── sealed-secrets/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── networkpolicies/
    │   │
    │   ├── podsecurity/
    │   │
    │   └── policies/
    │       ├── restrict-privileged.yaml
    │       ├── require-limits.yaml
    │       ├── require-probes.yaml
    │       └── disallow-latest-tag.yaml
    │
    └── overlays/
        ├── dev/
        ├── staging/
        └── prod/
|
|
    networking/
    ├── base/
    │   ├── namespace.yaml
    │   ├── kustomization.yaml
    │   │
    │   ├── cni/
    │   │
    │   ├── metrics-server/
    │   │   ├── helm-release.yaml
    │   │   └── values.yaml
    │   │
    │   ├── gateway-api/
    │   │
    │   ├── networkpolicies/
    │   │
    │   └── dns/
    │       └── coredns-patch.yaml
    │
    └── overlays/
        ├── dev/
        ├── staging/
        └── prod/
|
|
storage/
├── base/
│   ├── namespace.yaml
│   ├── kustomization.yaml
│   │
│   ├── ebs-csi-driver/
│   │   ├── helm-release.yaml
│   │   └── values.yaml
│   │
│   ├── efs-csi-driver/
│   │   ├── helm-release.yaml
│   │   └── values.yaml
│   │
│   ├── storageclasses/
│   │   ├── gp3.yaml
│   │   ├── efs.yaml
│   │   └── io2.yaml
│   │
│   ├── volume-snapshots/
│   │   ├── snapshotclass.yaml
│   │   └── schedules.yaml
│   │
│   └── backup/
│       ├── velero/
│       │   ├── helm-release.yaml
│       │   └── values.yaml
│       └── backup-schedules.yaml
│
└── overlays/
    ├── dev/
    ├── staging/
    └── prod/
|
|___argocd
│   ├── dev
│   |   ├── applicationset-apps.yaml
|   |   ├── applicationset-infra.yaml
|   |   ├── applicationset-monitoring.yaml
│   |   └── root-app.yaml
|   |
│   ├── prod
|   |
│   ├── staging
|
|___infrastructure
│   |   ├── dev
│   |   |   ├── postgres-master-secret.yaml
│   |   |   ├── order-db-secret.yaml
│   |   |   ├── user-db-secret.yaml
│   |   |   ├── payment-db-secret.yaml
│   |   |   ├── product-db-secret.yaml
│   |   |   └── postgres-bootstrap-job.yaml
│   |   |   ├── cluster-secret-store.yaml
│   |   |   ├── ingress.yaml
│   |   |   └── namespace-database.yaml
│   |   ├── prod
│   |   |
│   |   ├── staging


↓

EKS
### App Repo GitHub Action

name: Build

on:
  push:
    branches:
      - main

jobs:

  build:
9949494944994/C    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-ecr-push
          aws-region: us-east-1

      - uses: aws-actions/amazon-ecr-login@v2

      - run: |
          docker build -t my-app:${GITHUB_SHA} .
          docker tag my-app:${GITHUB_SHA} $ECR_REPO:${GITHUB_SHA}
          docker push $ECR_REPO:${GITHUB_SHA}



Update GitOps repo

After push

- name: Update GitOps
  run: |
    yq -i '.spec.template.spec.containers[0].image = "'"$ECR_REPO:${GITHUB_SHA}"'"' deployment.yaml

    git commit -am "Deploy ${GITHUB_SHA}"
    git push





infrastructure

├── dev
│   ├── postgres-master-secret.yaml
│   ├── order-db-secret.yaml
│   ├── user-db-secret.yaml
│   ├── payment-db-secret.yaml
│   ├── product-db-secret.yaml
│   └── postgres-bootstrap-job.yaml
│   ├── cluster-secret-store.yaml
│   ├── ingress.yaml
│   └── namespace-database.yaml
