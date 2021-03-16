# Deploy and Manage Cloud Environments with Google Cloud

## Continuous Delivery Pipelines with Spinnaker and Kubernetes Engine

gcloud config set compute/zone us-central1-f
gcloud container clusters create spinnaker-tutorial \
    --machine-type=n1-standard-2

gcloud iam service-accounts create spinnaker-account \
    --display-name spinnaker-account
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:spinnaker-account" \
    --format='value(email)')
export PROJECT=$(gcloud info --format='value(config.project)')
gcloud projects add-iam-policy-binding $PROJECT \
    --role roles/storage.admin \
    --member serviceAccount:$SA_EMAIL

gcloud iam service-accounts keys create spinnaker-sa.json \
     --iam-account $SA_EMAIL
gcloud pubsub topics create projects/$PROJECT/topics/gcr
gcloud pubsub subscriptions create gcr-triggers \
    --topic projects/${PROJECT}/topics/gcr
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:spinnaker-account" \
    --format='value(email)')
gcloud beta pubsub subscriptions add-iam-policy-binding gcr-triggers \
    --role roles/pubsub.subscriber --member serviceAccount:$SA_EMAIL

### Install Helm

wget https://get.helm.sh/helm-v3.1.1-linux-amd64.tar.gz
tar zxfv helm-v3.1.1-linux-amd64.tar.gz
cp linux-amd64/helm .
kubectl create clusterrolebinding user-admin-binding \
    --clusterrole=cluster-admin --user=$(gcloud config get-value account)
kubectl create clusterrolebinding --clusterrole=cluster-admin \
    --serviceaccount=default:default spinnaker-admin
./helm repo add stable https://charts.helm.sh/stable
./helm repo update
export PROJECT=$(gcloud info \
    --format='value(config.project)')
export BUCKET=$PROJECT-spinnaker-config
gsutil mb -c regional -l us-central1 gs://$BUCKET

export SA_JSON=$(cat spinnaker-sa.json)
export PROJECT=$(gcloud info --format='value(config.project)')
export BUCKET=$PROJECT-spinnaker-config
cat > spinnaker-config.yaml <<EOF
gcs:
  enabled: true
  bucket: $BUCKET
  project: $PROJECT
  jsonKey: '$SA_JSON'

dockerRegistries:
- name: gcr
  address: https://gcr.io
  username: _json_key
  password: '$SA_JSON'
  email: 1234@5678.com

# Disable minio as the default storage backend
minio:
  enabled: false

# Configure Spinnaker to enable GCP services
halyard:
  spinnakerVersion: 1.19.4
  image:
    repository: us-docker.pkg.dev/spinnaker-community/docker/halyard
    tag: 1.32.0
    pullSecrets: []
  additionalScripts:
    create: true
    data:
      enable_gcs_artifacts.sh: |-
        \$HAL_COMMAND config artifact gcs account add gcs-$PROJECT --json-path /opt/gcs/key.json
        \$HAL_COMMAND config artifact gcs enable
      enable_pubsub_triggers.sh: |-
        \$HAL_COMMAND config pubsub google enable
        \$HAL_COMMAND config pubsub google subscription add gcr-triggers \
          --subscription-name gcr-triggers \
          --json-path /opt/gcs/key.json \
          --project $PROJECT \
          --message-format GCR
EOF

./helm install -n default cd stable/spinnaker -f spinnaker-config.yaml \
           --version 2.0.0-rc9 --timeout 10m0s --wait

1. You will need to create 2 port forwarding tunnels in order to access the Spinnaker UI:
  export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" -o jsonpath="{.items[0].metadata.name}")
  kubectl port-forward --namespace default $DECK_POD 9000

  export GATE_POD=$(kubectl get pods --namespace default -l "cluster=spin-gate" -o jsonpath="{.items[0].metadata.name}")
  kubectl port-forward --namespace default $GATE_POD 8084

2. Visit the Spinnaker UI by opening your browser to: http://127.0.0.1:9000

To customize your Spinnaker installation. Create a shell in your Halyard pod:

  kubectl exec --namespace default -it cd-spinnaker-halyard-0 bash

For more info on using Halyard to customize your installation, visit:
  https://www.spinnaker.io/reference/halyard/

For more info on the Kubernetes integration for Spinnaker, visit:
  https://www.spinnaker.io/reference/providers/kubernetes-v2/



export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" \
    -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward --namespace default $DECK_POD 8080:9000 >> /dev/null &


### Building the Docker image

gsutil -m cp -r gs://spls/gsp114/sample-app.tar .
mkdir sample-app
tar xvf sample-app.tar -C ./sample-app
cd sample-app
git config --global user.email "$(gcloud config get-value core/account)"
git config --global user.name student-02-7604fffc54c2@qwiklabs.net

git init
git add .
git commit -m "Initial commit"
gcloud source repos create sample-app
git config credential.helper gcloud.sh
export PROJECT=$(gcloud info --format='value(config.project)')
git remote add origin https://source.developers.google.com/p/$PROJECT/r/sample-app
git push origin master


### Prepare your Kubernetes Manifests for use in Spinnaker

export PROJECT=$(gcloud info --format='value(config.project)')
gsutil mb -l us-central1 gs://$PROJECT-kubernetes-manifests
gsutil versioning set on gs://$PROJECT-kubernetes-manifests
sed -i s/PROJECT/$PROJECT/g k8s/deployments/*
git commit -a -m "Set project ID"
git tag v1.0.0
git push --tags

### Configuring your deployment pipelines
curl -LO https://storage.googleapis.com/spinnaker-artifacts/spin/1.14.0/linux/amd64/spin
chmod +x spin
./spin application save --application-name sample \
                        --owner-email "$(gcloud config get-value core/account)" \
                        --cloud-providers kubernetes \
                        --gate-endpoint http://localhost:8080/gate
export PROJECT=$(gcloud info --format='value(config.project)')
sed s/PROJECT/$PROJECT/g spinnaker/pipeline-deploy.json > pipeline.json
./spin pipeline save --gate-endpoint http://localhost:8080/gate -f pipeline.json

### Triggering your pipeline from code changes

sed -i 's/orange/blue/g' cmd/gke-info/common-service.go
git commit -a -m "Change color to blue"
git tag v1.0.1
git push --tags


## Deploy and Manage Cloud Environments with Google Cloud: Challenge Lab

Task 1: Create the production environment

1.1 Create the kraken-prod-vpc using the given Deployment Manager configuration
- SET_REGION to us-east1.
- gcloud deployment-manager deployments create prod-network  --config prod-network.yaml

1.2 Create a Kubernetes cluster in the new network
name the cluser kraken-prod
set the number of nodes to 2
choose kraken-prod-vpc in the network tab

1.3 Create the frontend and backend deployments and services
cd /work/k8s
gcloud container clusters get-credentials kraken-prod --zone=us-east1-b
kubectl create -f deployment-prod-backend.yaml
kubectl create -f deployment-prod-frontend.yaml
kubectl get pods
kubectl create -f service-prod-backend.yaml
kubectl create -f service-prod-frontend.yaml
kubectl get services

Task 2: Setup the Admin instance

2.1 Create a VM instance
name the instance kraken-admin
choose the zone us-east1-b
setup both kraken-mgmt-subnet and kraken-prod-subnet as the network interfaces in the Networking tab

2.2 Create a Monitoring workspace
Click on Navigation menu > Monitoring.

Click Alerting from the left menu, then click Create Policy.

Click Add Condition, and then set up the Metrics with the following parameters:

Fields	Options
Resource Type	GCE VM Instance
Metric	CPU Utilization compute.googleapis.com/instance/cpu/utilization
Filter	Choose instance id and paste the value copied from kraken-admin
Threshold	0.5 for 1 minute

Task 3: Verify the Spinnaker deployment

3.2 Clone your source code repository
In the Console, click on Navigation menu > Source Repositories.

Click sample-app.

Click Clone at the top of the repository, and copy the git clone command to the Cloud Shell.

cd sample-app
git config --global user.email "$(gcloud config get-value core/account)"
git config --global user.name "$(gcloud config get-value core/account)"

export DECK_POD=$(kubectl get pods --namespace default -l "cluster=spin-deck" \
    -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward --namespace default spin-deck-5d9b54dd7d-lltvt 8080:9000 >> /dev/null &

3.3 Triggering your pipeline from code changes
sed -i 's/orange/blue/g' cmd/gke-info/common-service.go
git commit -a -m "Change color to blue"
git tag v1.0.1
git push --tags

https://cdn.qwiklabs.com/wld3ZcGahbatGIOofQFRB%2FWBZnfMfQ8fGNqfgtS3C6s%3D