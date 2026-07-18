#  GitOps-Driven Microservices Architecture on AWS EKS

##  Project Overview
This project demonstrates the deployment of a complex, 10-tier microservices e-commerce application (Stan's Robot Shop) on **Amazon Elastic Kubernetes Service (EKS)**. 

Instead of traditional manual deployments, this project implements a modern **GitOps** workflow using **ArgoCD** and **Helm**. The infrastructure is provisioned using Infrastructure as Code (IaC) principles, with advanced AWS networking and storage controllers handling traffic routing and persistent state.

##  Architecture & Tech Stack
*   **Cloud Provider:** Amazon Web Services (AWS)
*   **Container Orchestration:** Kubernetes (Amazon EKS)
*   **Infrastructure Provisioning:** `eksctl`
*   **CI/CD & GitOps:** ArgoCD
*   **Package Management:** Helm
*   **Networking:** AWS Application Load Balancer (ALB) Controller, Ingress
*   **Storage:** AWS EBS CSI Driver, custom `gp3` StorageClasses
*   **Application Stack:** React (Frontend), Node.js, Python, Java, Go (Microservices), MongoDB, Redis, MySQL (Databases), RabbitMQ (Messaging).

---

##  Key Features & Technical Implementations

1.  **GitOps Deployment Strategy:** Deployed the entire microservices stack without running a single manual `kubectl apply` or `helm install` for the application code. ArgoCD continuously monitored the source repository and synchronized the cluster state.
2.  **Dynamic AWS Networking:** Configured the AWS Load Balancer Controller using IAM Roles for Service Accounts (IRSA) to automatically provision an internet-facing ALB based on Kubernetes Ingress rules.
3.  **Stateful Storage Management:** Overcame default EKS storage limitations by deploying the AWS EBS CSI Driver and mapping a custom `gp3` storage class to legacy `standard` PersistentVolumeClaims (PVCs) for stateful applications like Redis and MongoDB.
4.  **Cluster Auto-Scaling Awareness:** Diagnosed IP exhaustion and node resource limits (`Too many pods` errors) and successfully scaled EC2 node groups to accommodate the microservices footprint.

---

##  Step-by-Step Deployment Guide

### 1. Provision the EKS Cluster
Create a cluster using `eksctl` with a defined node group.
```bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: Application-Cluster
  region: us-east-1
managedNodeGroups:
  - name: standard-workers
    instanceType: t3.small
    minSize: 2
    maxSize: 5

```

```bash
eksctl create cluster -f cluster.yaml

```

*(Note: Cluster was later scaled to 4 nodes to ensure sufficient Pod IP/ENI availability).*

### 2. Configure IAM OIDC & Infrastructure Controllers

Enable IRSA to allow Kubernetes pods to securely interact with AWS APIs.

```bash
eksctl utils associate-iam-oidc-provider --cluster Application-Cluster --approve

```

**AWS EBS CSI Driver:**

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster Application-Cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --role-name AmazonEKS_EBS_CSI_DriverRole

eksctl create addon --name aws-ebs-csi-driver --cluster Application-Cluster --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/AmazonEKS_EBS_CSI_DriverRole --force

```

**AWS Load Balancer Controller:**

```bash
# Policy creation and IAM association omitted for brevity. 
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=Application-Cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

```

### 3. Resolve Stateful Storage Dependencies

The Helm chart requires a `standard` storage class, which EKS v1.30+ does not provide by default. A custom `gp3` storage class was implemented to fulfill the PersistentVolumeClaims.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"

```

### 4. GitOps Deployment via ArgoCD

Install ArgoCD and apply the GitOps Application manifest to synchronize the cluster with the upstream Helm repository.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: robot-shop
  namespace: argocd
spec:
  project: default
  source:
    repoURL: '[https://github.com/instana/robot-shop.git]'
    path: K8s/helm
    targetRevision: master
  destination:
    server: '[https://kubernetes.default.svc](https://kubernetes.default.svc)'
    namespace: robot-shop
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

```

### 5. Expose the Application

Deploy an Ingress resource to trigger the ALB Controller and expose the frontend to the public internet.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: robot-shop-ingress
  namespace: robot-shop
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080

```

Retrieve the DNS name:

```bash
kubectl get ingress -n robot-shop

```
*Disclaimer: The dummy application (Stan's Robot Shop) used in this deployment is the property of Instana and is used here strictly for educational and demonstration purposes.*
