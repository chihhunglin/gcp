# Build and Secure Networks in Google Cloud

## User Authentication: Identity-Aware Proxy
done before

## Multiple VPC Networks
done before

## VPC Networks - Controlling Access

### In blue and green vm
sudo apt-get install nginx-light -y
sudo vim /var/www/html/index.nginx-debian.html

### test vm
gcloud compute instances create test-vm --machine-type=f1-micro --subnet=default --zone=us-central1-a
curl <Enter blue's internal IP here>
curl -c 3 <Enter green's internal IP here>

Network Admin: Permissions to create, modify, and delete networking resources, except for firewall rules and SSL certificates.
Security Admin: Permissions to create, modify, and delete firewall rules and SSL certificates.

## HTTP Load Balancer with Cloud Armor

### In siege vm
sudo apt-get -y install siege
export LB_IP=34.149.198.217
siege -c 250 http://$LB_IP

## Create an Internal Load Balancer

## Build and Secure Networks in Google Cloud: Challenge Lab
	
