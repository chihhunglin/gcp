# Ensure Access and Identity in Google Cloud

## IAM Custom Roles

gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
gcloud iam roles describe [ROLE_NAME]
gcloud iam list-grantable-roles //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID

title: [ROLE_TITLE]
description: [ROLE_DESCRIPTION]
stage: [LAUNCH_STAGE]
includedPermissions:
- [PERMISSION_1]
- [PERMISSION_2]

vim role-definition.yaml
title: "Role Editor"
description: "Edit access for App Versions"
stage: "ALPHA"
includedPermissions:
- appengine.versions.create
- appengine.versions.delete

gcloud iam roles create editor --project $DEVSHELL_PROJECT_ID \
--file role-definition.yaml
gcloud iam roles create viewer --project $DEVSHELL_PROJECT_ID \
--title "Role Viewer" --description "Custom role description." \
--permissions compute.instances.get,compute.instances.list --stage ALPHA
gcloud iam roles list --project $DEVSHELL_PROJECT_ID
gcloud iam roles list
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--add-permissions storage.buckets.get,storage.buckets.list
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--stage DISABLED
gcloud iam roles delete viewer --project $DEVSHELL_PROJECT_ID
gcloud iam roles undelete viewer --project $DEVSHELL_PROJECT_ID


## Service Accounts and Roles: Fundamentals

gcloud iam service-accounts create my-sa-123 --display-name "my service account"
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member serviceAccount:my-sa-123@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/editor


## VPC Network Peering

gcloud config set project <PROJECT_ID2>
A:
gcloud compute networks create network-a --subnet-mode custom
gcloud compute networks subnets create network-a-central --network network-a \
    --range 10.0.0.0/16 --region us-central1
gcloud compute instances create vm-a --zone us-central1-a --network network-a --subnet network-a-central
gcloud compute firewall-rules create network-a-fw --network network-a --allow tcp:22,icmp
B:
gcloud compute networks create network-b --subnet-mode custom
gcloud compute networks subnets create network-b-central --network network-b \
    --range 10.8.0.0/16 --region us-central1
gcloud compute instances create vm-b --zone us-central1-a --network network-b --subnet network-b-central
gcloud compute firewall-rules create network-b-fw --network network-b --allow tcp:22,icmp

### Setting up a VPC Network Peering session https://cdn.qwiklabs.com/i6fwOoVXTt4J7oToas%2BFV61tUgLuDiaw5y7zGEnr6lU%3D

gcloud compute routes list --project <FIRST_PROJECT_ID>


## User Authentication: Identity-Aware Proxy https://cloud.google.com/iap/docs/concepts-overview?hl=zh-tw

git clone https://github.com/googlecodelabs/user-authentication-with-iap.git
cd user-authentication-with-iap
gcloud app deploy
gcloud app browse
gcloud services disable appengineflex.googleapis.com


## Getting Started with Cloud KMS

BUCKET_NAME=<YOUR_NAME>_enron_corpus
gsutil mb gs://${BUCKET_NAME}
gsutil cp gs://enron_emails/allen-p/inbox/1. .
tail 1.
gcloud services enable cloudkms.googleapis.com
KEYRING_NAME=test CRYPTOKEY_NAME=qwiklab
gcloud kms keyrings create $KEYRING_NAME --location global
gcloud kms keys create $CRYPTOKEY_NAME --location global \
      --keyring $KEYRING_NAME \
      --purpose encryption
PLAINTEXT=$(cat 1. | base64 -w0)
curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:encrypt" \
  -d "{\"plaintext\":\"$PLAINTEXT\"}" \
  -H "Authorization:Bearer $(gcloud auth application-default print-access-token)"\
  -H "Content-Type: application/json"
curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:encrypt" \
  -d "{\"plaintext\":\"$PLAINTEXT\"}" \
  -H "Authorization:Bearer $(gcloud auth application-default print-access-token)"\
  -H "Content-Type:application/json" \
| jq .ciphertext -r > 1.encrypted
curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:decrypt" \
  -d "{\"ciphertext\":\"$(cat 1.encrypted)\"}" \
  -H "Authorization:Bearer $(gcloud auth application-default print-access-token)"\
  -H "Content-Type:application/json" \
| jq .plaintext -r | base64 -d
gsutil cp 1.encrypted gs://${BUCKET_NAME}
USER_EMAIL=$(gcloud auth list --limit=1 2>/dev/null | grep '@' | awk '{print $2}')
gcloud kms keyrings add-iam-policy-binding $KEYRING_NAME \
    --location global \
    --member user:$USER_EMAIL \
    --role roles/cloudkms.admin
gcloud kms keyrings add-iam-policy-binding $KEYRING_NAME \
    --location global \
    --member user:$USER_EMAIL \
    --role roles/cloudkms.cryptoKeyEncrypterDecrypter
gsutil -m cp -r gs://enron_emails/allen-p .


## Setting up a Private Kubernetes Cluster

gcloud config set compute/zone us-central1-a
gcloud beta container clusters create private-cluster \
    --enable-private-nodes \
    --master-ipv4-cidr 172.16.0.16/28 \
    --enable-ip-alias \
    --create-subnetwork ""
gcloud compute networks subnets list --network default
gcloud compute networks subnets describe gke-private-cluster-subnet-f906703f --region us-central1
gcloud compute instances create source-instance --zone us-central1-a --scopes 'https://www.googleapis.com/auth/cloud-platform'
gcloud compute instances describe source-instance --zone us-central1-a | grep natIP
gcloud container clusters update private-cluster \
    --enable-master-authorized-networks \
    --master-authorized-networks 34.122.76.183/32
gcloud compute ssh source-instance --zone us-central1-a
gcloud components install kubectl
gcloud container clusters get-credentials private-cluster --zone us-central1-a
kubectl get nodes --output yaml | grep -A4 addresses
kubectl get nodes --output wide
exit
gcloud container clusters delete private-cluster --zone us-central1-a

gcloud compute networks subnets create my-subnet \
    --network default \
    --range 10.0.4.0/22 \
    --enable-private-ip-google-access \
    --region us-central1 \
    --secondary-range my-svc-range=10.0.32.0/20,my-pod-range=10.4.0.0/14
gcloud beta container clusters create private-cluster2 \
    --enable-private-nodes \
    --enable-ip-alias \
    --master-ipv4-cidr 172.16.0.32/28 \
    --subnetwork my-subnet \
    --services-secondary-range-name my-svc-range \
    --cluster-secondary-range-name my-pod-range
gcloud container clusters update private-cluster2 \
    --enable-master-authorized-networks \
    --master-authorized-networks 34.122.76.183/32
gcloud compute ssh source-instance --zone us-central1-a
gcloud container clusters get-credentials private-cluster2 --zone us-central1-a
kubectl get nodes --output yaml | grep -A4 addresses


## Ensure Access & Identity in Google Cloud: Challenge Lab

Task 1: Create a custom security role.
vim role-definition.yaml
title: "Role Editor"
description: "Edit access for App Versions"
stage: "ALPHA"
includedPermissions:
- storage.buckets.get
- storage.objects.get
- storage.objects.list
- storage.objects.update
- storage.objects.create

gcloud iam roles create orca_storage_update --project $DEVSHELL_PROJECT_ID \
--file role-definition.yaml

Task 2: Create a service account.
gcloud iam service-accounts create orca-private-cluster-sa --display-name "my service account"

Task 3: Bind a custom security role to a service account.
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member serviceAccount:orca-private-cluster-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role projects/qwiklabs-gcp-04-2fa553d5157d/roles/orca_storage_update

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member serviceAccount:orca-private-cluster-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/monitoring.viewer

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member serviceAccount:orca-private-cluster-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/monitoring.metricWriter

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member serviceAccount:orca-private-cluster-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/logging.logWriter


Task 4: Create and configure a new Kubernetes Engine private cluster

gcloud config set compute/zone us-east1-b

gcloud container clusters create orca-test-cluster \
	--num-nodes 1 \
	--master-ipv4-cidr=172.16.0.64/28 \
	--network orca-build-vpc \
	--subnetwork orca-build-subnet \
	--enable-master-authorized-networks  \
	--master-authorized-networks 192.168.10.2/32 \
	--enable-ip-alias \
	--enable-private-nodes \
	--enable-private-endpoint \
	--service-account orca-private-cluster-sa@qwiklabs-gcp-04-2fa553d5157d.iam.gserviceaccount.com 


Task 5: Deploy an application to a private Kubernetes Engine cluster.

gcloud container clusters get-credentials orca-test-cluster --zone us-east1-b --project qwiklabs-gcp-04-2fa553d5157d
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0

https://cdn.qwiklabs.com/kQVL4GDnqAMr7rQqvpp2q3yVol%2FtCmHKIN8nuzsFiRg%3D