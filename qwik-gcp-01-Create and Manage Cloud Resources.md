# Create and manage cloud resources

## Creating a Virtual Machine

```bash
gcloud auth list
gcloud config list project
gcloud compute instances create gcelab2 --machine-type n1-standard-2 --zone us-central1-c
gcloud config set compute/zone ...
gcloud config set compute/region ...
gcloud compute ssh gcelab2 --zone us-central1-c
```

## Getting Started with Cloud Shell & gcloud

```bash
gcloud compute project-info describe --project <your_project_ID>
export PROJECT_ID=<your_project_ID>
export ZONE=<your_zone>
gcloud compute instances create gcelab2 --machine-type n1-standard-2 --zone $ZONE
gcloud config list --all
gcloud components list
gcloud beta interactive
```

## Kubernetes Engine: Qwik Start

When you run a Kubernetes Engine cluster, you also gain the benefit of advanced cluster management features that Google Cloud provides. These include:

- Load-balancing for Compute Engine instances.
- Node Pools to designate subsets of nodes within a cluster for additional flexibility.
- Automatic scaling of your cluster's node instance count.
- Automatic upgrades for your cluster's node software.
- Node auto-repair to maintain node health and availability.
- Logging and Monitoring with Cloud Monitoring for visibility into your cluster.

```bash
gcloud config list project
gcloud container clusters create [CLUSTER-NAME]
gcloud container clusters get-credentials [CLUSTER-NAME]
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
gcloud container clusters delete
```

## Set Up Network and HTTP Load Balancers

There are several ways you can load balance in Google Cloud. This lab takes you through the setup of the following load balancers.:

### L4 Network Load Balancer

```bash
gcloud compute addresses create network-lb-ip-1 \
 --region us-central1
gcloud compute http-health-checks create basic-check
gcloud compute target-pools create www-pool \
    --region us-central1 --http-health-check basic-check
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF
gcloud compute instance-templates create nginx-template \
         --metadata-from-file startup-script=startup.sh
gcloud compute instance-groups managed create nginx-group \
         --base-instance-name nginx \
         --size 2 \
         --template nginx-template \
         --target-pool www-pool
gcloud compute instances list

gcloud compute forwarding-rules create www-rule \
    --region us-central1 \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
gcloud compute forwarding-rules describe www-rule --region us-central1
while true; do curl -m1 IP_ADDRESS; done
```

### L7 HTTP(s) Load Balancer

```bash
gcloud compute instance-templates create lb-backend-template \
   --region=us-central1 \
   --network=default \
   --subnet=default \
   --tags=allow-health-check \
   --image-family=debian-9 \
   --image-project=debian-cloud \
   --metadata=startup-script='#! /bin/bash
     apt-get update
     apt-get install apache2 -y
     a2ensite default-ssl
     a2enmod ssl
     vm_hostname="$(curl -H "Metadata-Flavor:Google" \
     http://169.254.169.254/computeMetadata/v1/instance/name)"
     echo "Page served from: $vm_hostname" | \
     tee /var/www/html/index.html
     systemctl restart apache2'
gcloud compute instance-groups managed create lb-backend-group \
   --template=lb-backend-template --size=2 --zone=us-central1-a
gcloud compute firewall-rules create fw-allow-health-check \
    --network=default \
    --action=allow \
    --direction=ingress \
    --source-ranges=130.211.0.0/22,35.191.0.0/16 \
    --target-tags=allow-health-check \
    --rules=tcp:80
gcloud compute addresses create lb-ipv4-1 \
    --ip-version=IPV4 \
    --global
gcloud compute addresses describe lb-ipv4-1 \
    --format="get(address)" \
    --global
gcloud compute health-checks create http http-basic-check \
    --port 80
gcloud compute backend-services create web-backend-service \
    --protocol=HTTP \
    --port-name=http \
    --health-checks=http-basic-check \
    --global
gcloud compute backend-services add-backend web-backend-service \
    --instance-group=lb-backend-group \
    --instance-group-zone=us-central1-b \
    --global
gcloud compute url-maps create web-map-http \
    --default-service web-backend-service
gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map-http
gcloud compute forwarding-rules create http-content-rule \
    --address=lb-ipv4-1\
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80        
```