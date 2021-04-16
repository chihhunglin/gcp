# Google Cloud Solutions I Scaling Your Infrastructure

## Autoscaling an Instance Group with Custom Cloud Monitoring Metrics
https://cdn.qwiklabs.com/peFD70aHkYcyJdJHw7VfhmNAaG1jgPYJlIbcqMjcB9A%3D

gsutil cp -r gs://spls/gsp087/* gs://qwiklabs-gcp-01-103aa5373a5e
gs://qwiklabs-gcp-01-103aa5373a5e/startup.sh
gs://qwiklabs-gcp-01-103aa5373a5e

## Setting up Jenkins on Kubernetes Engine
gcloud config set compute/zone us-east1-d
git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
cd continuous-deployment-on-kubernetes
gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/projecthosting,cloud-platform"
gcloud container clusters list
gcloud container clusters get-credentials jenkins-cd
kubectl cluster-info
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm install cd stable/jenkins -f jenkins/values.yaml --version 1.2.2 --wait
kubectl get pods
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
kubectl get svc
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

## Deploying a Fault-Tolerant Microsoft Active Directory Environment
export region1=us-central1
export region2=us-east1
export zone_1=${region1}-b
export zone_2=${region2}-c
export vpc_name=webappnet
export project_id=$(gcloud config get-value project)
gcloud config set compute/region ${region1}
gcloud compute networks create ${vpc_name}  \
    --description "VPC network to deploy Active Directory" \
    --subnet-mode custom
gcloud compute networks subnets create private-ad-zone-1 \
    --network ${vpc_name} \
    --range 10.1.0.0/24 \
    --region ${region1}
gcloud compute networks subnets create private-ad-zone-2 \
    --network ${vpc_name} \
    --range 10.2.0.0/24 \
    --region ${region2}
gcloud compute firewall-rules create allow-internal-ports-private-ad \
    --network ${vpc_name} \
    --allow tcp:1-65535,udp:1-65535,icmp \
    --source-ranges  10.1.0.0/24,10.2.0.0/24
gcloud compute firewall-rules create allow-rdp \
    --network ${vpc_name} \
    --allow tcp:3389 \
    --source-ranges 0.0.0.0/0

gcloud compute instances create ad-dc1 --machine-type n1-standard-2 \
    --boot-disk-type pd-ssd \
    --boot-disk-size 50GB \
    --image-family windows-2016 --image-project windows-cloud \
    --network ${vpc_name} \
    --zone ${zone_1} --subnet private-ad-zone-1 \
    --private-network-ip=10.1.0.100
gcloud compute reset-windows-password ad-dc1 --zone ${zone_1} --quiet --user=admin
