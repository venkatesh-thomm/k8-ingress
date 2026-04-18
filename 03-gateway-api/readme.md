# Gateway API with AWS Load Balancer Controller

## What is Gateway API and why does it exist?

Old way (Ingress) put everything in annotations — scheme, target type, certificates, WAF — all crammed into one resource. It worked but it was messy and gave no separation between who manages the load balancer vs who manages the app routing.

Gateway API splits those concerns into separate resources, each owned by a different team:

```
Platform team  →  GatewayClass, Gateway, LoadBalancerConfiguration
App team       →  TargetGroupConfiguration, HTTPRoute
```

---

## Kubernetes resource vs AWS resource — side by side

| Kubernetes Resource | AWS Resource Created | Who manages it |
|---|---|---|
| `GatewayClass` | Nothing — just tells controller which provider (ALB) | Platform team |
| `Gateway` | ALB + Security Group | Platform team |
| `Gateway.listeners` | ALB Listener (port 80 / 443) | Platform team |
| `LoadBalancerConfiguration` | ALB settings (scheme, IP type) | Platform team |
| `HTTPRoute` | ALB Listener Rule | App team |
| `TargetGroupConfiguration` | Target Group (ip mode) | App team |
| `Service` | ALB Target Group registration | App team |

---

## What each resource does — in plain AWS terms

**GatewayClass** — tells the controller which load balancer type to use. `controllerName: gateway.k8s.aws/alb` means "use AWS ALB". One per cluster.

**Gateway** — creates the actual ALB and listener (port 80/443) in AWS. Once `PROGRAMMED=True` you'll see the ALB DNS name in `kubectl get gateway`.

**LoadBalancerConfiguration** — ALB-level settings like scheme (internet-facing vs internal), IP type, WAF. In LBC v3.x annotations on Gateway are ignored — all settings must go here.

**HTTPRoute** — creates the ALB Listener Rule. Defines which path or hostname goes to which service. Also supports header-based routing and weighted routing for canary deployments.

**TargetGroupConfiguration** — controls how pods register in the ALB Target Group. `targetType: ip` registers pod IPs directly, so traffic goes straight to the pod without a NodePort hop.

### How they connect

```
User request
    ↓
Route53 / DNS
    ↓
ALB  ←────────────────  Gateway + LoadBalancerConfiguration
    ↓
ALB Listener (port 80)  ←──  Gateway.listeners
    ↓
ALB Listener Rule  ←────────  HTTPRoute
    ↓
Target Group  ←─────────────  TargetGroupConfiguration
    ↓
Pod IP directly
```

---

## File structure

```
03-gateway-api/
├── gatewayclass.yaml               # GatewayClass — one per cluster
├── loadbalancerconfiguration.yaml  # ALB settings (scheme, IP type)
├── gateway.yaml                    # ALB + Listener
├── app1/
│   └── manifest.yaml               # Deployment, Service, TargetGroupConfiguration, HTTPRoute
└── app2/
    └── manifest.yaml               # Deployment, Service, TargetGroupConfiguration, HTTPRoute
```

---

## Setup steps

### Step 1: Install Gateway API CRDs

CRDs teach Kubernetes about new resource types. Without them, Kubernetes has no idea what a `Gateway` or `HTTPRoute` is.

```bash
# Must be v1.5.0 — earlier versions missing TLSRoute v1 which LBC v3.x requires
kubectl apply --server-side=true \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml

# Verify — should see 8+ CRDs
kubectl get crd | grep gateway
```

Expected output:
```
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
tlsroutes.gateway.networking.k8s.io   ← must be v1 not v1alpha2
```

### Step 2: Install AWS-specific CRDs

These are extra CRDs that only exist for AWS — `LoadBalancerConfiguration`, `TargetGroupConfiguration`, `ListenerRuleConfiguration`.

```bash
kubectl apply \
  -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/config/crd/gateway/gateway-crds.yaml

# Verify TLSRoute is v1 (required by LBC v3.x)
kubectl get crd tlsroutes.gateway.networking.k8s.io -o yaml | grep "name: v1"
# Must show: name: v1
```

### Step 3: Create IAM Policy

LBC needs IAM permissions to create and manage ALBs, Target Groups, and Security Groups in your account.

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json
```

### Step 4: Create Service Account with IAM Role (IRSA)

This links a Kubernetes ServiceAccount to an IAM Role so the LBC pod can call AWS APIs.

```bash
eksctl create iamserviceaccount \
  --cluster=roboshop-dev \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::160885265516:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region=us-east-1

# If it says "already exists" with a stale CloudFormation stack:
# 1. Find the stack
aws cloudformation list-stacks --region us-east-1 \
  --stack-status-filter CREATE_COMPLETE \
  --query "StackSummaries[?contains(StackName,'aws-load-balancer-controller')].StackName" \
  --output text

# 2. Disable termination protection
aws cloudformation update-termination-protection \
  --no-enable-termination-protection \
  --stack-name eksctl-roboshop-dev-addon-iamserviceaccount-kube-system-aws-load-balancer-controller \
  --region us-east-1

# 3. Delete the stale stack
aws cloudformation delete-stack \
  --stack-name eksctl-roboshop-dev-addon-iamserviceaccount-kube-system-aws-load-balancer-controller \
  --region us-east-1

aws cloudformation wait stack-delete-complete \
  --stack-name eksctl-roboshop-dev-addon-iamserviceaccount-kube-system-aws-load-balancer-controller \
  --region us-east-1

# 4. Run eksctl create again

# Verify SA has the annotation
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml | grep "eks.amazonaws.com"
```

### Step 5: Install LBC via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

VPC_ID=$(aws ssm get-parameter --name /roboshop/dev/vpc_id \
  --region us-east-1 --query Parameter.Value --output text)

helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=roboshop-dev \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID \
  --set controllerConfig.featureGates.ALBGatewayAPI=true \
  --set controllerConfig.featureGates.NLBGatewayAPI=true

kubectl rollout status deployment aws-load-balancer-controller -n kube-system
kubectl get pods -n kube-system | grep aws-load-balancer
```

Note: `vpcId` is required for private clusters — without it the controller can't auto-discover the VPC.


### Step 6: Create GatewayClass

One per cluster. Tells Kubernetes that anything referencing `aws-alb` should be handled by the AWS LBC.

```bash
kubectl apply -f gatewayclass.yaml

# Wait for ACCEPTED=True before moving on
kubectl get gatewayclass aws-alb
```

### Step 7: Create LoadBalancerConfiguration

ALB-level settings. This is where you set internet-facing vs internal. Annotations on the Gateway itself are ignored in LBC v3.x — must use this resource.

```bash
kubectl apply -f loadbalancerconfiguration.yaml
```

### Step 8: Create Gateway

This actually creates the ALB in AWS. Takes 1-2 minutes. The ADDRESS field shows the ALB DNS once it's ready.

```bash
kubectl apply -f gateway.yaml

# Watch until PROGRAMMED=True and ADDRESS appears
kubectl get gateway -w
```

### Step 9: Deploy app1

```bash
kubectl apply -f app1/manifest.yaml

# Verify route attached to gateway
kubectl get httproute app1-route
```

### Step 10: Deploy app2

```bash
kubectl apply -f app2/manifest.yaml

kubectl get httproute app2-route
```

### Step 11: Test

```bash
ALB=$(kubectl get gateway my-gateway -o jsonpath='{.status.addresses[0].value}')
echo "ALB DNS: $ALB"

# Test by hostname (make sure DNS points to ALB)
curl -H "Host: app1.daws88s.online" http://$ALB
curl -H "Host: app2.daws88s.online" http://$ALB

kubectl get gatewayclass,gateway,httproute,targetgroupconfiguration
```

---

## Cleanup

```bash
# App resources
kubectl delete -f app1/manifest.yaml
kubectl delete -f app2/manifest.yaml

# Shared resources — Gateway deletion triggers ALB deletion in AWS
kubectl delete -f gateway.yaml
kubectl delete -f loadbalancerconfiguration.yaml
kubectl delete -f gatewayclass.yaml

# LBC
helm uninstall aws-load-balancer-controller -n kube-system

# Service account + IAM role
eksctl delete iamserviceaccount \
  --name aws-load-balancer-controller \
  --namespace kube-system \
  --cluster roboshop-dev \
  --region us-east-1

# IAM policy
aws iam delete-policy \
  --policy-arn arn:aws:iam::160885265516:policy/AWSLoadBalancerControllerIAMPolicy

# CRDs
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/config/crd/gateway/gateway-crds.yaml
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
```
