# Deploying Applications

## Deploying a Python Flask Web Application to App Engine Flexible
git clone https://github.com/GoogleCloudPlatform/python-docs-samples.git
cd python-docs-samples/codelabs/flex_and_vision
export PROJECT_ID=qwiklabs-gcp-00-14678ea8b7a9
gcloud iam service-accounts create qwiklab \
  --display-name "My Qwiklab Service Account"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member serviceAccount:qwiklab@${PROJECT_ID}.iam.gserviceaccount.com \
--role roles/owner
gcloud iam service-accounts keys create ~/key.json \
--iam-account qwiklab@${PROJECT_ID}.iam.gserviceaccount.com

export GOOGLE_APPLICATION_CREDENTIALS="/home/${USER}/key.json"

virtualenv -p python3 env
source env/bin/activate
pip install -r requirements.txt

gcloud app create
export CLOUD_STORAGE_BUCKET=${PROJECT_ID}
gsutil mb gs://${PROJECT_ID}


python main.py

vim app.yaml
gcloud config set app/cloud_build_timeout 1000
gcloud app deploy
https://<PROJECT_ID>.appspot.com

## Deploy an ASP.NET Core App to App Engine
vim global.json
{
"sdk": {
    "version": "3.1.401"
    }
}
dotnet --version
export DOTNET_CLI_TELEMETRY_OPTOUT=1
dotnet new razor -o HelloWorldAspNetCore
cd HelloWorldAspNetCore
dotnet run --urls=http://localhost:8080
dotnet publish -c Release
cd bin/Release/netcoreapp3.1/publish/

touch Dockerfile
vim Dockerfile
```
FROM gcr.io/google-appengine/aspnetcore:3.1
ADD ./ /app
ENV ASPNETCORE_URLS=http://*:${PORT}
WORKDIR /app
ENTRYPOINT [ "dotnet", "HelloWorldAspNetCore.dll" ]
```

vim app.yaml
```
env: flex
runtime: custom
```

gcloud app deploy --version v0
gcloud app browse

## Firebase Web
git clone https://github.com/firebase/friendlychat-web

<!-- The core Firebase JS SDK is always required and must be listed first -->
<script src="/__/firebase/8.3.1/firebase-app.js"></script>

<!-- TODO: Add SDKs for Firebase products that you want to use
     https://firebase.google.com/docs/web/setup#available-libraries -->
<script src="/__/firebase/8.3.1/firebase-analytics.js"></script>

<!-- Initialize Firebase -->
<script src="/__/firebase/init.js"></script>


firebase login --no-localhost
cd ~/friendlychat-web/web-start/
firebase use --add
firebase serve --only hosting


## Firebase SDK for Cloud Functions

git clone https://github.com/firebase/friendlychat

<!-- The core Firebase JS SDK is always required and must be listed first -->
<script src="https://www.gstatic.com/firebasejs/8.3.1/firebase-app.js"></script>

<!-- TODO: Add SDKs for Firebase products that you want to use
     https://firebase.google.com/docs/web/setup#available-libraries -->
<script src="https://www.gstatic.com/firebasejs/8.3.1/firebase-analytics.js"></script>

<script>
  // Your web app's Firebase configuration
  // For Firebase JS SDK v7.20.0 and later, measurementId is optional
  var firebaseConfig = {
    apiKey: "AIzaSyD2FNGpdhl7a32Zjmm4TgYvDIzHe-470vw",
    authDomain: "qwiklabs-gcp-02-73510c7a4dac.firebaseapp.com",
    projectId: "qwiklabs-gcp-02-73510c7a4dac",
    storageBucket: "qwiklabs-gcp-02-73510c7a4dac.appspot.com",
    messagingSenderId: "388940652130",
    appId: "1:388940652130:web:6559baf4056924e5003b3b",
    measurementId: "G-J3Y7BN9KFC"
  };
  // Initialize Firebase
  firebase.initializeApp(firebaseConfig);
  firebase.analytics();
</script>

npm install -g firebase
firebase --version
firebase login --no-localhost
firebase use --add
firebase deploy --except functions

cd functions
npm install
cd ..
firebase deploy --only functions

## Deploying Memcached on Kubernetes Engine
gcloud container clusters create demo-cluster --num-nodes 3 --zone us-central1-f
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm install mycache stable/memcached --set replicaCount=3
kubectl get pods
kubectl get service mycache-memcached -o jsonpath="{.spec.clusterIP}" ; echo
kubectl get endpoints mycache-memcached
kubectl run -it --rm alpine --image=alpine:3.6 --restart=Never nslookup mycache-memcached.default.svc.cluster.local

kubectl run -it --rm python --image=python:3.6-alpine --restart=Never python
import socket
print(socket.gethostbyname_ex('mycache-memcached.default.svc.cluster.local'))
exit()

kubectl run -it --rm alpine --image=alpine:3.6 --restart=Never telnet mycache-memcached-0.mycache-memcached.default.svc.cluster.local 11211
set mykey 0 0 5
hello
get mykey
quit

https://cdn.qwiklabs.com/sc0rg%2BcEvqxI8%2BojmPOzhOnlLCCrsw%2F8YxhMAZsmlgw%3D

kubectl run -it --rm python --image=python:3.6-alpine --restart=Never sh
pip install pymemcache
python
```
import socket
from pymemcache.client.hash import HashClient
_, _, ips = socket.gethostbyname_ex('mycache-memcached.default.svc.cluster.local')
servers = [(ip, 11211) for ip in ips]
client = HashClient(servers, use_pooling=True)
client.set('mykey', 'hello')
client.get('mykey')
```
exit()

helm install mycache stable/mcrouter --set memcached.replicaCount=3
kubectl get pods
MCROUTER_POD_IP=$(kubectl get pods -l app=mycache-mcrouter -o jsonpath="{.items[0].status.podIP}")
kubectl run -it --rm alpine --image=alpine:3.6 --restart=Never telnet $MCROUTER_POD_IP 5000
set anotherkey 0 0 15
Mcrouter is fun
get anotherkey
quit

cat <<EOF | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-application-py
spec:
  replicas: 5
  selector:
    matchLabels:
      app: sample-application-py
  template:
    metadata:
      labels:
        app: sample-application-py
    spec:
      containers:
        - name: python
          image: python:3.6-alpine
          command: [ "sh", "-c"]
          args:
          - while true; do sleep 10; done;
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
EOF

kubectl get pods

POD=$(kubectl get pods -l app=sample-application-py -o jsonpath="{.items[0].metadata.name}")
kubectl exec -it $POD -- sh -c 'echo $NODE_NAME'
kubectl run -it --rm alpine --image=alpine:3.6 --restart=Never telnet gke-demo-cluster-default-pool-d5f893bc-mnvq 5000
get anotherkey
quit

kubectl exec -it $POD -- sh
pip install pymemcache
python
```
import os
from pymemcache.client.base import Client

NODE_NAME = os.environ['NODE_NAME']
client = Client((NODE_NAME, 5000))
client.set('some_key', 'some_value')
result = client.get('some_key')
result
```