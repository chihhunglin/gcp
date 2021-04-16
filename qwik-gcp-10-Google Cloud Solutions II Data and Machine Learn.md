# Google Cloud Solutions II Data and Machine Learning

## Exploring NCAA Data with BigQuer
#standardSQL
SELECT
  event_type,
  COUNT(*) AS event_count
FROM `bigquery-public-data.ncaa_basketball.mbb_pbp_sr`
GROUP BY 1
ORDER BY event_count DESC;

#standardSQL
#most three points made
SELECT
  scheduled_date,
  name,
  market,
  alias,
  three_points_att,
  three_points_made,
  three_points_pct,
  opp_name,
  opp_market,
  opp_alias,
  opp_three_points_att,
  opp_three_points_made,
  opp_three_points_pct,
  (three_points_made + opp_three_points_made) AS total_threes
FROM `bigquery-public-data.ncaa_basketball.mbb_teams_games_sr`
WHERE season > 2010
ORDER BY total_threes DESC
LIMIT 5;

#standardSQL
SELECT
  venue_name, venue_capacity, venue_city, venue_state
FROM `bigquery-public-data.ncaa_basketball.mbb_teams_games_sr`
GROUP BY 1,2,3,4
ORDER BY venue_capacity DESC
LIMIT 5;

#standardSQL
#highest scoring game of all time
SELECT
  scheduled_date,
  name,
  market,
  alias,
  points_game AS team_points,
  opp_name,
  opp_market,
  opp_alias,
  opp_points_game AS opposing_team_points,
  points_game + opp_points_game AS point_total
FROM `bigquery-public-data.ncaa_basketball.mbb_teams_games_sr`
WHERE season > 2010
ORDER BY point_total DESC
LIMIT 5;

#standardSQL
#biggest point difference in a championship game
SELECT
  scheduled_date,
  name,
  market,
  alias,
  points_game AS team_points,
  opp_name,
  opp_market,
  opp_alias,
  opp_points_game AS opposing_team_points,
  ABS(points_game - opp_points_game) AS point_difference
FROM `bigquery-public-data.ncaa_basketball.mbb_teams_games_sr`
WHERE season > 2015 AND tournament_type = 'National Championship'
ORDER BY point_difference DESC
LIMIT 5;


##　TensorFlow for Poets
sudo apt-get update

sudo apt-get install -y python-pip python-dev python3-pip python3-dev virtualenv
pip install --upgrade pip
pip install virtualenv
virtualenv -p python3 venv
source venv/bin/activate
pip install --upgrade tensorflow==1.15.2

python
import tensorflow as tf
hello = tf.constant('Hello, TensorFlow!')
sess = tf.Session()
print(sess.run(hello))
exit()

git clone https://github.com/googlecodelabs/tensorflow-for-poets-2
cd tensorflow-for-poets-2
curl http://download.tensorflow.org/example_images/flower_photos.tgz \
    | tar xz
ls flower_photos
mv flower_photos tf_files

export IMAGE_SIZE=224
export ARCHITECTURE="mobilenet_0.50_${IMAGE_SIZE}"

## Creating an Object Detection Application Using TensorFlow
sudo -i
apt-get update
apt-get install -y protobuf-compiler python3-pil python3-lxml python3-pip python3-dev git
pip3 install Flask==1.1.1 WTForms==2.2.1 Flask_WTF==0.14.2 Werkzeug==0.16.0
pip3 install tensorflow==2.0.0b1
cd /opt
git clone https://github.com/tensorflow/models
cd models/research
protoc object_detection/protos/*.proto --python_out=.
mkdir -p /opt/graph_def
cd /tmp
for model in \
  ssd_mobilenet_v1_coco_11_06_2017 \
  ssd_inception_v2_coco_11_06_2017 \
  rfcn_resnet101_coco_11_06_2017 \
  faster_rcnn_resnet101_coco_11_06_2017 \
  faster_rcnn_inception_resnet_v2_atrous_coco_11_06_2017
do \
  curl -OL http://download.tensorflow.org/models/object_detection/$model.tar.gz
  tar -xzf $model.tar.gz $model/frozen_inference_graph.pb
  cp -a $model /opt/graph_def/
done
ln -sf /opt/graph_def/faster_rcnn_resnet101_coco_11_06_2017/frozen_inference_graph.pb /opt/graph_def/frozen_inference_graph.pb
cd $HOME
git clone https://github.com/GoogleCloudPlatform/tensorflow-object-detection-example
cp -a tensorflow-object-detection-example/object_detection_app_p3 /opt/
chmod u+x /opt/object_detection_app_p3/app.py
cp /opt/object_detection_app_p3/object-detection.service /etc/systemd/system/
systemctl daemon-reload

systemctl enable object-detection
systemctl start object-detection
systemctl status object-detection

## Using OpenTSDB to Monitor Time-Series Data on Cloud Platform
gcloud config set compute/zone us-central1-f
git clone https://github.com/GoogleCloudPlatform/opentsdb-bigtable.git
cd opentsdb-bigtable

gcloud container clusters create opentsdb-cluster \
--no-enable-autoupgrade \
--no-enable-ip-alias --no-enable-basic-auth \
--no-issue-client-certificate \
--metadata disable-legacy-endpoints=true \
--scopes "https://www.googleapis.com/auth/bigtable.admin","https://www.googleapis.com/auth/bigtable.data"

kubectl create -f configmaps/opentsdb-config.yaml
kubectl create -f jobs/opentsdb-init.yaml
kubectl describe jobs
pods=$(kubectl get pods --selector=job-name=opentsdb-init \
--output=jsonpath={.items..metadata.name})
kubectl logs $pods
kubectl create -f deployments/opentsdb-write.yaml
kubectl get pods
kubectl create -f deployments/opentsdb-read.yaml
kubectl get pods
kubectl create -f services/opentsdb-write.yaml
kubectl get services
kubectl create -f services/opentsdb-read.yaml
kubectl get services
vim rbac.yaml

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: heapster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:heapster
subjects:
- kind: ServiceAccount
  name: heapster-opentsdb
  namespace: default
```

kubectl apply -f rbac.yaml
kubectl create -f deployments/heapster.yaml
kubectl get deployments

kubectl create -f configmaps/grafana.yaml
kubectl get configmaps
kubectl create -f deployments/grafana.yaml
kubectl get deployments
grafana=$(kubectl get pods --selector=app=grafana \
  --output=jsonpath={.items..metadata.name})
kubectl port-forward $grafana 8080:3000

## Scanning User-generated Content Using the Cloud Video Intelligence and Cloud Vision APIs
https://cdn.qwiklabs.com/F%2Bjs06t3HuOCj44qnIRbLW6rnMlAdB%2Fchedbbze6W4U%3D

export PROJECT_ID=$(gcloud info --format='value(config.project)')
export IV_BUCKET_NAME=${PROJECT_ID}-upload
export FILTERED_BUCKET_NAME=${PROJECT_ID}-filtered
export FLAGGED_BUCKET_NAME=${PROJECT_ID}-flagged
export STAGING_BUCKET_NAME=${PROJECT_ID}-staging

gsutil mb gs://${IV_BUCKET_NAME}
gsutil mb gs://${FILTERED_BUCKET_NAME}
gsutil mb gs://${FLAGGED_BUCKET_NAME}
gsutil mb gs://${STAGING_BUCKET_NAME}
gsutil ls

export UPLOAD_NOTIFICATION_TOPIC=upload_notification
gcloud pubsub topics create ${UPLOAD_NOTIFICATION_TOPIC}
gcloud pubsub topics create visionapiservice
gcloud pubsub topics create videointelligenceservice
gcloud pubsub topics create bqinsert
gcloud pubsub topics list

gsutil notification create -t upload_notification -f json -e OBJECT_FINALIZE gs://${IV_BUCKET_NAME}
gsutil notification list gs://${IV_BUCKET_NAME}

gsutil -m cp -r gs://spls/gsp138/cloud-functions-intelligentcontent-nodejs .
cd cloud-functions-intelligentcontent-nodejs

export DATASET_ID=intelligentcontentfilter
export TABLE_NAME=filtered_content

bq --project_id ${PROJECT_ID} mk ${DATASET_ID}
bq --project_id ${PROJECT_ID} mk --schema intelligent_content_bq_schema.json -t ${DATASET_ID}.${TABLE_NAME}

sed -i "s/\[PROJECT-ID\]/$PROJECT_ID/g" config.json
sed -i "s/\[FLAGGED_BUCKET_NAME\]/$FLAGGED_BUCKET_NAME/g" config.json
sed -i "s/\[FILTERED_BUCKET_NAME\]/$FILTERED_BUCKET_NAME/g" config.json
sed -i "s/\[DATASET_ID\]/$DATASET_ID/g" config.json
sed -i "s/\[TABLE_NAME\]/$TABLE_NAME/g" config.json

gcloud functions deploy GCStoPubsub --runtime nodejs10 --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic ${UPLOAD_NOTIFICATION_TOPIC} --entry-point GCStoPubsub
gcloud functions deploy visionAPI --runtime nodejs10 --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic visionapiservice --entry-point visionAPI
gcloud functions deploy videoIntelligenceAPI --runtime nodejs10 --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic videointelligenceservice --entry-point videoIntelligenceAPI --timeout 540
gcloud functions deploy insertIntoBigQuery --runtime nodejs10 --stage-bucket gs://${STAGING_BUCKET_NAME} --trigger-topic bqinsert --entry-point insertIntoBigQuery
gcloud beta functions list

gcloud beta functions logs read --filter "finished with status" "GCStoPubsub" --limit 100
gcloud beta functions logs read --filter "finished with status" "insertIntoBigQuery" --limit 100

echo "
#standardSql

SELECT insertTimestamp,
  contentUrl,
  flattenedSafeSearch.flaggedType,
  flattenedSafeSearch.likelihood
FROM \`$PROJECT_ID.$DATASET_ID.$TABLE_NAME\`
CROSS JOIN UNNEST(safeSearch) AS flattenedSafeSearch
ORDER BY insertTimestamp DESC,
  contentUrl,
  flattenedSafeSearch.flaggedType
LIMIT 1000
" > sql.txt

bq --project_id ${PROJECT_ID} query < sql.txt
