# Perform Foundational Data ML and AI Tasks in Google Cloud

## AI Platform: Qwik Start
git clone https://github.com/GoogleCloudPlatform/training-data-analyst

## Dataprep: Qwik Start

## Dataflow: Qwik Start - Python
python3 --version
pip3 --version
sudo pip3 install -U pip
sudo pip3 install --upgrade virtualenv
virtualenv -p python3.7 env
source env/bin/activate
pip install apache-beam[gcp]
python -m apache_beam.examples.wordcount --output OUTPUT_FILE
ls
cat <file name>
BUCKET=gs://qwiklabs-gcp-04-37cffca0c055
python -m apache_beam.examples.wordcount --project $DEVSHELL_PROJECT_ID \
  --runner DataflowRunner \
  --staging_location $BUCKET/staging \
  --temp_location $BUCKET/temp \
  --output $BUCKET/results/output \
  --region us-central1

## Dataproc: Qwik Start - Command Line
gcloud config set dataproc/region us-central1
gcloud dataproc clusters create example-cluster --worker-boot-disk-size 500
gcloud dataproc jobs submit spark --cluster example-cluster \
  --class org.apache.spark.examples.SparkPi \
  --jars file:///usr/lib/spark/examples/jars/spark-examples.jar -- 1000
gcloud dataproc clusters update example-cluster --num-workers 4

## Cloud Natural Language API: Qwik Start
export GOOGLE_CLOUD_PROJECT=$(gcloud config get-value core/project)
gcloud iam service-accounts create my-natlang-sa \
  --display-name "my natural language service account"
gcloud iam service-accounts keys create ~/key.json \
  --iam-account my-natlang-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com
export GOOGLE_APPLICATION_CREDENTIALS="/home/USER/key.json"

### VM
gcloud ml language analyze-entities --content="Michelangelo Caravaggio, Italian painter, is known for 'The Calling of Saint Matthew'." > result.json

## Google Cloud Speech API: Qwik Start
export API_KEY=<YOUR_API_KEY>

touch request.json

curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}"
curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > result.json

## Video Intelligence: Qwik Start
gcloud iam service-accounts create quickstart
gcloud iam service-accounts keys create key.json --iam-account quickstart@qwiklabs-gcp-00-de90eaaea9fa.iam.gserviceaccount.com
gcloud auth activate-service-account --key-file key.json
gcloud auth print-access-token

vim 
```bash
{
   "inputUri":"gs://spls/gsp154/video/train.mp4",
   "features": [
       "LABEL_DETECTION"
   ]
}
```

curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/videos:annotate' \
    -d @request.json

curl -s -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer '$(gcloud auth print-access-token)'' \
    'https://videointelligence.googleapis.com/v1/projects/643148839883/locations/asia-east1/operations/4338705407339675289'

## Perform Foundational Data, ML, and AI Tasks in Google Cloud: Challenge Lab
Task 1: Run a simple Dataflow job

Task 2: Run a simple Dataproc job
Task 3: Run a simple Dataprep job

=== 以下有點亂.....
Task 4
gcloud iam service-accounts create my-natlang-sa \
	--display-name "my natural language service account"
gcloud iam service-accounts keys create ~/key.json \
	--iam-account my-natlang-sa@qwiklabs-gcp-04-0de4ffededed.iam.gserviceaccount.com
export GOOGLE_APPLICATION_CREDENTIALS="/home/USER/key.json"
gcloud ml language analyze-entities --content="Old Norse texts portray Odin as one-eyed and long-bearded, frequently wielding a spear named Gungnir and wearing a cloak and a broad hat." > result.json
gsutil cp result.json gs://qwiklabs-gcp-04-0de4ffededed-marking/task4-cnl.result

export API_KEY=AIzaSyBzjMHAGnzUPxgMIch1lsv3iMPgTMYNhxQ
curl -s -X POST -H "Content-Type: application/json" --data-binary @gsc-request.json \
"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > task4-gcs.result

gsutil cp task4-gcs.result gs://qwiklabs-gcp-04-0de4ffededed-marking/task4-gcs.result

Video Intelligence:
gcloud iam service-accounts create quickstart
gcloud iam service-accounts keys create key.json --iam-account quickstart@qwiklabs-gcp-04-0de4ffededed.iam.gserviceaccount.com
gcloud auth activate-service-account --key-file key.json
export ACCESS_TOKEN=$(gcloud auth print-access-token)
curl -s -H 'Content-Type: application/json' \
	-H "Authorization: Bearer $ACCESS_TOKEN" \
	'https://videointelligence.googleapis.com/v1/videos:annotate' \
	-d @gvi-request.json
curl -s -H 'Content-Type: application/json' -H "Authorization: Bearer $ACCESS_TOKEN" 'https://videointelligence.googleapis.com/v1/operations/OPERATION_FROM_PREVIOUS_REQUEST' > result1.json

Task 4: 



Part 1:



Cloud Natural Language:



In cloud shell



gcloud iam service-accounts create my-natlang-sa \

  --display-name "my natural language service account"



gcloud iam service-accounts keys create ~/key.json \

  --iam-account my-natlang-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com



export GOOGLE_APPLICATION_CREDENTIALS="/home/USER/key.json"



gcloud ml language analyze-entities --content="Old Norse texts portray Odin as one-eyed and long-bearded, frequently wielding a spear named Gungnir and wearing a cloak and a broad hat." > result.json



gsutil cp result.json gs://YOUR_PROJECT-marking/task4-cnl.result




Cloud Speech:



Create an API key and set API_KEY var in cloud shell



Create the file request.json with:

{

  "config": {

      "encoding":"FLAC",

      "languageCode": "en-US"

  },

  "audio": {

      "uri":"gs://cloud-training/gsp323/task4.flac"

  }

}



curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \

"https://speech.googleapis.com/v1/speech:recognize?key=${API_KEY}" > result.json



gsutil cp result.json gs://YOUR_PROJECT-marking/task4-gcs.result




Video Intelligence:



gcloud iam service-accounts create quickstart



gcloud iam service-accounts keys create key.json --iam-account quickstart@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com

Marking Guide



gcloud auth activate-service-account --key-file key.json



export ACCESS_TOKEN=$(gcloud auth print-access-token)



Create a file called request.json with 

{

   "inputUri":"gs://spls/gsp154/video/chicago.mp4",

   "features": [

       "TEXT_DETECTION"

   ]

}



curl -s -H 'Content-Type: application/json' \

    -H "Authorization: Bearer $ACCESS_TOKEN" \

    'https://videointelligence.googleapis.com/v1/videos:annotate' \

    -d @request.json



Wait for the operation to complete, check with 

curl -s -H 'Content-Type: application/json' -H "Authorization: Bearer $ACCESS_TOKEN" 'https://videointelligence.googleapis.com/v1/operations/OPERATION_FROM_PREVIOUS_REQUEST' > result1.json




gsutil cp result1.json gs://YOUR_PROJECT-marking/task4-gvi.result




Task 4 steps: (Alternative)



gcloud iam service-accounts create my-natlang-sa \

  --display-name "my natural language service account"



gcloud iam service-accounts keys create ~/key.json \

  --iam-account my-natlang-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com



export GOOGLE_APPLICATION_CREDENTIALS="/home/$USER/key.json"



gcloud auth activate-service-account my-natlang-sa@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com --key-file=$GOOGLE_APPLICATION_CREDENTIALS



gcloud ml language analyze-entities --content="Old Norse texts portray Odin as one-eyed and long-bearded, frequently wielding a spear named Gungnir and wearing a cloak and a broad hat." > result.json



gcloud auth login

(Copy the token from the link provided)



gsutil cp result.json gs://$GOOGLE_CLOUD_PROJECT-marking/task4-cnl.result