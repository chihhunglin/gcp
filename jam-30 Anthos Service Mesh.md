# Anthos Service Mesh

## Installing the Istio on GKE Add-On with Kubernetes Engine
export CLUSTER_NAME=central
export CLUSTER_ZONE=us-central1-b
export CLUSTER_VERSION=latest
gcloud beta container clusters create $CLUSTER_NAME \
    --zone $CLUSTER_ZONE --num-nodes 4 \
    --machine-type "n1-standard-2" --image-type "COS" \
    --cluster-version=$CLUSTER_VERSION --enable-ip-alias \
    --addons=Istio --istio-config=auth=MTLS_STRICT
export GCLOUD_PROJECT=$(gcloud config get-value project)

gcloud container clusters get-credentials $CLUSTER_NAME \
    --zone $CLUSTER_ZONE --project $GCLOUD_PROJECT
kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)

gcloud container clusters list
kubectl get service -n istio-system
kubectl get pods -n istio-system


export LAB_DIR=$HOME/bookinfo-lab
export ISTIO_VERSION=1.4.6

mkdir $LAB_DIR
cd $LAB_DIR

curl -L https://git.io/getLatestIstio | ISTIO_VERSION=$ISTIO_VERSION sh -

cd ./istio-*

export PATH=$PWD/bin:$PATH

istioctl version

cat samples/bookinfo/platform/kube/bookinfo.yaml
istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
cat samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get services
kubectl get pods
kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') \
    -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
kubectl get gateway
kubectl get svc istio-ingressgateway -n istio-system

## Installing Anthos Service Mesh on Google Kubernetes Engine
export PROJECT_ID=$(gcloud config get-value project)
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} \
    --format="value(projectNumber)")
export CLUSTER_NAME=central
export CLUSTER_ZONE=us-central1-b
export WORKLOAD_POOL=${PROJECT_ID}.svc.id.goog
export MESH_ID="proj-${PROJECT_NUMBER}"
gcloud projects get-iam-policy $PROJECT_ID \
    --flatten="bindings[].members" \
    --filter="bindings.members:user:$(gcloud config get-value core/account 2>/dev/null)"
gcloud config set compute/zone ${CLUSTER_ZONE}
gcloud beta container clusters create ${CLUSTER_NAME} \
    --machine-type=n1-standard-4 \
    --num-nodes=4 \
    --workload-pool=${WORKLOAD_POOL} \
    --enable-stackdriver-kubernetes \
    --subnetwork=default \
    --release-channel=regular \
    --labels mesh_id=${MESH_ID}

kubectl auth can-i '*' '*' --all-namespaces

gcloud iam service-accounts create connect-sa
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
 --member="serviceAccount:connect-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
 --role="roles/gkehub.connect"
gcloud iam service-accounts keys create connect-sa-key.json \
  --iam-account=connect-sa@${PROJECT_ID}.iam.gserviceaccount.com
gcloud container hub memberships register ${CLUSTER_NAME}-connect \
   --gke-cluster=${CLUSTER_ZONE}/${CLUSTER_NAME}  \
   --service-account-key-file=./connect-sa-key.json

curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9 > install_asm
curl https://storage.googleapis.com/csm-artifacts/asm/install_asm_1.9.sha256 > install_asm.sha256
chmod +x install_asm
./install_asm \
  --project_id $PROJECT_ID \
  --cluster_name $CLUSTER_NAME \
  --cluster_location us-central1-b \
  --mode install \
  --enable_all \
  --output_dir asm
kubectl label namespace default  istio-injection- istio.io/rev=asm-196-2 --overwrite
cd ~/asm/istio-1.9.6-asm.2/
cat samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
cat samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl get services
kubectl get pods
kubectl exec -it $(kubectl get pod -l app=ratings \
    -o jsonpath='{.items[0].metadata.name}') \
    -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
kubectl get gateway
kubectl get svc istio-ingressgateway -n istio-system



## Observing Services using Prometheus, Grafana, Jaeger, and Kiali

## Traffic Management with Anthos Service Mesh
export CLUSTER_NAME=gke
export CLUSTER_ZONE=us-central1-b
export GCLOUD_PROJECT=$(gcloud config get-value project)
gcloud container clusters get-credentials $CLUSTER_NAME \
    --zone $CLUSTER_ZONE --project $GCLOUD_PROJECT
gcloud container clusters list
kubectl get pods -n istio-system
kubectl get service -n istio-system
kubectl get pods -n asm-system
kubectl get pods
kubectl get services
sudo apt install siege
kubectl describe svc istio-ingressgateway -n istio-system
export GATEWAY_URL=$(kubectl get svc istio-ingressgateway \
-o=jsonpath='{.status.loadBalancer.ingress[0].ip}' -n istio-system)
echo The gateway address is $GATEWAY_URL
siege http://${GATEWAY_URL}/productpage

export CLUSTER_NAME=gke
export CLUSTER_ZONE=us-central1-b
export GCLOUD_PROJECT=$(gcloud config get-value project)
gcloud container clusters get-credentials $CLUSTER_NAME \
    --zone $CLUSTER_ZONE --project $GCLOUD_PROJECT
export GATEWAY_URL=$(kubectl get svc istio-ingressgateway \
-o=jsonpath='{.status.loadBalancer.ingress[0].ip}' -n istio-system)
kubectl describe gateway bookinfo-gateway
kubectl describe virtualservices bookinfo
kubectl exec -it \
$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') \
-c ratings -- curl productpage:9080/productpage \
| grep -o "<title>.*</title>"
curl -I http://${GATEWAY_URL}/productpage

kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/destination-rule-all.yaml
kubectl get destinationrules
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/virtual-service-all-v1.yaml
kubectl get virtualservices
echo $GATEWAY_URL
kubectl apply -f https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
curl -c cookies.txt -F "username=jason" -L -X \
    POST http://$GATEWAY_URL/login
cookie_info=$(grep -Eo "session.*" ./cookies.txt)
cookie_name=$(echo $cookie_info | cut -d' ' -f1)
cookie_value=$(echo $cookie_info | cut -d' ' -f2)
siege -c 5 http://$GATEWAY_URL/productpage \
    --header "Cookie: $cookie_name=$cookie_value"

## Managing Policies and Security with Istio and Citadel
export CLUSTER_NAME=central
export CLUSTER_ZONE=us-central1-b
export GCLOUD_PROJECT=$(gcloud config get-value project)
gcloud container clusters get-credentials $CLUSTER_NAME \
    --zone $CLUSTER_ZONE --project $GCLOUD_PROJECT
gcloud container clusters list
kubectl get service -n istio-system
kubectl get pods -n istio-system
export LAB_DIR=$HOME/security-lab
export ISTIO_VERSION=1.5.2
mkdir $LAB_DIR
cd $LAB_DIR
curl -L https://git.io/getLatestIstio | ISTIO_VERSION=$ISTIO_VERSION sh -
cd ./istio-*
export PATH=$PWD/bin:$PATH
istioctl version
cd $LAB_DIR
git clone https://github.com/GoogleCloudPlatform/istio-samples.git
cd istio-samples/security-intro
mkdir ./hipstershop
curl -o ./hipstershop/kubernetes-manifests.yaml https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/release/kubernetes-manifests.yaml
cat ./hipstershop/kubernetes-manifests.yaml
istioctl kube-inject -f hipstershop/kubernetes-manifests.yaml -o ./hipstershop/kubernetes-manifests-withistio.yaml
kubectl apply -f ./hipstershop/kubernetes-manifests-withistio.yaml
curl -o ./hipstershop/istio-manifests.yaml https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/master/release/istio-manifests.yaml
kubectl apply -f hipstershop/istio-manifests.yaml
kubectl get pods -n default
kubectl get -n istio-system service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

### Enable mTLS for one service: frontend
export LAB_DIR=$HOME/security-lab
cd $LAB_DIR/istio-samples/security-intro
cat ./manifests/mtls-frontend.yaml
kubectl apply -f ./manifests/mtls-frontend.yaml
kubectl exec $(kubectl get pod -l app=productcatalogservice -o jsonpath={.items..metadata.name}) -c istio-proxy -- curl http://frontend:80/ -o /dev/null -s -w '%{http_code}\n'
kubectl exec $(kubectl get pod -l app=productcatalogservice -o jsonpath={.items..metadata.name}) -c istio-proxy \
-- curl https://frontend:80/ -o /dev/null -s -w '%{http_code}\n'  --key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k

### Enable mTLS for an entire namespace: default

cat ./manifests/mtls-default-ns.yaml
kubectl apply -f ./manifests/mtls-default-ns.yaml

### Enable end-user JWT authentication alongside mTLS
cat ./manifests/jwt-frontend-request.yaml
kubectl apply -f ./manifests/jwt-frontend-request.yaml

kubectl exec $(kubectl get pod -l app=productcatalogservice -o jsonpath={.items..metadata.name}) -c istio-proxy \
-- curl  https://frontend:80/ -o /dev/null -s -w '%{http_code}\n' \
--key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k

cat ./manifests/jwt-frontend-authz.yaml
kubectl apply -f manifests/jwt-frontend-authz.yaml
kubectl exec $(kubectl get pod -l app=productcatalogservice -o jsonpath={.items..metadata.name}) -c istio-proxy \
-- curl  https://frontend:80/ -o /dev/null -s -w '%{http_code}\n' \
--key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k

TOKEN=$(curl -k https://raw.githubusercontent.com/istio/istio/release-1.4/security/tools/jwt/samples/demo.jwt -s); echo $TOKEN
kubectl exec $(kubectl get pod -l app=productcatalogservice -o jsonpath={.items..metadata.name}) -c istio-proxy \
-- curl --header "Authorization: Bearer $TOKEN" https://frontend:80/ -o /dev/null -s -w '%{http_code}\n' \
--key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k

kubectl delete -f manifests/jwt-frontend-authz.yaml
kubectl delete -f manifests/jwt-frontend-request.yaml

### Understand Istio authorization and enable frontend authorization
Authentication refers to the who by providing strong identity and secure service-to-service and end-user-to-service communication.

Authorization refers to the what: what a service or user is allowed to do.

Istio Authorization is designed to be simple, flexible, and high performance. Istio authorization policies are defined using .yaml and enforced by Envoy proxies running an authorization engine.

AuthorizationPolicy resources include a selector and a list of rules. The selector specifies the target: which specifies to which namespace and/or workload(s) the policy applies.

The rules in an AuthorizationPolicy specify who is allowed to do what under which conditions. Rules use from, to, and when lists.

### Enable authorization for one service: frontend
kubectl apply -f ./manifests/authz-frontend.yaml
kubectl exec $(kubectl get pod -l app=productcatalogservice -o jsonpath={.items..metadata.name}) -c istio-proxy \
-- curl  --header "Authorization: Bearer $TOKEN" https://frontend:80/ -o /dev/null -s -w '%{http_code}\n' \
--key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k
kubectl exec $(kubectl get pod -l app=productcatalogservice -o jsonpath={.items..metadata.name}) -c istio-proxy \
-- curl --header "Authorization: Bearer $TOKEN" --header "hello:world"  \
   https://frontend:80/ -o /dev/null -s -w '%{http_code}\n' \
  --key /etc/certs/key.pem --cert /etc/certs/cert-chain.pem --cacert /etc/certs/root-cert.pem -k

	