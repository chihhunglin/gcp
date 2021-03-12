# Set Up and Configure a Cloud Environment in Google Cloud

## Cloud IAM: Qwik Start
done before

## Introduction to SQL for BigQuery and Cloud SQL
gcloud sql connect  qwiklabs-demo --user=root
CREATE DATABASE bike;
USE bike;
CREATE TABLE london1 (start_station_name VARCHAR(255), num INT);
USE bike;
CREATE TABLE london2 (end_station_name VARCHAR(255), num INT);
SELECT * FROM london1;
SELECT * FROM london2;

## Multiple VPC Networks

![gcp_vpc.png](gcp_vpc.png)

Create custom mode VPC networks with firewall rules
Create VM instances using Compute Engine
Explore the connectivity for VM instances across VPC networks
Create a VM instance with multiple network interfaces

gcloud compute networks create managementnet --project=qwiklabs-gcp-03-485b92ae14a5 --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional

gcloud compute networks subnets create managementsubnet-us --project=qwiklabs-gcp-03-485b92ae14a5 --range=10.130.0.0/20 --network=managementnet --region=us-central1

gcloud compute networks create privatenet --subnet-mode=custom
gcloud compute networks subnets create privatesubnet-us --network=privatenet --region=us-central1 --range=172.16.0.0/24
gcloud compute networks subnets create privatesubnet-eu --network=privatenet --region=europe-west4 --range=172.20.0.0/20
gcloud compute networks list

gcloud compute networks subnets list --sort-by=NETWORK

gcloud compute --project=qwiklabs-gcp-03-485b92ae14a5 firewall-rules create managementnet-allow-icmp-ssh-rdp --direction=INGRESS --priority=1000 --network=managementnet --action=ALLOW --rules=tcp:22,tcp:3389,icmp --source-ranges=0.0.0.0/0

gcloud compute firewall-rules create privatenet-allow-icmp-ssh-rdp --direction=INGRESS --priority=1000 --network=privatenet --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0
gcloud compute firewall-rules list --sort-by=NETWORK

gcloud compute instances create privatenet-us-vm --zone=us-central1-c --machine-type=n1-standard-1 --subnet=privatesubnet-us
gcloud compute instances list --sort-by=ZONE


## Set Up and Configure a Cloud Environment in Google Cloud: Challenge Lab
Creating and using VPCs and subnets
Creating a Kubernetes cluster
Configuring and launching a Kubernetes deployment and service
Setting up stackdriver monitoring
Configuring an IAM role for an account

Task 1: Create development VPC manually

gcloud compute networks create griffin-dev-vpc --project=qwiklabs-gcp-03-5130e499313f --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional

gcloud compute networks subnets create griffin-dev-wp --project=qwiklabs-gcp-03-5130e499313f --range=192.168.16.0/20 --network=griffin-dev-vpc --region=us-east1

gcloud compute networks subnets create griffin-dev-mgmt --project=qwiklabs-gcp-03-5130e499313f --range=192.168.32.0/20 --network=griffin-dev-vpc --region=us-east1

Task 2: Create production VPC manually

gcloud compute networks create griffin-prod-vpc --project=qwiklabs-gcp-03-5130e499313f --subnet-mode=custom --mtu=1460 --bgp-routing-mode=regional

gcloud compute networks subnets create griffin-prod-wp --project=qwiklabs-gcp-03-5130e499313f --range=192.168.48.0/20 --network=griffin-prod-vpc --region=us-east1

gcloud compute networks subnets create griffin-prod-mgmt --project=qwiklabs-gcp-03-5130e499313f --range=192.168.64.0/20 --network=griffin-prod-vpc --region=us-east1


Task 3: Create bastion host
firewall rule:
Field	Value
Name:	allow-bastion-dev-ssh
Network:	griffin-dev-vpc
Targets:	bastion
Source IP ranges:	192.168.32.0/20
Protocols and ports:	tcp: 22

Field	Value
Name:	allow-bastion-prod-ssh
Network:	griffin-prod-vpc
Targets:	bastion
Source IP ranges:	192.168.64.0/20
Protocols and ports:	tcp: 22

gcloud compute --project=qwiklabs-gcp-03-5130e499313f firewall-rules create allow-bastion-dev-ssh --direction=INGRESS --priority=1000 --network=griffin-dev-vpc --action=ALLOW --rules=tcp:22 --source-ranges=192.168.32.0/20 --target-tags=bastion

gcloud compute --project=qwiklabs-gcp-03-5130e499313f firewall-rules create allow-bastion-prod-ssh --direction=INGRESS --priority=1000 --network=griffin-prod-vpc --action=ALLOW --rules=tcp:22 --source-ranges=192.168.64.0/20 --target-tags=bastion

Task 4: Create and configure Cloud SQL Instance

gcloud sql connect griffin-dev-db --user=root --quiet

CREATE DATABASE wordpress;
GRANT ALL PRIVILEGES ON wordpress.* TO "wp_user"@"%" IDENTIFIED BY "stormwind_rules";
FLUSH PRIVILEGES;

Task 5: Create Kubernetes cluster

Task 6: Prepare the Kubernetes cluster

gsutil cp -r gs://cloud-training/gsp321/wp-k8s ~/

gcloud container clusters get-credentials griffin-dev --zone=us-east1-b
kubectl apply -f wp-env.yaml

gcloud iam service-accounts keys create key.json \
    --iam-account=cloud-sql-proxy@qwiklabs-gcp-03-5130e499313f.iam.gserviceaccount.com
kubectl create secret generic cloudsql-instance-credentials \
    --from-file key.json

Task 7: Create a WordPress deployment
qwiklabs-gcp-03-5130e499313f:us-east1:griffin-dev-db
kubectl create -f wp-deployment.yaml
kubectl delete -f wp-deployment.yaml

kubectl create -f wp-service.yaml

Task 8: Enable monitoring

Monitoring > Uptime checks > CREATE UPTIME CHECK

Field	Value
Title	WordPress Uptime
Check Type	HTTP
Resource Type	URL
Hostname	34.74.161.234
Path	/

Task 9: Provide access for an additional engineer


https://cdn.qwiklabs.com/tZbcgn7I6buxMKPfWQmXyf7RGr%2FVhzhssCv2Seav9AI%3D
