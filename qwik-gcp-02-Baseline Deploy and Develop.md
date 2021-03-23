# Baseline: Deploy and Develop

## Google Cloud SDK: Qwik Start - Redhat/Centos
# Update YUM with Cloud SDK repo information:
sudo tee -a /etc/yum.repos.d/google-cloud-sdk.repo << EOM
[google-cloud-sdk]
name=Google Cloud SDK
baseurl=https://packages.cloud.google.com/yum/repos/cloud-sdk-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOM

# The indentation for the 2nd line of gpgkey is important.

# Install the Cloud SDK
sudo yum install google-cloud-sdk

gcloud init --console-only

gcloud auth list
gcloud config list
gcloud info
gcloud help compute instances create


## App Engine: Qwik Start - Python

gsutil -m cp -r gs://spls/gsp067/python-docs-samples .
cd python-docs-samples/appengine/standard_python3/hello_world

###  Google Cloud development server
dev_appserver.py app.yaml
cd python-docs-samples/appengine/standard_python3/hello_world
gcloud app deploy
gcloud app browse


## Cloud Source Repositories: Qwik Start
gcloud source repos create REPO_DEMO
gcloud source repos clone REPO_DEMO
cd REPO_DEMO
echo 'Hello World!' > myfile.txt
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
git add myfile.txt
git commit -m "First file using Cloud Source Repositories" myfile.txt
git push origin master

## Container-Optimized OS: Qwik Start
sudo docker ps

gcloud compute images list \
    --project cos-cloud \
    --no-standard-images

cos-77-12371-1109-0

 gcloud beta compute instances create-with-container containerized-vm2 \
     --image cos-77-12371-1109-0 \
     --image-project cos-cloud \
     --container-image nginx \
     --container-restart-policy always \
     --zone us-central1-a \
     --machine-type n1-standard-1

gcloud compute firewall-rules create allow-containerized-internal\
  --allow tcp:80 \
  --source-ranges 0.0.0.0/0 \
  --network default

## Datastore: Qwik Start

## Cloud SQL for PostgreSQL: Qwik Start
gcloud sql connect myinstance --user=postgres

## Data Loss Prevention: Qwik Start - Command Line
git clone https://github.com/googleapis/nodejs-dlp.git
export GCLOUD_PROJECT=qwiklabs-gcp-03-3faa47543f94
npm install --save @google-cloud/dlp
npm install yargs
cd nodejs-dlp/samples
node inspectString.js $DEVSHELL_PROJECT_ID "My email address is joe@example.com." LIKELY 0 EMAIL_ADDRESS DICT_TYPE true

## Video Intelligence: Qwik Start
gcloud iam service-accounts create quickstart
gcloud iam service-accounts keys create key.json --iam-account quickstart@qwiklabs-gcp-00-670581b1b096.iam.gserviceaccount.com
gcloud auth activate-service-account --key-file key.json
gcloud auth print-access-token

request.json
{
   "inputUri":"gs://spls/gsp154/video/train.mp4",
   "features": [
       "LABEL_DETECTION"
   ]
}

curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/videos:annotate' \
    -d @request.json

curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/projects/854340189025/locations/asia-east1/operations/991034211068405977'

## Cloud Security Scanner: Qwik Start
gsutil -m cp -r gs://spls/gsp067/python-docs-samples .
cd python-docs-samples/appengine/standard_python3/hello_world
dev_appserver.py app.yaml
gcloud app deploy
gcloud app browse

## Cloud Endpoints: Qwik Start
git clone https://github.com/GoogleCloudPlatform/endpoints-quickstart
cd endpoints-quickstart
cd scripts
./deploy_api.sh
`gcloud endpoints services deploy openapi.yaml`
./deploy_app.sh
./query_api.sh
./generate_traffic.sh
./deploy_api.sh ../openapi_with_ratelimit.yaml
./deploy_app.sh
export API_KEY=YOUR-API-KEY
./query_api_with_key.sh $API_KEY

```
curl -H 'x-api-key: AIzeSyDbdQdaSdhPMdiAuddd_FALbY7JevoMzAB' "https://example-project.appspot.com/airportName?iataCode=SFO"
San Francisco International Airport

```

./generate_traffic_with_key.sh $API_KEY
./query_api_with_key.sh $API_KEY

https://cdn.qwiklabs.com/yJo%2FmEJ2qY6rl0YpP0Mi9Pp3jWGy89ubsylAKx6xsVA%3D