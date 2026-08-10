# Post-Deployment Verification and Troubleshooting

This guide describes the standard verification flow after deploying the Kubernetes platform with ArgoCD. It covers cluster validation, application health checks, Kubernetes object inspection, AWS authentication, and common troubleshooting steps for the GitOps deployment lifecycle.

---

## 1. Confirm you are connected to the correct EKS cluster

Start by verifying the active Kubernetes context:

```bash
kubectl config current-context
```

You want the context to point to the correct AWS EKS cluster. If the command returns an unexpected cluster or no value, update the kubeconfig.

### Check cluster reachability

```bash
kubectl get nodes
```

Expected result:

```text
NAME                        STATUS   ROLES    AGE
ip-10-0-1-45.ec2.internal   Ready    <none>   3h
```

If you see any of the following:

```text
No resources found
Unable to connect to the server
```

Then the cluster is either incorrect or kubeconfig is stale.

### Refresh kubeconfig

```bash
aws eks update-kubeconfig \
  --region <region> \
  --name <cluster-name>
```

Then confirm again:

```bash
kubectl config current-context
kubectl get nodes
```

---

## 2. Verify AWS credentials and cluster identity

Check the AWS identity used by your local environment:

```bash
aws sts get-caller-identity
```

This confirms whether your AWS session is authenticated and associated with the expected account.

Verify the EKS cluster is visible in the target region:

```bash
aws eks list-clusters --region us-east-1
```

If the AWS authentication plugin is failing, test it directly:

```bash
aws eks get-token \
  --region us-east-1 \
  --cluster-name <cluster-name>
```

If this command fails, the issue is usually related to:

- expired AWS credentials
- incorrect IAM role assignment
- wrong AWS region
- missing CLI configuration

---

## 3. Check whether ArgoCD applications exist

Verify the ArgoCD Application resources:

```bash
kubectl get applications -A
```

Expected output should look similar to:

```text
NAMESPACE   NAME             SYNC STATUS   HEALTH STATUS
argocd      user-service     Synced        Healthy
argocd      order-service    Synced        Healthy
argocd      monitoring       Synced        Healthy
```

If no Application resources are present, the root application or ApplicationSet may not have been applied successfully.

If you use the ArgoCD CLI, you can also verify with:

```bash
argocd app list
```

An `OutOfSync` or missing application state usually means the manifests were not reconciled yet.

---

## 4. Validate the ArgoCD control plane

Check the ArgoCD namespace:

```bash
kubectl get pods -n argocd
```

You should see components such as:

```text
argocd-server
argocd-repo-server
argocd-application-controller
argocd-applicationset-controller
```

If the namespace does not exist, ArgoCD may not have been installed or bootstrapped correctly.

Inspect the ArgoCD controller logs for reconciliation problems:

```bash
kubectl logs -n argocd deployment/argocd-application-controller
```

This often reveals:

- invalid Kustomize configuration
- missing namespaces
- invalid Git repository access
- manifest generation issues
- sync failures caused by bad YAML or resource naming

If needed, force a refresh:

```bash
kubectl annotate application <app-name> \
  -n argocd \
  argocd.argoproj.io/refresh=hard \
  --overwrite
```

Inspect the application state:

```bash
kubectl describe application <app-name> -n argocd
```

---

## 5. Check project namespaces

List all namespaces:

```bash
kubectl get ns
```

You should see namespaces similar to:

```text
argocd
user
order
payment
product
monitoring
```

If expected namespaces are missing, verify whether ArgoCD has created them from the applied manifests.

---

## 6. Verify deployments and service availability

Check all deployments across namespaces:

```bash
kubectl get deployments -A
```

Expected pattern:

```text
NAMESPACE   NAME             READY
order       order-service    1/1
user        user-service     1/1
```

If deployments exist but pods are not ready, inspect the deployment resource:

```bash
kubectl describe deployment <deployment-name> -n <namespace>
```

Check pods:

```bash
kubectl get pods -A
```

Common problem states:

- Pending
- ImagePullBackOff
- CrashLoopBackOff
- Init:Error

When a pod is unhealthy, inspect it:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

To view the previous container logs if the app restarted:

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

---

## 7. Common deployment troubleshooting

### Deployment exists but no pods are running

This often indicates a scheduling or image issue. Verify the deployment status and inspect events:

```bash
kubectl describe deployment <deployment-name> -n <namespace>
```

Look for problems such as:

- insufficient resources
- image pull failures
- invalid volume mounts
- readiness probe failures
- node pressure or taints

### Pods are stuck in Pending

Check node capacity and scheduling constraints:

```bash
kubectl get nodes
kubectl describe node <node-name>
```

### Image pull failures

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Typical causes include:

- private ECR image permissions
- wrong image name or tag
- registry authentication issue

### CrashLoopBackOff

```bash
kubectl logs <pod-name> -n <namespace>
```

Check whether the process is failing because of:

- application startup errors
- missing environment variables
- secret/config map issues
- database connectivity failures

---

## 8. Restart workloads after a failed sync or config change

To restart a specific deployment:

```bash
kubectl rollout restart deployment <deployment-name> -n <namespace>
```

For multiple services:

```bash
kubectl rollout restart deployment order-service -n <namespace>
kubectl rollout restart deployment payment-service -n <namespace>
kubectl rollout restart deployment product-service -n <namespace>
kubectl rollout restart deployment user-service -n <namespace>
```

Restart all deployments in a namespace:

```bash
kubectl rollout restart deployment -n <namespace>
```

Monitor rollout status:

```bash
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

---

## 9. Kyverno troubleshooting

If Kyverno is causing issues, start by checking its namespace and workloads:

```bash
kubectl get pods -n kyverno
```

Temporarily scale Kyverno down if needed:

```bash
kubectl scale deployment -n kyverno --all --replicas=0
```

Confirm pods are removed:

```bash
kubectl get pods -n kyverno
```

Check Helm release records:

```bash
kubectl get secret -n kyverno | grep sh.helm.release
```

If the Helm release metadata is corrupt, remove it:

```bash
kubectl delete secret -n kyverno sh.helm.release.v1.kyverno.v1
```

Disable Kyverno webhooks if needed:

```bash
kubectl get validatingwebhookconfiguration | grep kyverno
kubectl get mutatingwebhookconfiguration | grep kyverno
```

Delete problematic resources if they are not recoverable:

```bash
kubectl delete mutatingwebhookconfiguration \
  kyverno-policy-mutating-webhook-cfg \
  kyverno-resource-mutating-webhook-cfg \
  kyverno-verify-mutating-webhook-cfg

kubectl delete validatingwebhookconfiguration \
  kyverno-cel-exception-validating-webhook-cfg \
  kyverno-cleanup-validating-webhook-cfg \
  kyverno-exception-validating-webhook-cfg \
  kyverno-global-context-validating-webhook-cfg \
  kyverno-policy-validating-webhook-cfg \
  kyverno-resource-validating-webhook-cfg \
  kyverno-ttl-validating-webhook-cfg
```

Finally, confirm Kyverno has been removed cleanly:

```bash
kubectl get all -n kyverno
```

---

## 10. Monitoring and observability checks

Check whether Prometheus and Grafana pods are running:

```bash
kubectl get pods -n monitoring
```

Check exposed services:

```bash
kubectl get svc -n monitoring
```

Port-forward Grafana locally:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Open:

```text
http://localhost:3000
```

Retrieve the admin password:

```bash
kubectl get secret -n monitoring \
  kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode && echo
```

Port-forward Prometheus locally:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Open:

```text
http://localhost:9090
```

---

## 11. Final deployment validation checklist

Use this quick checklist after every deployment:

```bash
kubectl config current-context
kubectl get nodes
kubectl get applications -A
kubectl get pods -A
kubectl get deployments -A
kubectl get svc -A
kubectl get ns
```

At minimum, the environment should show:

- connected cluster context
- healthy nodes
- ArgoCD applications present
- expected namespaces created
- deployments ready
- pods running without crash loops
- platform services healthy

---

## 12. Troubleshooting summary

If the deployment is not working, follow this order:

1. Confirm the correct cluster and kubeconfig
2. Verify AWS identity and EKS access
3. Check ArgoCD application presence and sync state
4. Inspect ArgoCD controller logs
5. Check namespaces, deployments, and pods
6. Review pod logs and events
7. Restart the affected deployment if needed
8. Validate monitoring services for platform health
9. Resolve Kyverno or policy issues when relevant

This methodical approach quickly narrows the issue to either cluster access, ArgoCD reconciliation, Kubernetes resource health, or application runtime failures.
