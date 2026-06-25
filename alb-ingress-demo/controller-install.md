# Installing AWS Load Balancer Controller on EKS

This guide details the complete, production-grade installation process for the **AWS Load Balancer Controller** on an Amazon EKS cluster. This controller manages AWS Application Load Balancers (ALBs) dynamically when you deploy Ingress resources.

---

## Prerequisites

Before beginning, ensure you have:
* An active EKS Cluster.
* `kubectl` configured to access your cluster.
* `helm` (v3+) installed.
* `aws` CLI installed and configured with administrative permissions.
* `eksctl` CLI installed (recommended for simplified IAM role association).

---

## Step 1: Enable IAM OIDC Provider
EKS clusters require an IAM OpenID Connect (OIDC) provider to associate IAM roles with Kubernetes service accounts (IRSA).

1. Check if an OIDC provider is already associated with your cluster:
   ```bash
   aws eks describe-cluster --name <CLUSTER_NAME> --query "cluster.identity.oidc.issuer" --output text
   ```
   *Expected output URL structure: `https://oidc.eks.<region>.amazonaws.com/id/<OIDC_PROVIDER_ID>`*

2. List your IAM OIDC providers to see if this ID is already registered:
   ```bash
   aws iam list-open-id-connect-providers | grep <OIDC_PROVIDER_ID>
   ```

3. If no provider exists, associate it using `eksctl`:
   ```bash
   eksctl utils associate-iam-oidc-provider --cluster <CLUSTER_NAME> --approve
   ```

---

## Step 2: Create IAM Policy
The controller needs permissions to call AWS APIs (like EC2, Elastic Load Balancing, ACM, WAF) to configure resources.

1. Download the official IAM policy document (v2.7.2 is the latest stable version):
   ```bash
   curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
   ```

2. Create the IAM policy in your AWS Account:
   ```bash
   aws iam create-policy \
       --policy-name AWSLoadBalancerControllerIAMPolicy \
       --policy-document file://iam_policy.json
   ```
   *Note the Policy ARN returned in the output (e.g., `arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy`).*

---

## Step 3: Create Service Account & IAM Role
Use `eksctl` to automatically create a Kubernetes Service Account, create an IAM Role with the policy attached, and set up the trust relationship.

Run the following command (replace `<CLUSTER_NAME>` and `<ACCOUNT_ID>`):
```bash
eksctl create iamserviceaccount \
  --cluster=<CLUSTER_NAME> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

---

## Step 4: Install Target CRDs
Install the Custom Resource Definitions (CRDs) required for the controller:

```bash
kubectl apply -k "github.com/aws/kubernetes-sigs/aws-load-balancer-controller/config/crd?ref=v2.7.2"
```

---

## Step 5: Install AWS Load Balancer Controller via Helm

1. Add the Amazon EKS Helm repository:
   ```bash
   helm repo add eks https://aws.github.io/eks-charts
   ```

2. Update your local Helm charts:
   ```bash
   helm repo update
   ```

3. Install the Helm chart:
   ```bash
   helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
     -n kube-system \
     --set clusterName=<CLUSTER_NAME> \
     --set serviceAccount.create=false \
     --set serviceAccount.name=aws-load-balancer-controller
```

> [!NOTE]
> In target-type `ip` mode (which this project uses), the pods must run in a VPC subnets configuration where they have direct IP reachability. The controller will query the EKS API and AWS VPC endpoints to register pod IPs directly with target groups.

---

## Step 6: Verify Installation

Check if the controller pods are running successfully:
```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

### Expected Output
```text
NAME                                            READY   STATUS    RESTARTS   AGE
aws-load-balancer-controller-75865c9578-8prwz   1/1     Running   0          45s
aws-load-balancer-controller-75865c9578-f7sjw   1/1     Running   0          45s
```

Check the logs to verify no permission errors:
```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller --tail=100
```
