# Deploy to Kubernetes in Google Cloud

## Introduction to Docker
docker run hello-world
docker images
mkdir test && cd test
```bash
cat > Dockerfile <<EOF
# Use an official Node runtime as the parent image
FROM node:6

# Set the working directory in the container to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
ADD . /app

# Make the container's port 80 available to the outside world
EXPOSE 80

# Run app.js using node when the container launches
CMD ["node", "app.js"]
EOF
```
```bash
cat > app.js <<EOF
const http = require('http');

const hostname = '0.0.0.0';
const port = 80;

const server = http.createServer((req, res) => {
    res.statusCode = 200;
      res.setHeader('Content-Type', 'text/plain');
        res.end('Hello World\n');
});

server.listen(port, hostname, () => {
    console.log('Server running at http://%s:%s/', hostname, port);
});

process.on('SIGINT', function() {
    console.log('Caught interrupt signal and will exit');
    process.exit();
});
EOF
```
docker build -t node-app:0.1 .
docker run -p 4000:80 --name my-app node-app:0.1
docker build -t node-app:0.2 .
gcloud config list project
docker tag node-app:0.2 gcr.io/qwiklabs-gcp-04-4c631338f8af/node-app:0.2
docker push gcr.io/qwiklabs-gcp-04-4c631338f8af/node-app:0.2

docker stop $(docker ps -q)
docker rm $(docker ps -aq)
docker rmi node-app:0.2 gcr.io/[project-id]/node-app node-app:0.1
docker rmi node:6
docker rmi $(docker images -aq) # remove remaining images
docker images
docker pull gcr.io/[project-id]/node-app:0.2
docker run -p 4000:80 -d gcr.io/[project-id]/node-app:0.2
curl http://localhost:4000

## Kubernetes Engine: Qwik Start
done before

## Orchestrating the Cloud with Kubernetes

App is hosted on GitHub and provides an example 12-Factor application. During this lab you will be working with the following Docker images:

kelseyhightower/monolith - Monolith includes auth and hello services.
kelseyhightower/auth - Auth microservice. Generates JWT tokens for authenticated users.
kelseyhightower/hello - Hello microservice. Greets authenticated users.
nginx - Frontend to the auth and hello services.

gcloud config set compute/zone us-central1-b
gcloud container clusters create io
gsutil cp -r gs://spls/gsp021/* .
cd orchestrate-with-kubernetes/kubernetes
kubectl create deployment nginx --image=nginx:1.10.0
kubectl get pods
kubectl expose deployment nginx --port 80 --type LoadBalancer
kubectl get services
kubectl create -f pods/monolith.yaml
kubectl get pods
kubectl describe pods monolith

### In the 2nd terminal
kubectl port-forward monolith 10080:80

### Now in the 1st terminal
curl http://127.0.0.1:10080
curl http://127.0.0.1:10080/secure
curl -u user http://127.0.0.1:10080/login
TOKEN=$(curl http://127.0.0.1:10080/login -u user|jq -r '.token')
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:10080/secure
kubectl logs monolith
kubectl logs -f monolith
kubectl exec monolith --stdin --tty -c monolith /bin/sh

### Creating a Service
cd ~/orchestrate-with-kubernetes/kubernetes
cat pods/secure-monolith.yaml
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-proxy-conf --from-file nginx/proxy.conf
kubectl create -f pods/secure-monolith.yaml
kubectl create -f services/monolith.yaml

gcloud compute firewall-rules create allow-monolith-nodeport \
  --allow=tcp:31000
gcloud compute instances list
curl -k https://<EXTERNAL_IP>:31000

### Adding Labels to Pods
kubectl get pods -l "app=monolith"
kubectl get pods -l "app=monolith,secure=enabled"
kubectl label pods secure-monolith 'secure=enabled'
kubectl get pods secure-monolith --show-labels
kubectl describe services monolith | grep Endpoints

### Creating Deployments
cat deployments/auth.yaml
kubectl create -f deployments/auth.yaml
kubectl create -f services/auth.yaml
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
kubectl get services frontend

##Continuous Delivery with Jenkins in Kubernetes Engine
gcloud config set compute/zone us-east1-d
git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
cd continuous-deployment-on-kubernetes
gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"

gcloud container clusters list
gcloud container clusters get-credentials jenkins-cd
kubectl cluster-info

### Setup Helm
helm repo add jenkins https://charts.jenkins.io
helm repo update
helm install cd jenkins/jenkins -f jenkins/values.yaml --version 1.2.2 --wait
kubectl get pods
kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
kubectl get svc
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

### Deploying the Application
cd sample-app
kubectl create ns production
kubectl apply -f k8s/production -n production
kubectl apply -f k8s/canary -n production
kubectl apply -f k8s/services -n production
kubectl scale deployment gceme-frontend-production -n production --replicas 4
kubectl get pods -n production -l app=gceme -l role=frontend
kubectl get pods -n production -l app=gceme -l role=backend
kubectl get service gceme-frontend -n production
export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
curl http://$FRONTEND_SERVICE_IP/version

### Creating the Jenkins Pipeline
gcloud source repos create default
git init
git config credential.helper gcloud.sh
git remote add origin https://source.developers.google.com/p/$DEVSHELL_PROJECT_ID/r/default
git config --global user.email "[EMAIL_ADDRESS]"
git config --global user.name "[USERNAME]"
git add .
git commit -m "Initial commit"
git push origin master

https://source.developers.google.com/p/qwiklabs-gcp-00-cbacdf3e9c14/r/default

### Creating the Development Environment
git checkout -b new-feature


kubectl proxy &
curl \
http://localhost:8001/api/v1/namespaces/new-feature/services/gceme-frontend:80/proxy/version

### Deploying a Canary Release
git checkout -b canary
git push origin canary
export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done

## Deploy to Kubernetes in Google Cloud: Challenge Lab
Task 1: Create a Docker image and store the Dockerfile
gcloud config set compute/zone us-east1-b
source <(gsutil cat gs://cloud-training/gsp318/marking/setup_marking.sh)
gcloud source repos clone valkyrie-app --project=qwiklabs-gcp-01-0f5d829cc9f4
cd valkyrie-app
vim Dockerfile
FROM golang:1.10
WORKDIR /go/src/app
COPY source .
RUN go install -v
ENTRYPOINT ["app","-single=true","-port=8080"]

docker build -t valkyrie-app:v0.0.1 .

Task 2: Test the created Docker image
docker run -p 8080:8080 --name kraken-webserver1 valkyrie-app:v0.0.1 &

Task 3: Push the Docker image in the Container Repository
docker tag valkyrie-app:v0.0.1 gcr.io/qwiklabs-gcp-01-0f5d829cc9f4/valkyrie-app:v0.0.1
docker push gcr.io/qwiklabs-gcp-01-0f5d829cc9f4/valkyrie-app:v0.0.1

Task 4: Create and expose a deployment in Kubernetes
gcloud container clusters list
gcloud config set compute/zone us-east1-d
gcloud container clusters get-credentials valkyrie-dev
kubectl cluster-info

Task 5: Update the deployment with a new version of valkyrie-app
kubectl scale deployment valkyrie-dev --replicas 3
git merge origin/kurt-dev
docker build -t valkyrie-app:v0.0.2 .
docker tag valkyrie-app:v0.0.2 gcr.io/qwiklabs-gcp-01-0f5d829cc9f4/valkyrie-app:v0.0.2
docker push gcr.io/qwiklabs-gcp-01-0f5d829cc9f4/valkyrie-app:v0.0.2
kubectl edit deployment valkyrie-dev

Task 6: Create a pipeline in Jenkins to deploy your app
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
9edNCghcZp
docker container kill $(docker ps -q)
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &

6.2 Adding your service account credentials
In the Jenkins user interface, click Credentials in the left navigation.

Click Jenkins

Click Global credentials (unrestricted).

Click Add Credentials in the left navigation.

Select Google Service Account from metadata from the Kind drop-down and click OK.

Click Jenkins > New Item in the left navigation:

Name the project valkyrie-app, then choose the Multibranch Pipeline option and click OK.

On the next page, in the Branch Sources section, click Add Source and select git.

Paste the HTTPS clone URL of your sample-app repo in Cloud Source Repositories https://source.developers.google.com/p/qwiklabs-gcp-01-0f5d829cc9f4/r/valkyrie-app into the Project Repository field. Remember to replace YOUR_PROJECT_ID with your GCP Project ID.

From the Credentials drop-down, select the name of the credentials you created when adding your service account in the previous steps.

Under Scan Multibranch Pipeline Triggers section, check the Periodically if not otherwise run box and set the Interval value to 1 minute.

Your job configuration should look like this:


git config --global user.email $PROJECT
git config --global user.name $PROJECT

git add *
git commit -m 'green to orange'
git push origin master

https://cdn.qwiklabs.com/PQNx2ZWzsymDLNMYVZgXVTOixmLXKgXS4S7cB5aHesY%3D