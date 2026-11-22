# Amazon EKS GitOps Platform

Terraform provisions an Amazon EKS development cluster in a two-AZ public VPC. Argo CD watches `k8s/` on `main` and reconciles a two-replica Nginx demo. An AWS load balancer serves a versioned HTML page on port 80.

| Directory | Responsibility |
| --- | --- |
| `terraform/` | VPC, public subnets, EKS control plane, and managed node group |
| `argocd/` | Argo CD Application with automated sync, pruning, and self-healing |
| `k8s/` | Demo namespace, page content, Deployment, and Service |

## Validate locally

```sh
cd terraform
terraform fmt -check
terraform init -backend=false
terraform validate
```

## Bootstrap an existing cluster

Once the Terraform stack is provisioned, run the command printed by `terraform output -raw update_kubeconfig_command` from `terraform/`. Install Argo CD in the `argocd` namespace using its [installation guide](https://argo-cd.readthedocs.io/en/stable/getting_started/). Make sure the demo manifests are on `main` in the repository configured in the Application. From the repository root, run:

```sh
kubectl apply -f argocd/application.yaml
kubectl -n argocd get application demo-web
kubectl -n demo-app rollout status deployment/demo-web
kubectl -n demo-app get service demo-web --watch
```

Open the Service's external hostname over HTTP once it appears. To demonstrate GitOps, change the version text in `k8s/configmap.yaml` and merge the change into `main`. The mounted ConfigMap refreshes in the running Pods after Kubernetes propagates the update.

## Cost and lifecycle

Public subnets avoid NAT gateway charges. The configuration starts with two On-Demand `t3.medium` nodes. Budget for the EKS control plane, EC2, EBS, public IPv4 addresses, the load balancer, and data transfer.

The project pins Kubernetes 1.31 as specified. AWS bills 1.31 at the extended-support rate until November 26, 2026. Check version availability before provisioning. [EKS version lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html) · [EKS pricing](https://aws.amazon.com/eks/pricing/)

For a short-lived demo, delete the Argo CD Application and the demo Service, wait for AWS to delete the load balancer, then destroy the Terraform stack.
