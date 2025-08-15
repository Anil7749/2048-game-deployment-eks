# Project-one-deploy-2048-game-eks

# Deploy 2048 Game on Amazon EKS

**Project:** End-to-end deployment of the `2048` sample game onto an Amazon EKS cluster.

**Goal:** Create an EKS cluster, install AWS Load Balancer Controller (ALB), deploy the 2048 app, and expose it via an Ingress backed by an ALB.

---

## 🚀 Overview

Deploy the open-source 2048 web game to AWS EKS using:

* `eksctl` for cluster creation and OIDC (IRSA) setup
* `helm` for AWS Load Balancer Controller installation
* Kubernetes manifests for Deployment, Service, and Ingress

---

## 🧩 Architecture

```
User
  │
  └──> AWS ALB (internet-facing)
         │
         └──> EKS Cluster
               └──> Namespace: game-2048
                     ├─ Deployment (3 replicas)
                     ├─ Service (ClusterIP)
                     └─ Ingress (ALB)
```

---

## 🔧 Prerequisites

* AWS account with EKS/IAM/EC2/ELB permissions
* AWS CLI v2, `eksctl`, `kubectl`, `helm`
* Supported AWS region (e.g., `ap-south-1`)

---

## ⚙️ Deployment Steps

### 1. Environment Variables

```bash
export REGION=ap-south-1
export CLUSTER=eks-2048-demo
```

### 2. Create EKS Cluster

```bash
eksctl create cluster --name $CLUSTER --region $REGION --version 1.29 \
--nodegroup-name ng1 --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 4 --managed
aws eks update-kubeconfig --region $REGION --name $CLUSTER
```

### 3. Associate OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider --cluster $CLUSTER --region $REGION --approve
```

### 4. Create IAM Policy

```bash
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.1/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json
POLICY_ARN=$(aws iam list-policies --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy'].Arn | [0]" --output text)
```

### 5. Create IRSA Service Account

```bash
eksctl create iamserviceaccount --cluster $CLUSTER --region $REGION --namespace kube-system \
--name aws-load-balancer-controller --attach-policy-arn "$POLICY_ARN" --approve
```

### 6. Install AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system --set clusterName=$CLUSTER --set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller
```

### 7. Deploy 2048 App

Create `2048-manifests.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
        - name: app-2048
          image: public.ecr.aws/l6m2t8p7/docker-2048:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/name: app-2048
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service-2048
                port:
                  number: 80
```

Apply it:

```bash
kubectl apply -f 2048-manifests.yaml
```

### 8. Access the App

```bash
kubectl get ingress -n game-2048 -o wide
```

Open the ALB DNS in your browser.

---

## ✅ Verification Checklist

* Controller running in `kube-system`
* 3 pods for 2048 app are Ready
* Service exists and Ingress has ALB DNS
* ALB DNS opens the 2048 game

---

## 🛠 Troubleshooting

* ALB not created → Check controller logs & annotations
* Pods not starting → Check image architecture compatibility

---

## 🔁 Cleanup

```bash
kubectl delete -f 2048-manifests.yaml
helm uninstall aws-load-balancer-controller -n kube-system
eksctl delete cluster --name $CLUSTER --region $REGION
```

---
