# Using the Cloud SDK Command Line

## Configuring Networks via gcloud

gcloud compute networks create labnet --subnet-mode=custom
gcloud compute networks subnets create labnet-sub \
   --network labnet \
   --region us-central1 \
   --range 10.0.0.0/28
gcloud compute networks list
gcloud compute networks describe NETWORK_NAME
gcloud compute networks subnets list
gcloud compute firewall-rules create labnet-allow-internal \
	--network=labnet \
	--action=ALLOW \
	--rules=icmp,tcp:22 \
	--source-ranges=0.0.0.0/0
gcloud compute firewall-rules describe [FIREWALL_RULE_NAME]
gcloud compute networks create privatenet --subnet-mode=custom
gcloud compute networks subnets create private-sub \
    --network=privatenet \
    --region=us-central1 \
    --range 10.1.0.0/28
gcloud compute firewall-rules create privatenet-deny \
    --network=privatenet \
    --action=DENY \
    --rules=icmp,tcp:22 \
    --source-ranges=0.0.0.0/0
gcloud compute firewall-rules list --sort-by=NETWORK
gcloud compute instances create pnet-vm \
--zone=us-central1-c \
--machine-type=n1-standard-1 \
--subnet=private-sub
gcloud compute instances create lnet-vm \
--zone=us-central1-c \
--machine-type=n1-standard-1 \
--subnet=labnet-sub
gcloud compute instances list --sort-by=ZONE


## Configuring IAM Permissions with gcloud
https://cdn.qwiklabs.com/SUcA5Cwo3NVw0UXmnxQ01ujmrWeTYID2C4EpNp8GmcI%3D

gcloud
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init

gcloud components list
gcloud components install beta

gcloud compute instances create lab-1
gcloud config list
gcloud compute zones list
The default configration is stored in ~/.config/gcloud/configurations/config_default.

gcloud config set compute/zone us-central1-b
cat ~/.config/gcloud/configurations/config_default

gcloud config configurations activate default

gcloud iam roles list | grep "name:"
gcloud iam roles describe roles/compute.instanceAdmin

gcloud config configurations activate user2
echo "export PROJECTID2=qwiklabs-gcp-03-e99e8edb3fd5" >> ~/.bashrc
. ~/.bashrc
gcloud config set project $PROJECTID2
gcloud config configurations activate default

sudo yum -y install epel-release
sudo yum -y install jq
echo "export USERID2=student-03-eff0b2821475@qwiklabs.net" >> ~/.bashrc
. ~/.bashrc
gcloud projects add-iam-policy-binding $PROJECTID2 --member user:$USERID2 --role=roles/viewer

gcloud config configurations activate user2
gcloud config set project $PROJECTID2
gcloud compute instances list
gcloud config configurations activate default
gcloud iam roles create devops --project $PROJECTID2 --permissions "compute.instances.create,compute.instances.delete,compute.instances.start,compute.instances.stop,compute.instances.update,compute.disks.create,compute.subnetworks.use,compute.subnetworks.useExternalIp,compute.instances.setMetadata,compute.instances.setServiceAccount"
gcloud projects add-iam-policy-binding $PROJECTID2 --member user:$USERID2 --role=roles/iam.serviceAccountUser
gcloud projects add-iam-policy-binding $PROJECTID2 --member user:$USERID2 --role=projects/$PROJECTID2/roles/devops
gcloud config configurations activate user2
gcloud compute instances create lab-2

gcloud config configurations activate default
gcloud config set project $PROJECTID2
gcloud iam service-accounts create devops --display-name devops
gcloud iam service-accounts list  --filter "displayName=devops"
SA=$(gcloud iam service-accounts list --format="value(email)" --filter "displayName=devops")
gcloud projects add-iam-policy-binding $PROJECTID2 --member serviceAccount:$SA --role=roles/iam.serviceAccountUser
gcloud projects add-iam-policy-binding $PROJECTID2 --member serviceAccount:$SA --role=roles/compute.instanceAdmin

gcloud compute instances create lab-3 --service-account $SA --scopes "https://www.googleapis.com/auth/compute"
gcloud compute ssh lab-3

## Using gsutil to Perform Operations on Buckets and Objects
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
cd training-data-analyst/blogs
PROJECT_ID=`gcloud config get-value project`
BUCKET=${PROJECT_ID}-bucket
gsutil mb -c multi_regional gs://${BUCKET}
gsutil -m cp -r endpointslambda gs://${BUCKET}

gsutil ls gs://${BUCKET}/*
mv endpointslambda/Apache2_0License.txt endpointslambda/old.txt
rm endpointslambda/aeflex-endpoints/app.yaml
gsutil -m rsync -d -r endpointslambda gs://${BUCKET}/endpointslambda
gsutil ls gs://${BUCKET}/*
gsutil -m acl set -R -a public-read gs://${BUCKET}

gsutil cp -s nearline ghcn/ghcn_on_bq.ipynb gs://${BUCKET}
gsutil ls -Lr gs://${BUCKET} | more
gsutil rm -rf gs://${BUCKET}/*
gsutil rb gs://${BUCKET}

## BigQuery: Qwik Start - Command Line
bq show bigquery-public-data:samples.shakespeare
bq help query
bq query --use_legacy_sql=false \
'SELECT
   word,
   SUM(word_count) AS count
 FROM
   `bigquery-public-data`.samples.shakespeare
 WHERE
   word LIKE "%raisin%"
 GROUP BY
   word'

bq query --use_legacy_sql=false \
'SELECT
   word
 FROM
   `bigquery-public-data`.samples.shakespeare
 WHERE
   word = "huzzah"'

bq ls
bq ls bigquery-public-data:
bq mk babynames
bq ls
wget http://www.ssa.gov/OACT/babynames/names.zip
unzip names.zip
bq load babynames.names2010 yob2010.txt name:string,gender:string,count:integer
bq ls babynames
bq show babynames.names2010
bq query "SELECT name,count FROM babynames.names2010 WHERE gender = 'F' ORDER BY count DESC LIMIT 5"
bq query "SELECT name,count FROM babynames.names2010 WHERE gender = 'M' ORDER BY count ASC LIMIT 5"
bq rm -r babynames
