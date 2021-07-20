# Set Up and Configure a Cloud Environment in Google Cloud

## Cloud IAM: Qwik Start
Done before

## Introduction to SQL for BigQuery and Cloud SQL
Done before

## Multiple VPC Networks

### Create the privatenet network
gcloud compute networks create privatenet --subnet-mode=custom
gcloud compute networks subnets create privatesubnet-us --network=privatenet --region=us-central1 --range=172.16.0.0/24
gcloud compute networks subnets create privatesubnet-eu --network=privatenet --region=europe-west4 --range=172.20.0.0/20
gcloud compute networks list
gcloud compute networks subnets list --sort-by=NETWORK

### Create the firewall rules for privatenet
gcloud compute firewall-rules create privatenet-allow-icmp-ssh-rdp --direction=INGRESS --priority=1000 --network=privatenet --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
gcloud compute firewall-rules list --sort-by=NETWORK

###　Create the privatenet-us-vm instance
gcloud compute instances create privatenet-us-vm --zone=us-central1-f --machine-type=n1-standard-1 --subnet=privatesubnet-us
gcloud compute instances list --sort-by=ZONE

## Managing Deployments Using Kubernetes Engine
gcloud config set compute/zone us-central1-a
gsutil -m cp -r gs://spls/gsp053/orchestrate-with-kubernetes .
cd orchestrate-with-kubernetes/kubernetes
gcloud container clusters create bootcamp --num-nodes 5 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"

### Learn about the deployment object
kubectl explain deployment
kubectl explain deployment --recursive
kubectl explain deployment.metadata.name

### Create a deployment
kubectl create -f deployments/auth.yaml
kubectl get deployments
kubectl get replicasets
kubectl get pods
kubectl create -f services/auth.yaml
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
kubectl get services frontend
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`

### Scale a Deployment
kubectl explain deployment.spec.replicas
kubectl scale deployment hello --replicas=5
kubectl get pods | grep hello- | wc -l
kubectl scale deployment hello --replicas=3
kubectl get pods | grep hello- | wc -l

### Rolling update
kubectl edit deployment hello
kubectl get replicaset
kubectl rollout history deployment/hello
kubectl rollout pause deployment/hello
kubectl rollout status deployment/hello
kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
kubectl rollout resume deployment/hello
kubectl rollout status deployment/hello
kubectl rollout undo deployment/hello
kubectl rollout history deployment/hello
kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

### Canary deployments
cat deployments/hello-canary.yaml
kubectl create -f deployments/hello-canary.yaml
kubectl get deployments
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version

## Set Up and Configure a Cloud Environment in Google Cloud: Challenge Lab
Task 1: Create development VPC manually

gcloud compute networks create griffin-dev-vpc --project=qwiklabs-gcp-04-40818ebf65df --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional

gcloud compute networks subnets create griffin-dev-wp --project=qwiklabs-gcp-04-40818ebf65df --range=192.168.16.0/20 --network=griffin-dev-vpc --region=us-east1

gcloud compute networks subnets create griffin-dev-mgmt --project=qwiklabs-gcp-04-40818ebf65df --range=192.168.32.0/20 --network=griffin-dev-vpc --region=us-east1

Task 2: Create production VPC manually

gcloud compute networks create griffin-prod-vpc --project=qwiklabs-gcp-04-40818ebf65df --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional

gcloud compute networks subnets create griffin-prod-wp --project=qwiklabs-gcp-04-40818ebf65df --range=192.168.48.0/20 --network=griffin-prod-vpc --region=us-east1

gcloud compute networks subnets create griffin-prod-mgmt --project=qwiklabs-gcp-04-40818ebf65df --range=192.168.64.0/20 --network=griffin-prod-vpc --region=us-east1

Task 3: Create bastion host firewall rule: Field Value Name: allow-bastion-dev-ssh Network: griffin-dev-vpc Targets: bastion Source IP ranges: 192.168.32.0/20 Protocols and ports: tcp: 22

Field Value Name: allow-bastion-prod-ssh Network: griffin-prod-vpc Targets: bastion Source IP ranges: 192.168.64.0/20 Protocols and ports: tcp: 22

gcloud compute --project=qwiklabs-gcp-04-40818ebf65df firewall-rules create allow-bastion-dev-ssh --direction=INGRESS --priority=1000 --network=griffin-dev-vpc --action=ALLOW --rules=tcp:22 --source-ranges=192.168.32.0/20 --target-tags=bastion

gcloud compute --project=qwiklabs-gcp-04-40818ebf65df firewall-rules create allow-bastion-prod-ssh --direction=INGRESS --priority=1000 --network=griffin-prod-vpc --action=ALLOW --rules=tcp:22 --source-ranges=192.168.64.0/20 --target-tags=bastion

Task 4: Create and configure Cloud SQL Instance

gcloud sql connect griffin-dev-db --user=root --quiet

CREATE DATABASE wordpress; GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%" IDENTIFIED BY "stormwind_rules"; FLUSH PRIVILEGES;

Task 5: Create Kubernetes cluster

Task 6: Prepare the Kubernetes cluster

gsutil cp -r gs://cloud-training/gsp321/wp-k8s ~/
gcloud container clusters get-credentials griffin-dev --zone=us-east1-b
gcloud container clusters get-credentials griffin-dev --zone=us-east1-b kubectl apply -f wp-env.yaml

gcloud iam service-accounts keys create key.json --iam-account=cloud-sql-proxy@qwiklabs-gcp-04-40818ebf65df.iam.gserviceaccount.com 
kubectl create secret generic cloudsql-instance-credentials --from-file key.json

Task 7: Create a WordPress deployment qwiklabs-gcp-03-5130e499313f:us-east1:griffin-dev-db 
kubectl create -f wp-deployment.yaml 
kubectl delete -f wp-deployment.yaml
kubectl create -f wp-service.yaml

Task 8: Enable monitoring

Monitoring > Uptime checks > CREATE UPTIME CHECK

Field Value Title WordPress Uptime Check Type HTTP Resource Type URL Hostname 35.190.165.96 Path /

Task 9: Provide access for an additional engineer

https://cdn.qwiklabs.com/tZbcgn7I6buxMKPfWQmXyf7RGr%2FVhzhssCv2Seav9AI%3D