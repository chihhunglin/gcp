# Perform Foundational Infrastructure Tasks

## Cloud Storage: Qwik Start - CLI/SDK

https://cloud.google.com/storage/docs/gsutil?hl=zh-tw

```bash
gsutil mb gs://YOUR-BUCKET-NAME/
wget --output-document ada.jpg https://upload.wikimedia.org/wikipedia/commons/thumb/a/a4/Ada_Lovelace_portrait.jpg/800px-Ada_Lovelace_portrait.jpg
gsutil cp ada.jpg gs://YOUR-BUCKET-NAME
gsutil cp -r gs://doro-qwik/ada.jpg .
gsutil cp gs://YOUR-BUCKET-NAME/ada.jpg gs://YOUR-BUCKET-NAME/image-folder/
gsutil ls gs://YOUR-BUCKET-NAME
gsutil acl ch -u AllUsers:R gs://doro-qwik/ada.jpg
gsutil acl ch -d AllUsers gs://YOUR-BUCKET-NAME/ada.jpg
gsutil rm gs://YOUR-BUCKET-NAME/ada.jpg
```

## Cloud Monitoring: Qwik Start

Navigation menu > Monitoring.

```bash
curl -sSO https://dl.google.com/cloudagents/add-monitoring-agent-repo.sh
sudo bash add-monitoring-agent-repo.sh
curl -sSO https://dl.google.com/cloudagents/add-logging-agent-repo.sh
sudo bash add-logging-agent-repo.sh
sudo apt-get update
sudo apt-get install stackdriver-agent
sudo apt-get install google-fluentd
```

## Cloud Functions: Qwik Start - Console

javascript serverless

```bash
mkdir gcf_hello_world
cd gcf_hello_world
nano index.js
/**
* Background Cloud Function to be triggered by Pub/Sub.
* This function is exported by index.js, and executed when
* the trigger topic receives a message.
*
* @param {object} data The event payload.
* @param {object} context The event metadata.
*/
exports.helloWorld = (data, context) => {
const pubSubMessage = data;
const name = pubSubMessage.data
    ? Buffer.from(pubSubMessage.data, 'base64').toString() : "Hello World";

console.log(`My Cloud Function: ${name}`);
};
gsutil mb -p [PROJECT_ID] gs://[BUCKET_NAME]
gcloud functions deploy helloWorld \
  --stage-bucket [BUCKET_NAME] \
  --trigger-topic hello_world \
  --runtime nodejs10
gcloud functions describe helloWorld
DATA=$(printf 'Hello World!'|base64) && gcloud functions call helloWorld --data '{"data":"'$DATA'"}'
gcloud functions logs read helloWorld
```