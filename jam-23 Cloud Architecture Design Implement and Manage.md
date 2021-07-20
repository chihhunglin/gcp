# Cloud Architecture Design Implement and Manage

## Google Cloud Essential Skills: Challenge Lab
sudo apt-get update
sudo apt-get install apache2 -y
echo '<!doctype html><html><body><h1>Hello World!</h1></body></html>' | sudo tee /var/www/html/index.html

## Deploy a Compute Instance with a Remote Startup Script

## Configure a Firewall and a Startup Script with Deployment Manager
mkdir deployment_manager
cd deployment_manager
gsutil cp gs://spls/gsp302/* .

resources:
- type: compute.v1.instance
  name: vm-test
  properties:
    zone: {{ properties["zone"] }}
    machineType: https://www.googleapis.com/compute/v1/projects/{{ env["project"] }}/zones/{{ properties["zone"] }}/machineTypes/f1-micro
    disks:
    - deviceName: boot
      type: PERSISTENT
      boot: true
      autoDelete: true
      initializeParams:
        diskName: disk-{{ env["deployment"] }}
        sourceImage: https://www.googleapis.com/compute/v1/projects/debian-cloud/global/images/family/debian-9
    networkInterfaces:
    - network: https://www.googleapis.com/compute/v1/projects/{{ env["project"] }}/global/networks/default
      accessConfigs:
      - name: External NAT
        type: ONE_TO_ONE_NAT
    metadata:
      items:
      - key: startup-script
        value: |
          #!/bin/bash
          apt-get update
          apt-get install -y apache2
    tags:
      items:
      - http
    serviceAccounts:
    - email: <YOUR-SERVICE-ACCOUNT-EMAIL>
      scopes:
      - https://www.googleapis.com/auth/devstorage.read_only
      - https://www.googleapis.com/auth/logging.write
      - https://www.googleapis.com/auth/monitoring.write
      - https://www.googleapis.com/auth/servicecontrol
      - https://www.googleapis.com/auth/service.management.readonly
      - https://www.googleapis.com/auth/trace.append
- type: compute.v1.firewall
  name: default-allow-http
  properties:
    network: https://www.googleapis.com/compute/v1/projects/{{ env["project"] }}/global/networks/default
    targetTags: 
    - http
    allowed:
    - IPProtocol: tcp
      ports: 
      - '80'
    sourceRanges: 
    - 0.0.0.0/0

gcloud deployment-manager deployments create vm-test --config=qwiklabs.yaml

## Configure Secure RDP using a Windows Bastion Host
### https://chriskyfung.medium.com/qwiklab-logbook-configure-secure-rdp-using-a-windows-bastion-host-with-terraform-on-gcp-26d2311a35b3

gcloud compute reset-windows-password vm-bastionhost --user app_admin --zone us-central1-a

### IIS
https://www.rootusers.com/how-to-install-iis-in-windows-server-2016/

## Build and Deploy a Docker Image to a Kubernetes Cluster
export PROJECT_ID=$(gcloud info --format='value(config.project)')
gsutil cp gs://${PROJECT_ID}/echo-web.tar.gz .
tar -xvzf echo-web.tar.gz
cd echo-web
docker build -t echo-app:v1 .
docker tag echo-app:v1 gcr.io/${PROJECT_ID}/echo-app:v1
docker push gcr.io/${PROJECT_ID}/echo-app:v1

### https://chriskyfung.github.io/blog/qwiklabs/Build-and-Deploy-a-Docker-Image-to-a-Kubernetes-Cluster

gcloud container clusters get-credentials echo-cluster --zone us-central1-a
kubectl run echo-app --image=gcr.io/${PROJECT_ID}/echo-app:v1 --port 8000
kubectl expose deployment echo-app --name echo-web \
  --type LoadBalancer --port 80 --target-port 8000
kubectl get service echo-web

## Scale Out and Update a Containerized Application on a Kubernetes Cluster

docker build -t echo-app:v2 .
export PROJECT_ID=$(gcloud info --format='value(config.project)')
docker tag echo-app:v2 gcr.io/${PROJECT_ID}/echo-app:v2
docker push gcr.io/${PROJECT_ID}/echo-app:v2

## Migrate a MySQL Database to Google Cloud SQL

### In blog vm
mysqldump --databases wordpress -h localhost -u blogadmin -p \
--hex-blob --skip-triggers --single-transaction \
--default-character-set=utf8mb4 > wordpress.sql

export PROJECT_ID=$(gcloud info --format='value(config.project)')
gsutil mb gs://${PROJECT_ID}
gsutil cp ~/wordpress.sql gs://${PROJECT_ID}
