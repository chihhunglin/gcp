# Build a Website on Google Cloud

## Deploy Your Website on Cloud Run
git clone https://github.com/googlecodelabs/monolith-to-microservices.git
cd ~/monolith-to-microservices
./setup.sh
cd ~/monolith-to-microservices/monolith
npm start

gcloud services enable cloudbuild.googleapis.com
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 .

gcloud services enable run.googleapis.com
gcloud run deploy --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 --platform managed
gcloud run services list

gcloud run deploy --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 --platform managed --concurrency 1
gcloud run deploy --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 --platform managed --concurrency 80

cd ~/monolith-to-microservices/react-app/src/pages/Home
mv index.js.new index.js
cat ~/monolith-to-microservices/react-app/src/pages/Home/index.js
cd ~/monolith-to-microservices/react-app
npm run build:monolith
cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0 .

gcloud run deploy --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0 --platform managed
gcloud run services describe monolith --platform managed
gcloud beta run services list

### Delete the container image for version 1.0.0 of our monolith
gcloud container images delete gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 --quiet

### Delete the container image for version 2.0.0 of our monolith
gcloud container images delete gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0 --quiet

### The following command will take all source archives from all builds and delete them from cloud storage

### Run this command to print all sources:
### gcloud builds list | awk 'NR > 1 {print $4}'

gcloud builds list | awk 'NR > 1 {print $4}' | while read line; do gsutil rm $line; done

gcloud beta run services delete monolith --platform managed

## Hosting a Web App on Google Cloud Using Compute Engine
gcloud config set compute/zone us-central1-f
gcloud services enable compute.googleapis.com
gsutil mb gs://fancy-store-$DEVSHELL_PROJECT_ID

git clone https://github.com/googlecodelabs/monolith-to-microservices.git
cd ~/monolith-to-microservices
./setup.sh
cd microservices
npm start

cd ~/monolith-to-microservices
vim startup-script.sh
echo $DEVSHELL_PROJECT_ID
gsutil cp ~/monolith-to-microservices/startup-script.sh gs://fancy-store-$DEVSHELL_PROJECT_ID
cd ~
rm -rf monolith-to-microservices/*/node_modules
gsutil -m cp -r monolith-to-microservices gs://fancy-store-$DEVSHELL_PROJECT_ID/

### Deploy backend instance
gcloud compute instances create backend \
    --machine-type=n1-standard-1 \
    --tags=backend \
   --metadata=startup-script-url=https://storage.googleapis.com/fancy-store-$DEVSHELL_PROJECT_ID/startup-script.sh
gcloud compute instances list
34.135.130.232
monolith-to-microservices > react-app
vim .env
cd ~/monolith-to-microservices/react-app
npm install && npm run-script build
cd ~
rm -rf monolith-to-microservices/*/node_modules
gsutil -m cp -r monolith-to-microservices gs://fancy-store-$DEVSHELL_PROJECT_ID/

### Deploy frontend instance
gcloud compute instances create frontend \
    --machine-type=n1-standard-1 \
    --tags=frontend \
    --metadata=startup-script-url=https://storage.googleapis.com/fancy-store-$DEVSHELL_PROJECT_ID/startup-script.sh

### Configure Network
gcloud compute firewall-rules create fw-fe \
    --allow tcp:8080 \
    --target-tags=frontend
gcloud compute firewall-rules create fw-be \
    --allow tcp:8081-8082 \
    --target-tags=backend
gcloud compute instances list
watch -n 2 curl http://[FRONTEND_ADDRESS]:8080
watch -n 2 curl http://34.133.12.149:8080

### Create Managed Instance Groups
gcloud compute instances stop frontend
gcloud compute instances stop backend
gcloud compute instance-templates create fancy-fe \
    --source-instance=frontend
gcloud compute instance-templates create fancy-be \
    --source-instance=backend
gcloud compute instance-templates list
gcloud compute instances delete backend
gcloud compute instance-groups managed create fancy-fe-mig \
    --base-instance-name fancy-fe \
    --size 2 \
    --template fancy-fe
gcloud compute instance-groups managed create fancy-be-mig \
    --base-instance-name fancy-be \
    --size 2 \
    --template fancy-be
gcloud compute instance-groups set-named-ports fancy-fe-mig \
    --named-ports frontend:8080
gcloud compute instance-groups set-named-ports fancy-be-mig \
    --named-ports orders:8081,products:8082

### Configure autohealing
gcloud compute health-checks create http fancy-fe-hc \
    --port 8080 \
    --check-interval 30s \
    --healthy-threshold 1 \
    --timeout 10s \
    --unhealthy-threshold 3
gcloud compute health-checks create http fancy-be-hc \
    --port 8081 \
    --request-path=/api/orders \
    --check-interval 30s \
    --healthy-threshold 1 \
    --timeout 10s \
    --unhealthy-threshold 3
gcloud compute firewall-rules create allow-health-check \
    --allow tcp:8080-8081 \
    --source-ranges 130.211.0.0/22,35.191.0.0/16 \
    --network default
gcloud compute instance-groups managed update fancy-fe-mig \
    --health-check fancy-fe-hc \
    --initial-delay 300
gcloud compute instance-groups managed update fancy-be-mig \
    --health-check fancy-be-hc \
    --initial-delay 300

### Create Load Balancers
gcloud compute http-health-checks create fancy-fe-frontend-hc \
  --request-path / \
  --port 8080
gcloud compute http-health-checks create fancy-be-orders-hc \
  --request-path /api/orders \
  --port 8081
gcloud compute http-health-checks create fancy-be-products-hc \
  --request-path /api/products \
  --port 8082
gcloud compute backend-services create fancy-fe-frontend \
  --http-health-checks fancy-fe-frontend-hc \
  --port-name frontend \
  --global
gcloud compute backend-services create fancy-be-orders \
  --http-health-checks fancy-be-orders-hc \
  --port-name orders \
  --global
gcloud compute backend-services create fancy-be-products \
  --http-health-checks fancy-be-products-hc \
  --port-name products \
  --global
gcloud compute backend-services add-backend fancy-fe-frontend \
  --instance-group fancy-fe-mig \
  --instance-group-zone us-central1-f \
  --global
gcloud compute backend-services add-backend fancy-be-orders \
  --instance-group fancy-be-mig \
  --instance-group-zone us-central1-f \
  --global
gcloud compute backend-services add-backend fancy-be-products \
  --instance-group fancy-be-mig \
  --instance-group-zone us-central1-f \
  --global
gcloud compute url-maps create fancy-map \
  --default-service fancy-fe-frontend
gcloud compute url-maps add-path-matcher fancy-map \
   --default-service fancy-fe-frontend \
   --path-matcher-name orders \
   --path-rules "/api/orders=fancy-be-orders,/api/products=fancy-be-products"
gcloud compute target-http-proxies create fancy-proxy \
  --url-map fancy-map
gcloud compute forwarding-rules create fancy-http-rule \
  --global \
  --target-http-proxy fancy-proxy \
  --ports 80

### Update Configuration
cd ~/monolith-to-microservices/react-app/
gcloud compute forwarding-rules list --global
REACT_APP_ORDERS_URL=http://[LB_IP]/api/orders
REACT_APP_PRODUCTS_URL=http://[LB_IP]/api/products

cd ~/monolith-to-microservices/react-app
npm install && npm run-script build
cd ~
rm -rf monolith-to-microservices/*/node_modules
gsutil -m cp -r monolith-to-microservices gs://fancy-store-$DEVSHELL_PROJECT_ID/

### Update the frontend instances
gcloud compute instance-groups managed rolling-action replace fancy-fe-mig \
    --max-unavailable 100%
watch -n 2 gcloud compute instance-groups list-instances fancy-fe-mig
watch -n 2 gcloud compute backend-services get-health fancy-fe-frontend --global

### Scaling Compute Engine
gcloud compute instance-groups managed set-autoscaling \
  fancy-fe-mig \
  --max-num-replicas 2 \
  --target-load-balancing-utilization 0.60
gcloud compute instance-groups managed set-autoscaling \
  fancy-be-mig \
  --max-num-replicas 2 \
  --target-load-balancing-utilization 0.60
gcloud compute backend-services update fancy-fe-frontend \
    --enable-cdn --global

### Update the website
gcloud compute instances set-machine-type frontend --machine-type custom-4-3840
gcloud compute instance-templates create fancy-fe-new \
    --source-instance=frontend \
    --source-instance-zone=us-central1-f
gcloud compute instance-groups managed rolling-action start-update fancy-fe-mig \
    --version template=fancy-fe-new
watch -n 2 gcloud compute instance-groups managed list-instances fancy-fe-mig

gcloud compute instances describe [VM_NAME] | grep machineType

cd ~/monolith-to-microservices/react-app/src/pages/Home
mv index.js.new index.js
cat ~/monolith-to-microservices/react-app/src/pages/Home/index.js
cd ~/monolith-to-microservices/react-app
npm install && npm run-script build
cd ~
rm -rf monolith-to-microservices/*/node_modules
gsutil -m cp -r monolith-to-microservices gs://fancy-store-$DEVSHELL_PROJECT_ID/
gcloud compute instance-groups managed rolling-action replace fancy-fe-mig \
    --max-unavailable=100%
watch -n 2 gcloud compute instance-groups list-instances fancy-fe-mig
watch -n 2 gcloud compute backend-services get-health fancy-fe-frontend --global
gcloud compute forwarding-rules list --global

### Simulate Failure
gcloud compute instance-groups list-instances fancy-fe-mig
gcloud compute ssh [INSTANCE_NAME]
watch -n 2 gcloud compute operations list \
--filter='operationType~compute.instances.repair.*'

## Deploy, Scale, and Update Your Website on Google Kubernetes Engine
gcloud config set compute/zone us-central1-f
gcloud services enable container.googleapis.com
gcloud container clusters create fancy-cluster --num-nodes 3
gcloud compute instances list

cd ~
git clone https://github.com/googlecodelabs/monolith-to-microservices.git
cd ~/monolith-to-microservices
./setup.sh
cd ~/monolith-to-microservices/monolith
npm start

### Create Docker container with Cloud Build
gcloud services enable cloudbuild.googleapis.com
cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 .

### Deploy container to GKE
kubectl create deployment monolith --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0
kubectl get all
kubectl expose deployment monolith --type=LoadBalancer --port 80 --target-port 8080
kubectl get service
kubectl scale deployment monolith --replicas=3
kubectl get all

### Make changes to the website
cd ~/monolith-to-microservices/react-app/src/pages/Home
mv index.js.new index.js
cat ~/monolith-to-microservices/react-app/src/pages/Home/index.js
cd ~/monolith-to-microservices/react-app
npm run build:monolith
cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0 .

### Update website with zero downtime
kubectl set image deployment/monolith monolith=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0
kubectl get pods

### Cleanup
cd ~
rm -rf monolith-to-microservices
gcloud builds list | awk 'NR > 1 {print $4}' | while read line; do gsutil rm $line; done
kubectl delete service monolith
kubectl delete deployment monolith
gcloud container clusters delete fancy-cluster

## Migrating a Monolithic Website to Microservices on Google Kubernetes Engine
gcloud config set compute/zone us-central1-f
cd ~
git clone https://github.com/googlecodelabs/monolith-to-microservices.git
cd ~/monolith-to-microservices
./setup.sh

### Create a GKE Cluster
gcloud services enable container.googleapis.com
gcloud container clusters create fancy-cluster --num-nodes 3
gcloud compute instances list

### Deploy Existing Monolith
cd ~/monolith-to-microservices
./deploy-monolith.sh
kubectl get service monolith

### Migrate Orders to a microservice
cd ~/monolith-to-microservices/microservices/src/orders
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/orders:1.0.0 .
kubectl create deployment orders --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/orders:1.0.0
kubectl get all
kubectl expose deployment orders --type=LoadBalancer --port 80 --target-port 8081
kubectl get service orders

### Reconfigure Monolith
34.72.113.196
cd ~/monolith-to-microservices/react-app
vim .env.monolith
http://34.72.113.196/api/orders
npm run build:monolith
cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0 .
kubectl set image deployment/monolith monolith=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0

### Migrate Products to Microservice
cd ~/monolith-to-microservices/microservices/src/products
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/products:1.0.0 .
kubectl create deployment products --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/products:1.0.0
kubectl expose deployment products --type=LoadBalancer --port 80 --target-port 8082
kubectl get service products
34.122.219.238
cd ~/monolith-to-microservices/react-app
vim .env.monolith
http://34.122.219.238/api/products
npm run build:monolith
cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:3.0.0 .
kubectl set image deployment/monolith monolith=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:3.0.0

### Migrate frontend to microservice
cd ~/monolith-to-microservices/react-app
cp .env.monolith .env
npm run build
cd ~/monolith-to-microservices/microservices/src/frontend
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/frontend:1.0.0 .
kubectl create deployment frontend --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/frontend:1.0.0
kubectl expose deployment frontend --type=LoadBalancer --port 80 --target-port 8080

### Delete The Monolith
kubectl delete deployment monolith
kubectl delete service monolith
kubectl get services

## Build a Website on Google Cloud: Challenge Lab
Task 1: Download the monolith code and build your container
git clone https://github.com/googlecodelabs/monolith-to-microservices.git
cd ~/monolith-to-microservices
./setup.sh
cd ~/monolith-to-microservices/monolith
npm start
gcloud services enable cloudbuild.googleapis.com
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/fancytest:1.0.0 .

Task 2: Create a kubernetes cluster and deploy the application
gcloud config set compute/zone us-central1-a
gcloud services enable container.googleapis.com
gcloud container clusters create fancy-cluster --num-nodes 3
kubectl create deployment fancytest --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/fancytest:1.0.0
kubectl expose deployment fancytest --type=LoadBalancer --port 80 --target-port 8080

Task 3: Create a containerized version of your Microservices
cd ~/monolith-to-microservices/microservices/src/orders
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/orders:1.0.0 .
cd ~/monolith-to-microservices/microservices/src/products
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/products:1.0.0 .

Task 4: Deploy the new microservices
kubectl create deployment orders --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/orders:1.0.0
kubectl expose deployment orders --type=LoadBalancer --port 80 --target-port 8081
kubectl create deployment products --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/products:1.0.0
kubectl expose deployment products --type=LoadBalancer --port 80 --target-port 8082

Task 5: Configure the Frontend microservice
cd ~/monolith-to-microservices/react-app
vim .env
35.239.95.80
104.197.80.1

Task 6: Create a containerized version of the Frontend microservice
cd ~/monolith-to-microservices/microservices/src/frontend
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/frontend:1.0.0 .

Task 7: Deploy the Frontend microservice
kubectl create deployment frontend --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/frontend:1.0.0
kubectl expose deployment frontend --type=LoadBalancer --port 80 --target-port 8080