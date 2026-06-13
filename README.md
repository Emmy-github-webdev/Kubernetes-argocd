# Kubernetes-argocd

### App Repo GitHub Action

name: Build

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

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