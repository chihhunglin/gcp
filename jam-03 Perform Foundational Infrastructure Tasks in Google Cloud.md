# 03 Perform Foundational Infrastructure Tasks in Google Cloud

## Cloud Monitoring: Qwik Start
### IN VM
curl -sSO https://dl.google.com/cloudagents/add-monitoring-agent-repo.sh
sudo bash add-monitoring-agent-repo.sh
sudo apt-get update
sudo apt-get install stackdriver-agent
curl -sSO https://dl.google.com/cloudagents/add-logging-agent-repo.sh
sudo bash add-logging-agent-repo.sh
sudo apt-get update
sudo apt-get install google-fluentd

## Cloud Functions: Qwik Start - Console

## Google Cloud Pub/Sub: Qwik Start - Console

## Perform Foundational Infrastructure Tasks in Google Cloud: Challenge Lab
Task 1: Create a bucket

Task 2: Create a Pub/Sub topic

Task 3: Create the thumbnail Cloud Function
Entrypoint: thumbnail
node version: 10