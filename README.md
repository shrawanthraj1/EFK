EFK Stack (Elasticsearch, Fluentbit, Kibana)
------------------------------------------------------------------------------------------------------------------------------------------------
EFK is a popular logging stack used to collect, store, and analyze logs in Kubernetes.
Elasticsearch: Stores and indexes log data for easy retrieval.
Fluentbit: A lightweight log forwarder that collects logs from different sources and sends them to Elasticsearch.
Kibana: A visualization tool that allows users to explore and analyze logs stored in Elasticsearch.


Step-by-Step Setup
--------------------------------------------------------------------------------------------------------------------------------------------------
Step 1: Create EKS Cluster

Prerequisites
Download and Install AWS Cli.
Setup and configure AWS CLI using the aws configure command.
Install and configure eksctl.
Install and configure kubectl.

eksctl create cluster --name=observability \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup
                      
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster observability \
    --approve

eksctl create nodegroup --cluster=observability \
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

# Update ./kube/config file
aws eks update-kubeconfig --name observability

<img width="749" alt="Screenshot 2025-06-23 170444" src="https://github.com/user-attachments/assets/1311bb5b-e250-4b83-80c8-a7c0dc11271a" />

2) Create IAM Role for Service Account

eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster observability \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
    
This command creates an IAM role for the EBS CSI controller.
IAM role allows EBS CSI controller to interact with AWS resources, specifically for managing EBS volumes in the Kubernetes cluster.
We will attach the Role with service account

<img width="795" alt="Screenshot 2025-06-23 170553" src="https://github.com/user-attachments/assets/7c7db4cc-9ff6-41b7-9b9c-1d1fc3b76f9f" />

3) Retrieve IAM Role ARN

$ARN = aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text

Command retrieves the ARN of the IAM role created for the EBS CSI controller service account.

4) Deploy EBS CSI Driver

eksctl create addon --cluster observability --name aws-ebs-csi-driver --version latest \
    --service-account-role-arn $ARN --force

<img width="956" alt="Screenshot 2025-06-23 171152" src="https://github.com/user-attachments/assets/8eb33c24-8667-42d4-acc1-c964086d742e" />
   

Above command deploys the AWS EBS CSI driver as an addon to your Kubernetes cluster.
It uses the previously created IAM service account role to allow the driver to manage EBS volumes securely.

5) Create Namespace for Logging

kubectl create namespace logging

6) Install Elasticsearch on K8s

helm repo add elastic https://helm.elastic.co

helm install elasticsearch \
 --set replicas=1 \
 --set volumeClaimTemplate.storageClassName=gp2 \
 --set persistence.labels.enabled=true elastic/elasticsearch -n logging

Installs Elasticsearch in the logging namespace.
It sets the number of replicas, specifies the storage class, and enables persistence labels to ensure data is stored on persistent volumes.

<img width="797" alt="Screenshot 2025-06-23 172951" src="https://github.com/user-attachments/assets/368a5d24-37a7-45a0-89a7-085ba83f8732" />

7) Retrieve Elasticsearch Username & Password

# for username
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
# for password
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d

Retrieves the password for the Elasticsearch cluster's master credentials from the Kubernetes secret.
The password is base64 encoded, so it needs to be decoded before use.
Note: Please write down the password for future reference

<img width="791" alt="Screenshot 2025-06-23 173657" src="https://github.com/user-attachments/assets/f32a77a0-1797-4915-8690-14f5dda811ab" />

8) Install Kibana

helm install kibana --set service.type=LoadBalancer elastic/kibana -n logging

Kibana provides a user-friendly interface for exploring and visualizing data stored in Elasticsearch.
It is exposed as a LoadBalancer service, making it accessible from outside the cluster.

9) Install Fluentbit with Custom Values/Configurations

Note: Please update the HTTP_Passwd field in the fluentbit-values.yml file with the password retrieved earlier in step 6.

helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit -f fluentbit-values.yaml -n logging

<img width="959" alt="Screenshot 2025-06-23 185405" src="https://github.com/user-attachments/assets/38dee039-11cb-405d-97e3-5179eff9b947" />

10) Kubernetes manifest

we are deploying an app using kubernetes-manifests to manage the app logs.
Review the Kubernetes manifest files located in /kubernetes-manifest.
Apply the Kubernetes manifest files to your cluster by running:

kubectl create ns dev
kubectl apply -k kubernetes-manifest/

<img width="916" alt="Screenshot 2025-06-23 191842" src="https://github.com/user-attachments/assets/7aa47de4-be5a-4a3e-8e34-6b3dd135efc4" />

11) Creating Kibana Dashboards

<img width="949" alt="Screenshot 2025-06-23 192030" src="https://github.com/user-attachments/assets/1073bd91-7d90-4bcc-8a1f-541964ccc9ca" />


<img width="944" alt="Screenshot 2025-06-23 192130" src="https://github.com/user-attachments/assets/93f3697d-ce5b-4755-92a6-d383f478dd99" />

Conclusion
------------------------------------------------------------------------------------------------------------------------------------------
We have successfully installed the EFK stack in our Kubernetes cluster, which includes Elasticsearch for storing logs, Fluentbit for collecting and forwarding logs, and Kibana for visualizing logs.
To verify the setup, access the Kibana dashboard by entering the `LoadBalancer DNS name followed by :5601 in your browser.
http://LOAD_BALANCER_DNS_NAME:5601
Use the username and password retrieved in step 7 to log in.
Once logged in, create a new data view in Kibana and explore the logs collected from your Kubernetes cluster.
