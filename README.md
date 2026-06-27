# Kubernetes-argocd

### Architecture

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
|
│
├── monitoring
│   ├── base
│   │   ├── namespace.yaml
│   │   ├── kustomization.yaml
│   │   ├── prometheus-values.yaml
│   │   ├── grafana-dashboards/
│   │       ├── kubernetes.json
│   │       ├── jvm.json
│   │       ├── postgres.json
│   │       ├── predis.json
│   │       └── springboot.json
│   │   ├── servicemonitors/
│   │   │   ├── order.yaml
│   │   │   ├── user.yaml
│   │   │   ├── payment.yaml
│   │   │   └── product.yaml
│   │   │
│   │   └── prometheusrules/
│   │       ├── high-cpu.yaml
│   │       ├── high-memory.yaml
│   │       ├── pod-restarts.yaml
│   │       └── database-down.yaml
│   │
│   └── overlays
│       ├── dev
│       │   ├── kustomization.yaml
│       │   └── values-patch.yaml
│       │
│       ├── staging
│       └── prod
│
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