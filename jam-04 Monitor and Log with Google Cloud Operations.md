# 04 Monitor and Log with Google Cloud Operations Suite

## Monitoring Multiple Projects with Cloud Monitoring

## Monitoring and Logging for Cloud Functions
https://us-central1-qwiklabs-gcp-00-cdbae7396d43.cloudfunctions.net/helloWorld
echo "GET https://us-central1-qwiklabs-gcp-00-cdbae7396d43.cloudfunctions.net/helloWorld" | ./vegeta attack -duration=300s > results.bin

## Reporting Application Metrics into Cloud Monitoring
### Install Go and OpenCensus on your instance
sudo curl -O https://storage.googleapis.com/golang/go1.16.2.linux-amd64.tar.gz
sudo tar -xvf go1.16.2.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo mv go /usr/local
sudo apt-get update
sudo apt-get install git
export PATH=$PATH:/usr/local/go/bin
go get go.opencensus.io
go get contrib.go.opencensus.io/exporter/stackdriver
go mod init test3
go mod tidy

### Install agents on the VM:
curl -sSO https://dl.google.com/cloudagents/add-monitoring-agent-repo.sh
sudo bash add-monitoring-agent-repo.sh
sudo apt-get update
sudo apt-get install stackdriver-agent
curl -sSO https://dl.google.com/cloudagents/add-logging-agent-repo.sh
sudo bash add-logging-agent-repo.sh
sudo apt-get update
sudo apt-get install google-fluentd

### Create a basic application server in Go
```bash
vim main.go

package main

import (
        "fmt"
        "math/rand"
        "time"
)

func main() {
        // Here's our fake video processing application. Every second, it
        // checks the length of the input queue (e.g., number of videos
        // waiting to be processed) and records that information.
        for {
                time.Sleep(1 * time.Second)
                queueSize := getQueueSize()

                // Record the queue size.
                fmt.Println("Queue size: ", queueSize)
        }
}

func getQueueSize() (int64) {
        // Fake out a queue size here by returning a random number between
        // 1 and 100.
        return rand.Int63n(100) + 1
}
```
go run main.go

```bash
vim main.go

package main

import (
"context"
"fmt"
"log"
"math/rand"
"os"        // [[Add]]
"time"

"contrib.go.opencensus.io/exporter/stackdriver"   // [[Add]]
"go.opencensus.io/stats"
"go.opencensus.io/stats/view"

monitoredrespb "google.golang.org/genproto/googleapis/api/monitoredres" // [[Add]]
)

var videoServiceInputQueueSize = stats.Int64(
"my.videoservice.org/measure/input_queue_size",
"Number of videos queued up in the input queue",
stats.UnitDimensionless)

func main() {
// [[Add block]]
// Setup metrics exporting to Stackdriver.
exporter, err := stackdriver.NewExporter(stackdriver.Options{
ProjectID: os.Getenv("MY_PROJECT_ID"),
Resource: &monitoredrespb.MonitoredResource {
Type: "gce_instance",
Labels: map[string]string {
"instance_id": os.Getenv("MY_GCE_INSTANCE_ID"),
"zone": os.Getenv("MY_GCE_INSTANCE_ZONE"),
},
},
})
if err != nil {
log.Fatalf("Cannot setup Stackdriver exporter: %v", err)
}
view.RegisterExporter(exporter)
// [[End: add block]]

ctx := context.Background()

// Setup a view so that we can export our metric.
if err := view.Register(&view.View{
Name: "my.videoservice.org/measure/input_queue_size",
Description: "Number of videos queued up in the input queue",
Measure: videoServiceInputQueueSize,
Aggregation: view.LastValue(),
}); err != nil {
log.Fatalf("Cannot setup view: %v", err)
}
// Set the reporting period to be once per second.
view.SetReportingPeriod(1 * time.Second)

// Here’s our fake video processing application. Every second, it
// checks the length of the input queue (e.g., number of videos
// waiting to be processed) and records that information.
for {
time.Sleep(1 * time.Second)
queueSize := getQueueSize()

// Record the queue size.
stats.Record(ctx, videoServiceInputQueueSize.M(queueSize))
fmt.Println("Queue size: ", queueSize)
}
}

func getQueueSize() (int64) {
// Fake out a queue size here by returning a random number between
// 1 and 100.
return rand.Int63n(100) + 1
}
```
export MY_PROJECT_ID=qwiklabs-gcp-00-1d009b2132b5
export MY_GCE_INSTANCE_ID=my-opencensus-demo
export MY_GCE_INSTANCE_ZONE=us-central1-a
go mod tidy
go run main.go

## Creating and Alerting on Logs-based Metrics
export PROJECT_ID=qwiklabs-gcp-04-1e20b3a09744
git clone https://github.com/GoogleCloudPlatform/appengine-guestbook-python
cd appengine-guestbook-python/
gcloud app create --project $PROJECT_ID
gcloud app deploy --project $PROJECT_ID --version 1
gcloud datastore indexes create --project $PROJECT_ID index.yaml

## Autoscaling an Instance Group with Custom Cloud Monitoring Metrics
gsutil cp -r gs://spls/gsp087/* gs://qwiklabs-gcp-01-a3346c8050fe

## Monitor and Log with Google Cloud Operations Suite: Challenge Lab
Task 1: Configure Cloud Monitoring
textPayload=~"file_format\: ([4,8]K).*"
