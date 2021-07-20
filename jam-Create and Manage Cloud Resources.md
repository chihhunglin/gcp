# Create and Manage Cloud Resources

https://github.com/chihhunglin/gcp/blob/master/qwik-gcp-01-Create%20and%20Manage%20Cloud%20Resources.md

## Create and Manage Cloud Resources: Challenge Lab

Task 1: Create a project jumphost instance
You will use this instance to perform maintenance for the project.

Requirements:

Name the instance nucleus-jumphost.
Use an f1-micro machine type.
Use the default image type (Debian Linux).

Task 2: Create a Kubernetes service cluster

The team is building an application that will use a service running on Kubernetes. You need to:

Create a cluster (in the us-east1-b zone) to host the service.
Use the Docker container hello-app (gcr.io/google-samples/hello-app:2.0) as a place holder; the team will replace the container with their own work later.
Expose the app on port 8080.


```bash
gcloud config set compute/zone us-east1-b
gcloud container clusters create my-cluster
gcloud container clusters get-credentials my-cluster
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:2.0
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
gcloud container clusters delete
```

Task 3: Set up an HTTP load balancer
You will serve the site via nginx web servers, but you want to ensure that the environment is fault-tolerant. Create an HTTP load balancer with a managed instance group of 2 nginx web servers. Use the following code to configure the web servers; the team will replace this with their own configuration later.

You need to:

Create an instance template.
Create a target pool.
Create a managed instance group.
Create a firewall rule to allow traffic (80/tcp).
Create a health check.
Create a backend service, and attach the managed instance group.
Create a URL map, and target the HTTP proxy to route requests to your URL map.
Create a forwarding rule.

```bash
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF

gcloud compute instance-templates create nginx-template \
   --metadata-from-file startup-script=startup.sh

gcloud compute target-pools create nginx-pool

gcloud compute instance-groups managed create nginx-group \
   --base-instance-name nginx \
   --size=2 \
   --template=nginx-template \
   --target-pool nginx-pool

gcloud compute firewall-rules create www-firewall \
    --allow tcp:80

gcloud compute forwarding-rules create nginx-lb \
    --region us-east1 \
    --ports=80 \
    --target-pool nginx-pool

gcloud compute http-health-checks create http-basic-check

gcloud compute instance-groups managed set-named-ports nginx-group --named-ports http:80

gcloud compute backend-services create nginx-backend \
    --protocol=HTTP \
    --http-health-checks http-basic-check \
    --global

gcloud compute backend-services add-backend nginx-backend \
    --instance-group=nginx-group \
    --instance-group-zone=us-east1-b \
    --global

gcloud compute url-maps create web-map \
    --default-service nginx-backend

gcloud compute target-http-proxies create http-lb-proxy \
    --url-map web-map

gcloud compute forwarding-rules create http-content-rule \
    --global \
    --target-http-proxy=http-lb-proxy \
    --ports=80        
```

wait 5 mins