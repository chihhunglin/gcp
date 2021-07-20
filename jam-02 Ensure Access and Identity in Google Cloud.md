# 02 Ensure Access and Identity in Google Cloud

https://github.com/chihhunglin/gcp/blob/master/qwik-gcp-04-Ensure%20Access%20and%20Identity%20in%20Google%20Cloud.md

## IAM Custom Roles
gcloud iam list-testable-permissions //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID
gcloud iam roles describe [ROLE_NAME]
gcloud iam list-grantable-roles //cloudresourcemanager.googleapis.com/projects/$DEVSHELL_PROJECT_ID

### YAML
title: [ROLE_TITLE]
description: [ROLE_DESCRIPTION]
stage: [LAUNCH_STAGE]
includedPermissions:
- [PERMISSION_1]
- [PERMISSION_2]

```shell
nano role-definition.yaml
title: "Role Editor"
description: "Edit access for App Versions"
stage: "ALPHA"
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
```

gcloud iam roles create editor --project $DEVSHELL_PROJECT_ID \
--file role-definition.yaml
gcloud iam roles create viewer --project $DEVSHELL_PROJECT_ID \
--title "Role Viewer" --description "Custom role description." \
--permissions compute.instances.get,compute.instances.list --stage ALPHA
gcloud iam roles list --project $DEVSHELL_PROJECT_ID
gcloud iam roles list

gcloud iam roles describe editor --project $DEVSHELL_PROJECT_ID
```shell
description: Edit access for App Versions
etag: BwXEsVtZi1k=
includedPermissions:
- appengine.versions.create
- appengine.versions.delete
name: projects/qwiklabs-gcp-01-b324e7528c92/roles/editor
stage: ALPHA
title: Role Editor
```
gcloud iam roles update editor --project $DEVSHELL_PROJECT_ID \
--file new-role-definition.yaml
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--add-permissions storage.buckets.get,storage.buckets.list
gcloud iam roles update viewer --project $DEVSHELL_PROJECT_ID \
--stage DISABLED
gcloud iam roles delete viewer --project $DEVSHELL_PROJECT_ID
gcloud iam roles undelete viewer --project $DEVSHELL_PROJECT_ID

## Service Accounts and Roles: Fundamentals
What are Service Accounts?
A service account is a special Google account that belongs to your application or a virtual machine (VM) instead of an individual end user. Your application uses the service account to call the Google API of a service, so that the users aren't directly involved.

### User-managed service accounts
PROJECT_NUMBER-compute@developer.gserviceaccount.com
PROJECT_ID@appspot.gserviceaccount.com

### Google-managed service accounts

### Google APIs service account
PROJECT_NUMBER@cloudservices.gserviceaccount.com

gcloud iam service-accounts create my-sa-123 --display-name "my service account"
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
    --member serviceAccount:my-sa-123@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/editor

### Types of Roles
There are three types of roles in Cloud IAM:

Primitive roles, which include the Owner, Editor, and Viewer roles that existed prior to the introduction of Cloud IAM.
Predefined roles, which provide granular access for a specific service and are managed by Google Cloud.
Custom roles, which provide granular access according to a user-specified list of permissions.
For more details, please visit IAM Roles.

### IN VM
sudo apt-get update
sudo apt-get install virtualenv
virtualenv -p python3 venv
source venv/bin/activate
sudo apt-get install -y git python3-pip
pip install google-cloud-bigquery
pip install pandas

```bash
echo "
from google.auth import compute_engine
from google.cloud import bigquery

credentials = compute_engine.Credentials(
    service_account_email='YOUR_SERVICE_ACCOUNT')

query = '''
SELECT
  year,
  COUNT(1) as num_babies
FROM
  publicdata.samples.natality
WHERE
  year > 2000
GROUP BY
  year
'''

client = bigquery.Client(
    project='YOUR_PROJECT_ID',
    credentials=credentials)
print(client.query(query).to_dataframe())
" > query.py

```

sed -i -e "s/YOUR_PROJECT_ID/$(gcloud config get-value project)/g" query.py
cat query.py
sed -i -e "s/YOUR_SERVICE_ACCOUNT/bigquery-qwiklab@$(gcloud config get-value project).iam.gserviceaccount.com/g" query.py
cat query.py
python query.py

## VPC Network Peering
gcloud config set project <PROJECT_ID2>

Project-A:
gcloud compute networks create network-a --subnet-mode custom
gcloud compute networks subnets create network-a-central --network network-a \
    --range 10.0.0.0/16 --region us-central1
gcloud compute instances create vm-a --zone us-central1-a --network network-a --subnet network-a-central
gcloud compute firewall-rules create network-a-fw --network network-a --allow tcp:22,icmp

Project-B:
gcloud compute networks create network-b --subnet-mode custom
gcloud compute networks subnets create network-b-central --network network-b \
    --range 10.8.0.0/16 --region us-central1
gcloud compute instances create vm-b --zone us-central1-a --network network-b --subnet network-b-central
gcloud compute firewall-rules create network-b-fw --network network-b --allow tcp:22,icmp

gcloud compute routes list --project <FIRST_PROJECT_ID>

## User Authentication: Identity-Aware Proxy
What is Identity-Aware Proxy?
Identity-Aware Proxy (IAP) is a Google Cloud service that intercepts web requests sent to your application, authenticates the user making the request using the Google Identity Service, and only lets the requests through if they come from a user you authorize. In addition, it can modify the request headers to include information about the authenticated user.

git clone https://github.com/googlecodelabs/user-authentication-with-iap.git
cd user-authentication-with-iap
cd 1-HelloWorld
cat main.py
gcloud app deploy
gcloud app browse

gcloud services disable appengineflex.googleapis.com

cd ~/user-authentication-with-iap/2-HelloUser
gcloud app deploy
curl -X GET <your-url-here> -H "X-Goog-Authenticated-User-Email: totally fake email"

cd ~/user-authentication-with-iap/3-HelloVerifiedUser
gcloud app deploy

## Getting Started with Cloud KMS
BUCKET_NAME=qwiklabs-gcp-02-e2ad5e64319b_enron_corpus
gsutil mb gs://${BUCKET_NAME}

### The Enron Corpus is a database of over 600,000 emails generated by 158 employees
https://en.wikipedia.org/wiki/Enron_Corpus

gsutil cp gs://enron_emails/allen-p/inbox/1. .
tail 1.

gcloud services enable cloudkms.googleapis.com

### Create a Keyring and Cryptokey
KEYRING_NAME=test CRYPTOKEY_NAME=qwiklab
gcloud kms keyrings create $KEYRING_NAME --location global
gcloud kms keys create $CRYPTOKEY_NAME --location global \
      --keyring $KEYRING_NAME \
      --purpose encryption

### Encrypt Your Data
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

### Configure IAM Permissions
In KMS, there are two major permissions to focus on. One permissions allows a user or service account to manage KMS resources, the other allows a user or service account to use keys to encrypt and decrypt data.

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
MYDIR=allen-p
FILES=$(find $MYDIR -type f -not -name "*.encrypted")
for file in $FILES; do
  PLAINTEXT=$(cat $file | base64 -w0)
  curl -v "https://cloudkms.googleapis.com/v1/projects/$DEVSHELL_PROJECT_ID/locations/global/keyRings/$KEYRING_NAME/cryptoKeys/$CRYPTOKEY_NAME:encrypt" \
    -d "{\"plaintext\":\"$PLAINTEXT\"}" \
    -H "Authorization:Bearer $(gcloud auth application-default print-access-token)" \
    -H "Content-Type:application/json" \
  | jq .ciphertext -r > $file.encrypted
done
gsutil -m cp allen-p/inbox/*.encrypted gs://${BUCKET_NAME}/allen-p/inbox

## Setting up a Private Kubernetes Cluster
gcloud config set compute/zone us-central1-a
gcloud beta container clusters create private-cluster \
    --enable-private-nodes \
    --master-ipv4-cidr 172.16.0.16/28 \
    --enable-ip-alias \
    --create-subnetwork ""
gcloud compute networks subnets list --network default
```bash
gke-private-cluster-subnet-b81838bf
```
gcloud compute networks subnets describe gke-private-cluster-subnet-b81838bf --region us-central1
gcloud compute instances create source-instance --zone us-central1-a --scopes 'https://www.googleapis.com/auth/cloud-platform'
gcloud compute instances describe source-instance --zone us-central1-a | grep natIP
gcloud container clusters update private-cluster \
    --enable-master-authorized-networks \
    --master-authorized-networks 35.193.106.225/32
gcloud compute ssh source-instance --zone us-central1-a
sudo apt-get install kubectl
gcloud container clusters get-credentials private-cluster --zone us-central1-a
kubectl get nodes --output yaml | grep -A4 addresses
kubectl get nodes --output wide
exit
gcloud container clusters delete private-cluster --zone us-central1-a

### Creating a private cluster that uses a custom subnetwork
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
    --master-authorized-networks 35.193.106.225/32

gcloud compute ssh source-instance --zone us-central1-a
gcloud container clusters get-credentials private-cluster2 --zone us-central1-a
kubectl get nodes --output yaml | grep -A4 addresses

## Ensure Access & Identity in Google Cloud: Challenge Lab
Task 1: Create a custom security role.
Your first task is to create a new custom IAM security role called orca_storage_update that will provide the Google Cloud storage bucket and object permissions required to be able to create and update storage objects.

```shell
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
```

gcloud iam roles create orca_storage_update --project $DEVSHELL_PROJECT_ID --file role-definition.yaml

Task 2: Create a service account.
Your second task is to create the dedicated service account that will be used as the service account for your new private cluster. You must name this account orca-private-cluster-sa.

gcloud iam service-accounts create orca-private-cluster-sa --display-name "my service account"

Task 3: Bind a custom security role to a service account.
You must now bind the Cloud Operations logging and monitoring roles that are required for Kubernetes Engine Cluster service accounts as well as the custom IAM role you created for storage permissions to the Service Account you created earlier.

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member serviceAccount:orca-private-cluster-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role projects/qwiklabs-gcp-04-8966320b6bbf/roles/orca_storage_update
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member serviceAccount:orca-private-cluster-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/monitoring.viewer
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member serviceAccount:orca-private-cluster-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/monitoring.metricWriter
gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member serviceAccount:orca-private-cluster-sa@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --role roles/logging.logWriter

Task 4: Create and configure a new Kubernetes Engine private cluster
gcloud config set compute/zone us-east1-b
gcloud container clusters create orca-test-cluster \
--num-nodes 1 \
--master-ipv4-cidr=172.16.0.64/28 \
--network orca-build-vpc \
--subnetwork orca-build-subnet \
--enable-master-authorized-networks \
--master-authorized-networks 192.168.10.2/32 \
--enable-ip-alias \
--enable-private-nodes \
--enable-private-endpoint \
--service-account orca-private-cluster-sa@qwiklabs-gcp-04-8966320b6bbf.iam.gserviceaccount.com

Task 5: Deploy an application to a private Kubernetes Engine cluster.
### In orca-jumphost.
gcloud container clusters get-credentials orca-test-cluster --zone us-east1-b --project qwiklabs-gcp-04-8966320b6bbf 
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0