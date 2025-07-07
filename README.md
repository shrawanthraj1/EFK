# EFK Stack on EKS (Elasticsearch, Fluent Bit, Kibana)

This guide provides a step-by-step approach to setting up the **EFK stack** (Elasticsearch, Fluent Bit, Kibana) in an **Amazon EKS (Elastic Kubernetes Service)** environment. The EFK stack is widely used for centralized logging in Kubernetes clusters.

---

## 📚 Components Overview

- **Elasticsearch**: Stores and indexes log data for easy retrieval.
- **Fluent Bit**: Lightweight log forwarder that collects logs and ships them to Elasticsearch.
- **Kibana**: A visualization tool to explore and analyze logs stored in Elasticsearch.

---

## 🛠️ Prerequisites

Ensure the following tools are installed and configured:

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [eksctl](https://eksctl.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

Configure AWS credentials:

## 🚀 Step-by-Step Setup
## Step 1: Create EKS Cluster

```bash

eksctl create cluster \
  --name=observability \
  --region=us-east-1 \
  --zones=us-east-1a,us-east-1b \
  --without-nodegroup

```
## Associate IAM OIDC provider:
```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster observability \
  --approve

```
## Create managed node group:
```bash
eksctl create nodegroup \
  --cluster=observability \
  --region=us-east-1 \
  --name=observability-ng-private \
  --node-type=t3.medium \
  --nodes-min=2 \
  --nodes-max=3 \
  --node-volume-size=20 \
  --managed \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access \
  --node-private-networking

```
## Update your kubeconfig:
```bash
aws eks update-kubeconfig --name observability
```
## Step 2: Create IAM Role for EBS CSI Driver
```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster observability \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```
## Retrieve the IAM Role ARN:
```bash
ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text)
```
## Deploy EBS CSI Driver:
```bash
eksctl create addon \
  --cluster observability \
  --name aws-ebs-csi-driver \
  --version latest \
  --service-account-role-arn $ARN \
  --force
```
## Step 3: Setup Logging Namespace
```bash
kubectl create namespace logging
```
## Step 4: Install Elasticsearch
## Add Elastic Helm repo:
```bash
helm repo add elastic https://helm.elastic.co
```
## Install Elasticsearch:
```bash
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --set replicas=1 \
  --set volumeClaimTemplate.storageClassName=gp2 \
  --set persistence.labels.enabled=true
```
## Step 5: Retrieve Elasticsearch Credentials
```bash
# Username
kubectl get secret elasticsearch-master-credentials -n logging \
  -o jsonpath='{.data.username}' | base64 -d && echo

# Password
kubectl get secret elasticsearch-master-credentials -n logging \
  -o jsonpath='{.data.password}' | base64 -d && echo
```
## Step 6: Install Kibana
```bash
helm install kibana elastic/kibana \
  --namespace logging \
  --set service.type=LoadBalancer
```
## Step 7: Install Fluent Bit
📝 Update HTTP_Passwd in your fluentbit-values.yaml file with the password retrieved in Step 5.
```bash
helm repo add fluent https://fluent.github.io/helm-charts

helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  -f fluentbit-values.yaml
```
## Step 8: Deploy Application for Logging
Review Kubernetes manifests located in /kubernetes-manifest.
```bash
kubectl create ns dev

kubectl apply -k kubernetes-manifest/
```
## 📊 Accessing Kibana
 1. Retrieve the external DNS of the Kibana service:
```bash
kubectl get svc kibana -n logging
```
 2. Open Kibana in a browser using:
```bash
http://<LOAD_BALANCER_DNS_NAME>:5601
```
 3. Log in with the credentials retrieved from Step 5.
 4. Create a new Data View in Kibana to start visualizing your logs.
## ✅ Conclusion
You have successfully:
- Created an EKS cluster with private nodes.
- Deployed the EBS CSI driver with IAM permissions.
- Installed and configured the EFK stack (Elasticsearch, Fluent Bit, Kibana).
- Deployed sample applications for log collection.
- Verified and visualized logs via Kibana.
  
For any issues, ensure resources are up and running with kubectl get all -n logging.

<img width="959" alt="Screenshot 2025-06-23 185405" src="https://github.com/user-attachments/assets/38dee039-11cb-405d-97e3-5179eff9b947" />

<img width="949" alt="Screenshot 2025-06-23 192030" src="https://github.com/user-attachments/assets/1073bd91-7d90-4bcc-8a1f-541964ccc9ca" />

<img width="944" alt="Screenshot 2025-06-23 192130" src="https://github.com/user-attachments/assets/93f3697d-ce5b-4755-92a6-d383f478dd99" />
