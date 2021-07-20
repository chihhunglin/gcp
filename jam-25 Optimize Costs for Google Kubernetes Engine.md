# Optimize Costs for Google Kubernetes Engine

## Managing a GKE Multi-tenant Cluster with Namespaces
gsutil -m cp -r gs://spls/gsp766/gke-qwiklab ~
cd ~/gke-qwiklab
gcloud config set compute/zone us-central1-a && gcloud container clusters get-credentials multi-tenant-cluster
kubectl get namespace
kubectl api-resources --namespaced=true
kubectl get services --namespace=kube-system
kubectl create namespace team-a && \
kubectl create namespace team-b
kubectl run app-server --image=centos --namespace=team-a -- sleep infinity && \
kubectl run app-server --image=centos --namespace=team-b -- sleep infinity
kubectl get pods -A
kubectl describe pod app-server --namespace=team-a
kubectl config set-context --current --namespace=team-a
kubectl describe pod app-server

### Access Control in Namespaces
gcloud projects add-iam-policy-binding ${GOOGLE_CLOUD_PROJECT} \
--member=serviceAccount:team-a-dev@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com  \
--role=roles/container.clusterViewer
kubectl create role pod-reader \
--resource=pods --verb=watch --verb=get --verb=list
kubectl create -f developer-role.yaml
kubectl create rolebinding team-a-developers \
--role=developer --user=team-a-dev@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com
gcloud iam service-accounts keys create /tmp/key.json --iam-account team-a-dev@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com

gcloud auth activate-service-account  --key-file=/tmp/key.json
gcloud container clusters get-credentials multi-tenant-cluster --zone us-central1-a --project ${GOOGLE_CLOUD_PROJECT}
kubectl get pods --namespace=team-a
kubectl get pods --namespace=team-b
gcloud container clusters get-credentials multi-tenant-cluster --zone us-central1-a --project ${GOOGLE_CLOUD_PROJECT}

kubectl create quota test-quota \
--hard=count/pods=2,count/services.loadbalancers=1 --namespace=team-a
kubectl run app-server-2 --image=centos --namespace=team-a -- sleep infinity
kubectl run app-server-3 --image=centos --namespace=team-a -- sleep infinity
kubectl describe quota test-quota --namespace=team-a
kubectl edit quota test-quota --namespace=team-a
kubectl describe quota test-quota --namespace=team-a

kubectl create -f cpu-mem-quota.yaml
kubectl create -f cpu-mem-demo-pod.yaml --namespace=team-a

gcloud container clusters \
update multi-tenant-cluster --zone us-central1-a \
--resource-usage-bigquery-dataset cluster_dataset

export GCP_BILLING_EXPORT_TABLE_FULL_PATH=${GOOGLE_CLOUD_PROJECT}.billing_dataset.gcp_billing_export_v1_xxxx
export USAGE_METERING_DATASET_ID=cluster_dataset
export COST_BREAKDOWN_TABLE_ID=usage_metering_cost_breakdown

export USAGE_METERING_QUERY_TEMPLATE=~/gke-qwiklab/usage_metering_query_template.sql
export USAGE_METERING_QUERY=cost_breakdown_query.sql
export USAGE_METERING_START_DATE=2020-10-26

sed \
-e "s/\${fullGCPBillingExportTableID}/$GCP_BILLING_EXPORT_TABLE_FULL_PATH/" \
-e "s/\${projectID}/$GOOGLE_CLOUD_PROJECT/" \
-e "s/\${datasetID}/$USAGE_METERING_DATASET_ID/" \
-e "s/\${startDate}/$USAGE_METERING_START_DATE/" \
"$USAGE_METERING_QUERY_TEMPLATE" \
> "$USAGE_METERING_QUERY"

bq query \
--project_id=$GOOGLE_CLOUD_PROJECT \
--use_legacy_sql=false \
--destination_table=$USAGE_METERING_DATASET_ID.$COST_BREAKDOWN_TABLE_ID \
--schedule='every 24 hours' \
--display_name="GKE Usage Metering Cost Breakdown Scheduled Query" \
--replace=true \
"$(cat $USAGE_METERING_QUERY)"

 SELECT *  FROM `[PROJECT-ID].cluster_dataset.usage_metering_cost_breakdown`


## Exploring Cost-optimization for GKE Virtual Machines
gcloud auth list
gcloud config list project
gcloud container clusters get-credentials hello-demo-cluster --zone us-central1-a
kubectl scale deployment hello-server --replicas=2
gcloud container clusters resize hello-demo-cluster --node-pool node \
    --num-nodes 3 --zone us-central1-a

gcloud container node-pools create larger-pool \
  --cluster=hello-demo-cluster \
  --machine-type=e2-standard-2 \
  --num-nodes=1 \
  --zone=us-central1-a

for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=node -o=name); do
  kubectl cordon "$node";
done

for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=node -o=name); do
  kubectl drain --force --ignore-daemonsets --delete-local-data --grace-period=10 "$node";
done

kubectl get pods -o=wide
gcloud container node-pools delete node --cluster hello-demo-cluster --zone us-central1-a

gcloud container clusters create regional-demo --region=us-central1 --num-nodes=1

cat << EOF > pod-1.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  labels:
    security: demo
spec:
  containers:
  - name: container-1
    image: gcr.io/google-samples/hello-app:2.0
EOF

kubectl apply -f pod-1.yaml

cat << EOF > pod-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-2
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - demo
        topologyKey: "kubernetes.io/hostname"
  containers:
  - name: container-2
    image: gcr.io/google-samples/node-hello:1.0
EOF

kubectl apply -f pod-2.yaml

kubectl get pod pod-1 pod-2 --output wide

sed -i 's/podAntiAffinity/podAffinity/g' pod-2.yaml
kubectl delete pod pod-2
kubectl create -f pod-2.yaml

## Understanding and Combining GKE Autoscaling Strategies
gcloud config set compute/zone us-central1-a
gcloud container clusters create scaling-demo --num-nodes=3 --enable-vertical-pod-autoscaling

cat << EOF > php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 3
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
EOF

kubectl apply -f php-apache.yaml

kubectl get deployment
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa

gcloud container clusters describe scaling-demo | grep ^verticalPodAutoscaling -A 1

kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
kubectl get deployment hello-server
kubectl set resources deployment hello-server --requests=cpu=450m
kubectl describe pod hello-server | sed -n "/Containers:$/,/Conditions:/p"

cat << EOF > hello-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hello-server-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind:       Deployment
    name:       hello-server
  updatePolicy:
    updateMode: "Off"
EOF

kubectl apply -f hello-vpa.yaml
kubectl describe vpa hello-server-vpa
sed -i 's/Off/Auto/g' hello-vpa.yaml
kubectl apply -f hello-vpa.yaml

kubectl scale deployment hello-server --replicas=2
kubectl get pods -w

kubectl get hpa
kubectl describe pod hello-server | sed -n "/Containers:$/,/Conditions:/p"
gcloud beta container clusters update scaling-demo --enable-autoscaling --min-nodes 1 --max-nodes 5
gcloud beta container clusters update scaling-demo \
--autoscaling-profile optimize-utilization
kubectl get deployment -n kube-system
kubectl create poddisruptionbudget kube-dns-pdb --namespace=kube-system --selector k8s-app=kube-dns --max-unavailable 1
kubectl create poddisruptionbudget prometheus-pdb --namespace=kube-system --selector k8s-app=prometheus-to-sd --max-unavailable 1
kubectl create poddisruptionbudget kube-proxy-pdb --namespace=kube-system --selector component=kube-proxy --max-unavailable 1
kubectl create poddisruptionbudget metrics-agent-pdb --namespace=kube-system --selector k8s-app=gke-metrics-agent --max-unavailable 1
kubectl create poddisruptionbudget metrics-server-pdb --namespace=kube-system --selector k8s-app=metrics-server --max-unavailable 1
kubectl create poddisruptionbudget fluentd-pdb --namespace=kube-system --selector k8s-app=fluentd-gke --max-unavailable 1
kubectl create poddisruptionbudget backend-pdb --namespace=kube-system --selector k8s-app=glbc --max-unavailable 1
kubectl create poddisruptionbudget kube-dns-autoscaler-pdb --namespace=kube-system --selector k8s-app=kube-dns-autoscaler --max-unavailable 1
kubectl create poddisruptionbudget stackdriver-pdb --namespace=kube-system --selector app=stackdriver-metadata-agent --max-unavailable 1
kubectl create poddisruptionbudget event-pdb --namespace=kube-system --selector k8s-app=event-exporter --max-unavailable 1

kubectl get nodes
gcloud container clusters update scaling-demo \
    --enable-autoprovisioning \
    --min-cpu 1 \
    --min-memory 2 \
    --max-cpu 45 \
    --max-memory 160
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
kubectl get hpa
kubectl get deployment php-apache

cat << EOF > pause-pod.yaml
---
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1
globalDefault: false
description: "Priority class used by overprovisioning."
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      run: overprovisioning
  template:
    metadata:
      labels:
        run: overprovisioning
    spec:
      priorityClassName: overprovisioning
      containers:
      - name: reserve-resources
        image: k8s.gcr.io/pause
        resources:
          requests:
            cpu: 1
            memory: 4Gi
EOF

kubectl apply -f pause-pod.yaml

## GKE Workload Optimization
gcloud config set compute/zone us-central1-a
gcloud container clusters create test-cluster --num-nodes=3  --enable-ip-alias

cat << EOF > gb_frontend_pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: gb-frontend
  name: gb-frontend
spec:
    containers:
    - name: gb-frontend
      image: gcr.io/google-samples/gb-frontend:v5
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
      ports:
      - containerPort: 80
EOF

kubectl apply -f gb_frontend_pod.yaml

### Container-native Load Balancing Through Ingress

cat << EOF > gb_frontend_cluster_ip.yaml
apiVersion: v1
kind: Service
metadata:
  name: gb-frontend-svc
  annotations:
    cloud.google.com/neg: '{"ingress": true}'
spec:
  type: ClusterIP
  selector:
    app: gb-frontend
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
EOF

kubectl apply -f gb_frontend_cluster_ip.yaml

cat << EOF > gb_frontend_ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: gb-frontend-ingress
spec:
  backend:
    serviceName: gb-frontend-svc
    servicePort: 80
EOF

kubectl apply -f gb_frontend_ingress.yaml

BACKEND_SERVICE=$(gcloud compute backend-services list | tail -1 | cut -d ' ' -f1)
gcloud compute backend-services get-health $BACKEND_SERVICE --global
kubectl get ingress gb-frontend-ingress

### Load Testing an Application
gsutil cp -r gs://spls/gsp769/locust-image .
gcloud builds submit \
    --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/locust-tasks:latest locust-image
gcloud container images list
gsutil cp gs://spls/gsp769/locust_deploy.yaml .
sed 's/${GOOGLE_CLOUD_PROJECT}/'$GOOGLE_CLOUD_PROJECT'/g' locust_deploy.yaml | kubectl apply -f -
kubectl get service locust-master

### Setting up a liveness probe

cat << EOF > liveness-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    demo: liveness-probe
  name: liveness-demo-pod
spec:
  containers:
  - name: liveness-demo-pod
    image: centos
    args:
    - /bin/sh
    - -c
    - touch /tmp/alive; sleep infinity
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/alive
      initialDelaySeconds: 5
      periodSeconds: 10
EOF

kubectl apply -f liveness-demo.yaml
kubectl describe pod liveness-demo-pod
kubectl exec liveness-demo-pod -- rm /tmp/alive
kubectl describe pod liveness-demo-pod

### Setting up a readiness probe
cat << EOF > readiness-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    demo: readiness-probe
  name: readiness-demo-pod
spec:
  containers:
  - name: readiness-demo-pod
    image: nginx
    ports:
    - containerPort: 80
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/healthz
      initialDelaySeconds: 5
      periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-demo-svc
  labels:
    demo: readiness-probe
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    demo: readiness-probe
EOF

kubectl apply -f readiness-demo.yaml
kubectl get service readiness-demo-svc
kubectl describe pod readiness-demo-pod
kubectl exec readiness-demo-pod -- touch /tmp/healthz
kubectl describe pod readiness-demo-pod | grep ^Conditions -A 5

### Pod Disruption Budgets
kubectl delete pod gb-frontend

cat << EOF > gb_frontend_deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gb-frontend
  labels:
    run: gb-frontend
spec:
  replicas: 5
  selector:
    matchLabels:
      run: gb-frontend
  template:
    metadata:
      labels:
        run: gb-frontend
    spec:
      containers:
        - name: gb-frontend
          image: gcr.io/google-samples/gb-frontend:v5
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
          ports:
            - containerPort: 80
              protocol: TCP
EOF

kubectl apply -f gb_frontend_deployment.yaml

for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
  kubectl drain --force --ignore-daemonsets --grace-period=10 "$node";
done

kubectl describe deployment gb-frontend | grep ^Replicas

for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
  kubectl uncordon "$node";
done

kubectl describe deployment gb-frontend | grep ^Replicas

kubectl create poddisruptionbudget gb-pdb --selector run=gb-frontend --min-available 4
for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
  kubectl drain --timeout=30s --ignore-daemonsets --grace-period=10 "$node";
done

kubectl describe deployment gb-frontend | grep ^Replicas

## Optimize Costs for Google Kubernetes Engine: Challenge Lab
ZONE=us-central1-a
gcloud container clusters create onlineboutique-cluster \
   --project=${DEVSHELL_PROJECT_ID} --zone=${ZONE} \
    --machine-type=n1-standard-2 --num-nodes=2
kubectl create namespace dev
kubectl create namespace prod
kubectl config set-context --current --namespace dev
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo
kubectl apply -f ./release/kubernetes-manifests.yaml --namespace dev
kubectl get svc -w --namespace dev

### Task 2: Migrate to an Optimized Nodepool
gcloud container node-pools create optimized-pool \
   --cluster=onlineboutique-cluster \
   --machine-type=custom-2-3584 \
   --num-nodes=2 \
   --zone=$ZONE

for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
   kubectl cordon "$node";
done

for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
   kubectl drain --force --ignore-daemonsets --delete-local-data --grace-period=10 "$node";
done

kubectl get pods -o=wide --namespace dev

gcloud container node-pools delete default-pool \
   --cluster onlineboutique-cluster --zone $ZONE

### Task 3: Apply a Frontend Update
kubectl create poddisruptionbudget onlineboutique-frontend-pdb \
--selector app=frontend --min-available 1  --namespace dev
KUBE_EDITOR="vim" kubectl edit deployment/frontend --namespace dev

Change the following in the configuration:
image to gcr.io/qwiklabs-resources/onlineboutique-frontend:v2.1, and
imagePullPolicy to Always

### Task 4: Autoscale from Estimated Traffic
You need to apply horizontal pod autoscaling to your frontend deployment with:

scaling based on a target CPU percentage of 50, and
setting the pod scaling between 1 minimum and 13 maximum.

kubectl autoscale deployment frontend --cpu-percent=50 \
   --min=1 --max=13 --namespace dev
kubectl get hpa --namespace dev
gcloud beta container clusters update onlineboutique-cluster \
   --enable-autoscaling --min-nodes 1 --max-nodes 6 --zone $ZONE
kubectl exec $(kubectl get pod --namespace=dev | grep 'loadgenerator' | cut -f1 -d ' ') \
   -it --namespace=dev -- bash -c "export USERS=8000; sh ./loadgen.sh"
