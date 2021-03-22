# Build and Secure Networks in Google Cloud

## VPC Networks - Controlling Access
sudo apt-get install nginx-light -y
sudo nano /var/www/html/index.nginx-debian.html

### Create the firewall rule

gcloud compute instances create test-vm --machine-type=f1-micro --subnet=default --zone=us-central1-a

gcloud auth activate-service-account --key-file credentials.json
gcloud compute firewall-rules list
gcloud compute firewall-rules delete allow-http-web-server

## HTTP Load Balancer with Cloud Armor
https://cdn.qwiklabs.com/7wJtCqbfTFLwKCpOMzUSyPjVKBjUouWHbduOqMpfRiM%3D

## Create an Internal Load Balancer
https://cdn.qwiklabs.com/k3u04mphJhk%2F2yM84NjgPiZHrbCuzbdwAQ98vnaoHQo%3D

## Build and Secure Networks in Google Cloud: Challenge Lab


https://cdn.qwiklabs.com/N7fffDviBDd2jo81cPmWalK21HJ6Mkq5eCGaPLH5bAQ%3D
