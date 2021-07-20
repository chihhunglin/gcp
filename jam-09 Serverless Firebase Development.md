# Serverless Firebase Development

## Importing Data to a Firestore Database
git clone https://github.com/rosera/pet-theory
cd pet-theory/lab01
npm install @google-cloud/firestore
npm install @google-cloud/logging

npm install faker

gcloud config set project qwiklabs-gcp-02-c94ffabffc1e
PROJECT_ID=$(gcloud config get-value project)
node createTestData 1000
node importTestData customers_1000.csv

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=user:student-03-a7ee661fa0f1@qwiklabs.net --role=roles/logging.viewer

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=user:student-03-a7ee661fa0f1@qwiklabs.net --role roles/source.writer

## Build a Serverless Web App with Firebase

### Install the Firebase CLI and deploy to Firebase Hosting
git clone https://github.com/rosera/pet-theory.git
cd pet-theory/lab02
npm init --yes
npm install -g firebase-tools
firebase login --no-localhost
gcloud config set compute/region us-central1
firebase init

### Deploy your application
cd ~/pet-theory/lab02/
firebase deploy

## Deploy a Hugo Website with Cloud Build and Firebase Pipeline
### In VM
cd ~
/tmp/installhugo.sh

cd ~
gcloud source repos create my_hugo_site
gcloud source repos clone my_hugo_site

cd ~
/tmp/hugo new site my_hugo_site --force

cd ~/my_hugo_site
git submodule add \
  https://github.com/budparr/gohugo-theme-ananke.git \
  themes/ananke
echo 'theme = "ananke"' >> config.toml

cd ~/my_hugo_site
/tmp/hugo server -D --bind 0.0.0.0 --port 8080

curl -sL https://firebase.tools | bash
cd ~/my_hugo_site
firebase init
/tmp/hugo && firebase deploy
cd ~/my_hugo_site
echo "resources" >> .gitignore
git add .
git commit -m "Add app to Cloud Source Repositories"
git push -u origin master

### Configure the build
cd ~/my_hugo_site
cp /tmp/cloudbuild.yaml .
cat cloudbuild.yaml

cd ~/my_hugo_site
git add .
git commit -m "I updated the site title"
git push -u origin master

## Google Assistant: Build an Application with Dialogflow and Cloud Functions

## Serverless Firebase Development: Challenge Lab
gcloud config set project $(gcloud projects list --format='value(PROJECT_ID)' --filter='qwiklabs-gcp')
git clone https://github.com/rosera/pet-theory.git

### 1. Firestore Database Create
Go to Firestore > Select Naive Mode > Location: nam5 > Create Database

### 2. Firestore Database Populate
cd pet-theory/lab06/firebase-import-csv/solution
npm install
node index.js netflix_titles_original.csv

### 3. Cloud Build Rest API Staging
cd ~/pet-theory/lab06/firebase-rest-api/solution-01
npm install
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1
gcloud beta run deploy netflix-dataset-service --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1 --allow-unauthenticated
#### Choose us-central1

### 4. Cloud Build Rest API Production
cd ~/pet-theory/lab06/firebase-rest-api/solution-02
npm install
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.2
gcloud beta run deploy netflix-dataset-service --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.2 --allow-unauthenticated
# go to cloud run and click netflix-dataset-service then copy the url
SERVICE_URL=<copy url from your netflix-dataset-service>
curl -X GET $SERVICE_URL/2019

### 5. Cloud Build Frontend Staging
cd ~/pet-theory/lab06/firebase-frontend/public
nano app.js # comment line 3 and uncomment line 4, insert your netflix-dataset-service url
npm install
cd ~/pet-theory/lab06/firebase-frontend
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1
gcloud beta run deploy frontend-staging-service --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-staging:0.1
#### Choose 1 and us-central1
https://netflix-dataset-service-ssjep56b5a-uc.a.run.app

### 6. Cloud Build Frontend Production
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-production:0.1
gcloud beta run deploy frontend-production-service --image gcr.io/$GOOGLE_CLOUD_PROJECT/frontend-production:0.1
#### Choose 1 and us-central1