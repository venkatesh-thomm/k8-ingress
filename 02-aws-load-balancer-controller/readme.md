# Version 2: AWS Load Balancer Controller (Current Way)

## How It Works
- AWS Load Balancer Controller runs inside EKS and watches Ingress resources
- When it sees an Ingress with `ingressClassName: alb`, it **provisions a real AWS ALB**
- Traffic flow: Internet → AWS ALB → directly to pod IP (target-type: ip, bypasses Service)
- `group.name: roboshop` → both app1 and app2 share **one ALB** (saves cost)

## Key Difference from Nginx Ingress
| | Nginx Ingress | AWS LB Controller |
|---|---|---|
| Controller runs | Inside cluster (pod) | Inside cluster (pod) |
| Load balancer | K8s LoadBalancer Service (1 NLB) | AWS ALB per group |
| Routing | nginx config inside pod | AWS ALB rules |
| Vendor config | nginx annotations | alb.ingress annotations |

## Prerequisites
Install OIDC provider:
```bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster roboshop \
    --approve
```

Create IAM Policy:
```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.2.1/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```

Create service account:
```bash
eksctl create iamserviceaccount \
--cluster=roboshop \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-east-1 \
--approve
```

Install controller via Helm:
```bash
helm repo add eks https://aws.github.io/eks-charts

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=roboshop \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

## Apply Manifests
```bash
kubectl apply -f app1/manifest.yaml
kubectl apply -f app2/manifest.yaml
```
