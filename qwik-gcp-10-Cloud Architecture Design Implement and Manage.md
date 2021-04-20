# Cloud Architecture Design Implement and Manage

## Google Cloud Essential Skills: Challenge Lab
sudo apt-get update
sudo apt-get install apache2 -y
echo '<!doctype html><html><body><h1>Hello World!</h1></body></html>' | sudo tee /var/www/html/index.html

## Deploy a Compute Instance with a Remote Startup Script

## Configure a Firewall and a Startup Script with Deployment Manager
mkdir deployment_manager
cd deployment_manager
gsutil cp gs://spls/gsp302/* .


resources:
- type: compute.v1.instance
  name: vm-test
  properties:
    zone: {{ properties["zone"] }}
    machineType: https://www.googleapis.com/compute/v1/projects/{{ env["project"] }}/zones/{{ properties["zone"] }}/machineTypes/f1-micro
    disks:
    - deviceName: boot
      type: PERSISTENT
      boot: true
      autoDelete: true
      initializeParams:
        diskName: disk-{{ env["deployment"] }}
        sourceImage: https://www.googleapis.com/compute/v1/projects/debian-cloud/global/images/family/debian-9
    networkInterfaces:
    - network: https://www.googleapis.com/compute/v1/projects/{{ env["project"] }}/global/networks/default
      accessConfigs:
      - name: External NAT
        type: ONE_TO_ONE_NAT
    metadata:
      items:
      - key: startup-script
        value: |
          #!/bin/bash
          apt-get update
          apt-get install -y apache2
    tags:
      items:
      - http
    serviceAccounts:
    - email: <YOUR-SERVICE-ACCOUNT-EMAIL>
      scopes:
      - https://www.googleapis.com/auth/devstorage.read_only
      - https://www.googleapis.com/auth/logging.write
      - https://www.googleapis.com/auth/monitoring.write
      - https://www.googleapis.com/auth/servicecontrol
      - https://www.googleapis.com/auth/service.management.readonly
      - https://www.googleapis.com/auth/trace.append
- type: compute.v1.firewall
  name: default-allow-http
  properties:
    network: https://www.googleapis.com/compute/v1/projects/{{ env["project"] }}/global/networks/default
    targetTags: 
    - http
    allowed:
    - IPProtocol: tcp
      ports: 
      - '80'
    sourceRanges: 
    - 0.0.0.0/0

view rawqwiklabs.jinja 
gcloud deployment-manager deployments create vm-test --config=qwiklabs.yaml
