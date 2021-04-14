# Kubernetes Solutions

## Managing Deployments Using Kubernetes Engine
Done

## Using Kubernetes Engine to Deploy Apps with Regional Persistent Disks
CLUSTER_VERSION=$(gcloud container get-server-config --region us-west1 --format='value(validMasterVersions[0])')
export CLOUDSDK_CONTAINER_USE_V1_API_CLIENT=false
gcloud container clusters create repd \
  --cluster-version=${CLUSTER_VERSION} \
  --machine-type=n1-standard-4 \
  --region=us-west1 \
  --num-nodes=1 \
  --node-locations=us-west1-a,us-west1-b,us-west1-c
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-cluster-rule \
  --clusterrole=cluster-admin \
  --serviceaccount=kube-system:tiller
helm init --service-account=tiller
until (helm version --tiller-connection-timeout=1 >/dev/null 2>&1); do echo "Waiting for tiller install..."; sleep 2; done && echo "Helm install complete"

kubectl apply -f - <<EOF
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: repd-west1-a-b-c
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
  replication-type: regional-pd
  zones: us-west1-a, us-west1-b, us-west1-c
EOF

kubectl get storageclass

kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: data-wp-repd-mariadb-0
  namespace: default
  labels:
    app: mariadb
    component: master
    release: wp-repd
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 8Gi
  storageClassName: standard
EOF

kubectl apply -f - <<EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: wp-repd-wordpress
  namespace: default
  labels:
    app: wp-repd-wordpress
    chart: wordpress-5.7.1
    heritage: Tiller
    release: wp-repd
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 200Gi
  storageClassName: repd-west1-a-b-c
EOF

kubectl get persistentvolumeclaims

helm install --name wp-repd \
  --set smtpHost= --set smtpPort= --set smtpUser= \
  --set smtpPassword= --set smtpUsername= --set smtpProtocol= \
  --set persistence.storageClass=repd-west1-a-b-c \
  --set persistence.existingClaim=wp-repd-wordpress \
  --set persistence.accessMode=ReadOnlyMany \
  stable/wordpress

kubectl get pods

while [[ -z $SERVICE_IP ]]; do SERVICE_IP=$(kubectl get svc wp-repd-wordpress -o jsonpath='{.status.loadBalancer.ingress[].ip}'); echo "Waiting for service external IP..."; sleep 2; done; echo http://$SERVICE_IP/admin

while [[ -z $PV ]]; do PV=$(kubectl get pvc wp-repd-wordpress -o jsonpath='{.spec.volumeName}'); echo "Waiting for PV..."; sleep 2; done

kubectl describe pv $PV

echo http://$SERVICE_IP/admin

cat - <<EOF
Username: user
Password: $(kubectl get secret --namespace default wp-repd-wordpress -o jsonpath="{.data.wordpress-password}" | base64 --decode)
EOF

NODE=$(kubectl get pods -l app.kubernetes.io/instance=wp-repd  -o jsonpath='{.items..spec.nodeName}')

ZONE=$(kubectl get node $NODE -o jsonpath="{.metadata.labels['failure-domain\.beta\.kubernetes\.io/zone']}")

IG=$(gcloud compute instance-groups list --filter="name~gke-repd-default-pool zone:(${ZONE})" --format='value(name)')

echo "Pod is currently on node ${NODE}"

echo "Instance group to delete: ${IG} for zone: ${ZONE}"

kubectl get pods -l app.kubernetes.io/instance=wp-repd -o wide

gcloud compute instance-groups managed delete ${IG} --zone ${ZONE}

kubectl get pods -l app.kubernetes.io/instance=wp-repd -o wide

echo http://$SERVICE_IP/admin

## NGINX Ingress Controller on Google Kubernetes Engine
gcloud compute zones list
gcloud config set compute/zone us-central1-a
gcloud container clusters create nginx-tutorial --num-nodes 2
helm version
helm repo add stable https://charts.helm.sh/stable
helm repo update
kubectl create deployment hello-app --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment hello-app  --port=8080
helm install nginx-ingress stable/nginx-ingress --set rbac.create=true
kubectl get service
kubectl get service nginx-ingress-controller
touch ingress-resource.yaml
vim ingress-resource.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-resource
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /hello
        backend:
          serviceName: hello-app
          servicePort: 8080

kubectl apply -f ingress-resource.yaml
kubectl get ingress ingress-resource

## Distributed Load Testing Using Kubernetes
PROJECT=$(gcloud config get-value project)
REGION=us-central1
ZONE=${REGION}-a
CLUSTER=gke-load-test
TARGET=${PROJECT}.appspot.com
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
gsutil -m cp -r gs://spls/gsp182/distributed-load-testing-using-kubernetes .
cd distributed-load-testing-using-kubernetes/
gcloud builds submit --tag gcr.io/$PROJECT/locust-tasks:latest docker-image/.
gcloud app deploy sample-webapp/app.yaml
gcloud container clusters create $CLUSTER \
  --zone $ZONE \
  --num-nodes=5

### locust
sed -i -e "s/\[TARGET_HOST\]/$TARGET/g" kubernetes-config/locust-master-controller.yaml
sed -i -e "s/\[TARGET_HOST\]/$TARGET/g" kubernetes-config/locust-worker-controller.yaml
sed -i -e "s/\[PROJECT_ID\]/$PROJECT/g" kubernetes-config/locust-master-controller.yaml
sed -i -e "s/\[PROJECT_ID\]/$PROJECT/g" kubernetes-config/locust-worker-controller.yaml

kubectl apply -f kubernetes-config/locust-master-controller.yaml
kubectl get pods -l app=locust-master
kubectl apply -f kubernetes-config/locust-master-service.yaml
kubectl get svc locust-master
kubectl apply -f kubernetes-config/locust-worker-controller.yaml
kubectl get pods -l app=locust-worker
kubectl scale deployment/locust-worker --replicas=20
kubectl get pods -l app=locust-worker
EXTERNAL_IP=$(kubectl get svc locust-master -o yaml | grep ip | awk -F": " '{print $NF}')
echo http://$EXTERNAL_IP:8089

## Running Dedicated Game Servers in Google Kubernetes Engine
sudo apt-get update
sudo apt-get -y install kubectl
sudo apt-get -y install \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common
curl -fsSL https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/$(. /etc/os-release; echo "$ID")    $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get -y install docker-ce
sudo usermod -aG docker $USER
docker run hello-world
gsutil -m cp -r gs://spls/gsp133/gke-dedicated-game-server .
export GCR_REGION=us PROJECT_ID=qwiklabs-gcp-02-bf59ea2c56a9
printf "$GCR_REGION \n$PROJECT_ID\n"
cd gke-dedicated-game-server/openarena
docker build -t \
${GCR_REGION}.gcr.io/${PROJECT_ID}/openarena:0.8.8 .
gcloud docker -- push \
  ${GCR_REGION}.gcr.io/${PROJECT_ID}/openarena:0.8.8
region=us-east1
zone_1=${region}-b
gcloud config set compute/region ${region}
gcloud compute instances create openarena-asset-builder \
   --machine-type f1-micro \
   --image-family debian-9 \
   --image-project debian-cloud \
   --zone ${zone_1}
gcloud compute disks create openarena-assets \
   --size=50GB --type=pd-ssd\
   --description="OpenArena data disk. Mount read-only at
/usr/share/games/openarena/baseoa/" \
   --zone ${zone_1}
gcloud compute instances attach-disk openarena-asset-builder \
   --disk openarena-assets --zone ${zone_1}

###  openarena-asset-builder
sudo lsblk
sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb
sudo mkdir -p /usr/share/games/openarena/baseoa/

sudo mount -o discard,defaults /dev/sdb \
    /usr/share/games/openarena/baseoa/
sudo apt-get update
sudo apt-get -y install openarena-server

sudo gsutil cp gs://qwiklabs-assets/single-match.cfg /usr/share/games/openarena/baseoa/single-match.cfg

sudo umount -f -l /usr/share/games/openarena/baseoa/
sudo shutdown -h now

### 
echo $zone_1
region=us-east1
zone_1=${region}-b
gcloud compute instances delete openarena-asset-builder --zone ${zone_1}

gcloud compute networks create game
gcloud compute firewall-rules create openarena-dgs --network game \
    --allow udp:27961-28061
gcloud container clusters create openarena-cluster \
   --num-nodes 3 \
   --network game \
   --machine-type=n1-standard-2 \
   --zone=${zone_1}
gcloud container clusters get-credentials openarena-cluster --zone ${zone_1}
kubectl apply -f k8s/asset-volume.yaml
kubectl apply -f k8s/asset-volumeclaim.yaml
kubectl get persistentVolume
kubectl get persistentVolumeClaim
export GCR_REGION=us
export PROJECT_ID=qwiklabs-gcp-02-bf59ea2c56a9
cd ../scaling-manager
chmod +x build-and-push.sh
source ./build-and-push.sh
gcloud compute instance-groups managed list
gke-openarena-cluster-default-pool-1d0e7cf6-grp
export GKE_BASE_INSTANCE_NAME=gke-openarena-cluster-default-pool-1d0e7cf6-grp
export GCP_ZONE=us-east1-b
printf "$GCR_REGION \n$PROJECT_ID \n$GKE_BASE_INSTANCE_NAME \n$GCP_ZONE \n"
sed -i "s/\[GCR_REGION\]/$GCR_REGION/g" k8s/openarena-scaling-manager-deployment.yaml
sed -i "s/\[PROJECT_ID\]/$PROJECT_ID/g" k8s/openarena-scaling-manager-deployment.yaml
sed -i "s/\[ZONE\]/$GCP_ZONE/g" k8s/openarena-scaling-manager-deployment.yaml
sed -i "s/\gke-openarena-cluster-default-pool-\[REPLACE_ME\]/$GKE_BASE_INSTANCE_NAME/g" k8s/openarena-scaling-manager-deployment.yaml
kubectl apply -f k8s/openarena-scaling-manager-deployment.yaml
kubectl get pods
cd ..
sed -i "s/\[GCR_REGION\]/$GCR_REGION/g" openarena/k8s/openarena-pod.yaml
sed -i "s/\[PROJECT_ID\]/$PROJECT_ID/g" openarena/k8s/openarena-pod.yaml
kubectl apply -f openarena/k8s/openarena-pod.yaml
kubectl get pods
export NODE_NAME=$(kubectl get pod openarena.dgs \
    -o jsonpath="{.spec.nodeName}")
export DGS_IP=$(gcloud compute instances list \
    --filter="name=( ${NODE_NAME} )" \
    --format='table[no-heading](EXTERNAL_IP)')

printf "Node Name: $NODE_NAME \nNode IP  : $DGS_IP \nPort         : 27961\n"
printf " launch client with: \nopenarena +connect $DGS_IP +set net_port 27961\n"

source ./scaling-manager/tests/test-loader.sh

## Awwvision: Cloud Vision API from a Kubernetes Cluster
gcloud config set compute/zone us-central1-a
gcloud container clusters create awwvision \
    --num-nodes 2 \
    --scopes cloud-platform
gcloud container clusters get-credentials awwvision
kubectl cluster-info
sudo apt-get update
sudo apt-get install virtualenv
virtualenv -p python3 venv
source venv/bin/activate
gsutil -m cp -r gs://spls/gsp066/cloud-vision .
cd cloud-vision/python/awwvision
make all
kubectl get pods
kubectl get deployments -o wide
kubectl get svc awwvision-webapp

## Running a MongoDB Database in Kubernetes with StatefulSets
gcloud config set compute/zone us-central1-f
gcloud container clusters create hello-world
gsutil -m cp -r gs://spls/gsp022/mongo-k8s-sidecar .
cd ./mongo-k8s-sidecar/example/StatefulSet/
cat googlecloud_ssd.yaml
kubectl apply -f googlecloud_ssd.yaml
nano mongo-statefulset.yaml
kubectl apply -f mongo-statefulset.yaml
kubectl get statefulset
kubectl get pods
kubectl exec -ti mongo-0 mongo
rs.initiate()
rs.conf()
kubectl scale --replicas=5 statefulset mongo
kubectl get pods
kubectl scale --replicas=3 statefulset mongo
kubectl get pods
kubectl delete statefulset mongo
kubectl delete svc mongo
kubectl delete pvc -l role=mongo
gcloud container clusters delete "hello-world"

##　Deploy a Web App on GKE with HTTPS Redirect using Lets Encrypt
