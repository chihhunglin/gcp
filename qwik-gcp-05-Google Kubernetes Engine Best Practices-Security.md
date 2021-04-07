# Google Kubernetes Engine Best Practices - Security

## Migrating to GKE Containers

sudo apt-get install apache2-utils
ab -V
git clone https://github.com/GoogleCloudPlatform/gke-migration-to-containers.git
cd gke-migration-to-containers
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
make create
gcloud compute ssh vm-webserver --zone us-central1-a
gcloud compute ssh cos-vm --zone us-central1-a
git clone https://github.com/GoogleCloudPlatform/gke-migration-to-containers.git
cd gke-migration-to-containers/container
ps aux | grep 8080
sudo kill -9 <YOUR_PORT_NUMBER>
sudo docker run --rm -d --name=appuser -p 8080:8080 gcr.io/migration-to-containers/prime-flask:1.0.2
ps aux
ls /usr/local/bin/python
sudo docker ps
sudo docker exec -it $(sudo docker ps |awk '/prime-flask/ {print $1}') ps aux
exit

gcloud container clusters get-credentials prime-server-cluster
kubectl get pods
kubectl exec $(kubectl get pods -lapp=prime-server -ojsonpath='{.items[].metadata.name}')  -- ps aux

make validate

ab -c 120 -t 60  http://34.117.166.201/prime/10000

kubectl scale --replicas 3 deployment/prime-server
make teardown

## How to Use a Network Policy on Google Kubernetes Engine
git clone https://github.com/GoogleCloudPlatform/gke-network-policy-demo.git
cd gke-network-policy-demo
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

make setup-project
cat terraform/terraform.tfvars
sed -i 's/~> 2.17.0/~> 2.17.0/g' terraform/provider.tf
make tf-apply

gcloud container clusters describe gke-demo-cluster | grep  -A2 networkPolicy
gcloud compute ssh gke-demo-bastion
kubectl apply -f ./manifests/hello-app/
kubectl get pods
kubectl logs --tail 10 -f $(kubectl get pods -oname -l app=hello)
kubectl logs --tail 10 -f $(kubectl get pods -oname -l app=not-hello)
kubectl apply -f ./manifests/network-policy.yaml
kubectl logs --tail 10 -f $(kubectl get pods -oname -l app=not-hello)
kubectl delete -f ./manifests/network-policy.yaml
kubectl create -f ./manifests/network-policy-namespaced.yaml
kubectl logs --tail 10 -f $(kubectl get pods -oname -l app=hello)
kubectl -n hello-apps apply -f ./manifests/hello-app/hello-client.yaml

kubectl logs --tail 10 -f -n hello-apps $(kubectl get pods -oname -l app=hello -n hello-apps)
exit
make teardown


## Using Role-based Access Control in Kubernetes Engine
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
git clone https://github.com/GoogleCloudPlatform/gke-rbac-demo.git
cd gke-rbac-demo
sed -i 's/latest_master_version/default_cluster_version/g' terraform/main.tf
make create
gcloud iam service-accounts list
gcloud compute instances list
gcloud compute ssh gke-tutorial-admin
kubectl apply -f ./manifests/rbac.yaml

gcloud compute ssh gke-tutorial-owner
kubectl create -n dev -f ./manifests/hello-server.yaml

gcloud compute ssh gke-tutorial-auditor
kubectl get pods -l app=hello-server --all-namespaces
kubectl get pods -l app=hello-server --namespace=dev
kubectl get pods -l app=hello-server --namespace=test
kubectl get pods -l app=hello-server --namespace=prod
kubectl create -n dev -f manifests/hello-server.yaml
kubectl delete deployment -n dev -l app=hello-server

kubectl apply -f manifests/pod-labeler.yaml
kubectl get pods -l app=pod-labeler
kubectl describe pod -l app=pod-labeler | tail -n 20
kubectl logs -l app=pod-labeler
kubectl get pod -oyaml -l app=pod-labeler
kubectl apply -f manifests/pod-labeler-fix-1.yaml
kubectl get deployment pod-labeler -oyaml
kubectl get pods -l app=pod-labeler
kubectl logs -l app=pod-labeler
kubectl get rolebinding pod-labeler -oyaml
kubectl get role pod-labeler -oyaml
kubectl apply -f manifests/pod-labeler-fix-2.yaml
kubectl get role pod-labeler -oyaml
kubectl delete pod -l app=pod-labeler
kubectl get pods --show-labels
kubectl logs -l app=pod-labeler

make teardown

## Google Kubernetes Engine Security: Binary Authorization
gsutil cp -r gs://spls/gke-binary-auth/* .
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
chmod +x create.sh
chmod 777 validate.sh
sed -i 's/validMasterVersions\[0\]/defaultClusterVersion/g' ./create.sh
./create.sh -c my-cluster-1
./validate.sh -c my-cluster-1

docker pull gcr.io/google-containers/nginx:latest
gcloud auth configure-docker
PROJECT_ID="$(gcloud config get-value project)"
docker tag gcr.io/google-containers/nginx "gcr.io/${PROJECT_ID}/nginx:latest"
docker push "gcr.io/${PROJECT_ID}/nginx:latest"
gcloud container images list-tags "gcr.io/${PROJECT_ID}/nginx"

cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: "gcr.io/${PROJECT_ID}/nginx:latest"
    ports:
    - containerPort: 80
EOF

kubectl get pods
kubectl delete pod nginx

cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: "gcr.io/${PROJECT_ID}/nginx:latest"
    ports:
    - containerPort: 80
EOF

Query build:
resource.type="k8s_cluster" protoPayload.response.reason="Forbidden"

echo "gcr.io/${PROJECT_ID}/nginx*"

cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: "gcr.io/${PROJECT_ID}/nginx:latest"
    ports:
    - containerPort: 80
EOF
kubectl delete pod nginx


ATTESTOR="manually-verified" # No spaces allowed
ATTESTOR_NAME="Manual Attestor"
ATTESTOR_EMAIL="$(gcloud config get-value core/account)" # This uses your current user/email
NOTE_ID="Human-Attestor-Note" # No spaces
NOTE_DESC="Human Attestation Note Demo"
NOTE_PAYLOAD_PATH="note_payload.json"
IAM_REQUEST_JSON="iam_request.json"
cat > ${NOTE_PAYLOAD_PATH} << EOF
{
  "name": "projects/${PROJECT_ID}/notes/${NOTE_ID}",
  "attestation_authority": {
    "hint": {
      "human_readable_name": "${NOTE_DESC}"
    }
  }
}
EOF

curl -X POST \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
    --data-binary @${NOTE_PAYLOAD_PATH}  \
    "https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/?noteId=${NOTE_ID}"

curl -H "Authorization: Bearer $(gcloud auth print-access-token)"  \
    "https://containeranalysis.googleapis.com/v1beta1/projects/${PROJECT_ID}/notes/${NOTE_ID}"

PGP_PUB_KEY="generated-key.pgp"
sudo apt-get install rng-tools
sudo rngd -r /dev/urandom
gpg --quick-generate-key --yes ${ATTESTOR_EMAIL}
gpg --armor --export "${ATTESTOR_EMAIL}" > ${PGP_PUB_KEY}
gcloud --project="${PROJECT_ID}" \
    beta container binauthz attestors create "${ATTESTOR}" \
    --attestation-authority-note="${NOTE_ID}" \
    --attestation-authority-note-project="${PROJECT_ID}"
gcloud --project="${PROJECT_ID}" \
    beta container binauthz attestors public-keys add \
    --attestor="${ATTESTOR}" \
    --pgp-public-key-file="${PGP_PUB_KEY}"
gcloud --project="${PROJECT_ID}" \
    beta container binauthz attestors list
GENERATED_PAYLOAD="generated_payload.json"
GENERATED_SIGNATURE="generated_signature.pgp"
PGP_FINGERPRINT="$(gpg --list-keys ${ATTESTOR_EMAIL} | head -2 | tail -1 | awk '{print $1}')"
IMAGE_PATH="gcr.io/${PROJECT_ID}/nginx"
IMAGE_DIGEST="$(gcloud container images list-tags --format='get(digest)' $IMAGE_PATH | head -1)"
gcloud beta container binauthz create-signature-payload \
    --artifact-url="${IMAGE_PATH}@${IMAGE_DIGEST}" > ${GENERATED_PAYLOAD}
cat "${GENERATED_PAYLOAD}"
gpg --local-user "${ATTESTOR_EMAIL}" \
    --armor \
    --output ${GENERATED_SIGNATURE} \
    --sign ${GENERATED_PAYLOAD}
cat "${GENERATED_SIGNATURE}"
gcloud beta container binauthz attestations create \
    --artifact-url="${IMAGE_PATH}@${IMAGE_DIGEST}" \
    --attestor="projects/${PROJECT_ID}/attestors/${ATTESTOR}" \
    --signature-file=${GENERATED_SIGNATURE} \
    --public-key-id="${PGP_FINGERPRINT}"
gcloud beta container binauthz attestations list \
    --attestor="projects/${PROJECT_ID}/attestors/${ATTESTOR}"
echo "projects/${PROJECT_ID}/attestors/${ATTESTOR}" # Copy this output to your copy/paste buffer

IMAGE_PATH="gcr.io/${PROJECT_ID}/nginx"
IMAGE_DIGEST="$(gcloud container images list-tags --format='get(digest)' $IMAGE_PATH | head -1)"
cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: "${IMAGE_PATH}@${IMAGE_DIGEST}"
    ports:
    - containerPort: 80
EOF


cat << EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-alpha
  annotations:
    alpha.image-policy.k8s.io/break-glass: "true"
spec:
  containers:
  - name: nginx
    image: "nginx:latest"
    ports:
    - containerPort: 80
EOF

./delete.sh -c my-cluster-1

## Securing Applications on Kubernetes Engine - Three Examples
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
git clone https://github.com/GoogleCloudPlatform/gke-security-scenarios-demo
cd gke-security-scenarios-demo
sed -i 's/f1-micro/n1-standard-1/g' ~/gke-security-scenarios-demo/terraform/variables.tf
make setup-project
cat terraform/terraform.tfvars
make create
gcloud compute ssh gke-tutorial-bastion
kubectl get pods --all-namespaces
kubectl apply -f manifests/nginx.yaml
kubectl get pods
kubectl describe pod -l app=nginx
kubectl apply -f manifests/apparmor-loader.yaml
kubectl delete pods -l app=nginx
kubectl get pods
kubectl get services
kubectl get pods --show-labels
kubectl apply -f manifests/pod-labeler.yaml
kubectl get pods --show-labels
exit
make teardown

## Hardening Default GKE Cluster Configurations
export MY_ZONE=us-central1-a
gcloud container clusters create simplecluster --zone $MY_ZONE --num-nodes 2 --metadata=disable-legacy-endpoints=false
kubectl version
kubectl run -it --rm gcloud --image=google/cloud-sdk:latest --restart=Never -- bash
curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/name
curl -s http://metadata.google.internal/computeMetadata/v1/instance/name
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name
curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/attributes/
curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/attributes/kube-env
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/scopes
exit

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpath
spec:
  containers:
  - name: hostpath
    image: google/cloud-sdk:latest
    command: ["/bin/bash"]
    args: ["-c", "tail -f /dev/null"]
    volumeMounts:
    - mountPath: /rootfs
      name: rootfs
  volumes:
  - name: rootfs
    hostPath:
      path: /
EOF

kubectl get pod

kubectl exec -it hostpath -- bash
chroot /rootfs /bin/bash
exit
exit
kubectl delete pod hostpath
gcloud beta container node-pools create second-pool --cluster=simplecluster --zone=$MY_ZONE --num-nodes=1 --metadata=disable-legacy-endpoints=true --workload-metadata-from-node=SECURE

kubectl run -it --rm gcloud --image=google/cloud-sdk:latest --restart=Never --overrides='{ "apiVersion": "v1", "spec": { "securityContext": { "runAsUser": 65534, "fsGroup": 65534 }, "nodeSelector": { "cloud.google.com/gke-nodepool": "second-pool" } } }' -- bash
curl -s http://metadata.google.internal/computeMetadata/v1beta1/instance/name
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/attributes/kube-env
curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/instance/name
exit
kubectl create clusterrolebinding clusteradmin --clusterrole=cluster-admin --user="$(gcloud config list account --format 'value(core.account)')"

cat <<EOF | kubectl apply -f -
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restrictive-psp
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'docker/default'
    apparmor.security.beta.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName:  'docker/default'
    apparmor.security.beta.kubernetes.io/defaultProfileName:  'runtime/default'
spec:
  privileged: false
  # Required to prevent escalations to root.
  allowPrivilegeEscalation: false
  # This is redundant with non-root + disallow privilege escalation,
  # but we can provide it for defense in depth.
  requiredDropCapabilities:
    - ALL
  # Allow core volume types.
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    # Assume that persistentVolumes set up by the cluster admin are safe to use.
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    # Require the container to run without root privileges.
    rule: 'MustRunAsNonRoot'
  seLinux:
    # This policy assumes the nodes are using AppArmor rather than SELinux.
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      # Forbid adding the root group.
      - min: 1
        max: 65535
EOF

cat <<EOF | kubectl apply -f -
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: restrictive-psp
rules:
- apiGroups:
  - extensions
  resources:
  - podsecuritypolicies
  resourceNames:
  - restrictive-psp
  verbs:
  - use
EOF

cat <<EOF | kubectl apply -f -
---
# All service accounts in kube-system
# can 'use' the 'permissive-psp' PSP
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: restrictive-psp
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: restrictive-psp
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
EOF

gcloud beta container clusters update simplecluster --zone $MY_ZONE --enable-pod-security-policy
gcloud iam service-accounts create demo-developer
MYPROJECT=$(gcloud config list --format 'value(core.project)')
gcloud projects add-iam-policy-binding "${MYPROJECT}" --role=roles/container.developer --member="serviceAccount:demo-developer@${MYPROJECT}.iam.gserviceaccount.com"
gcloud iam service-accounts keys create key.json --iam-account "demo-developer@${MYPROJECT}.iam.gserviceaccount.com"
gcloud auth activate-service-account --key-file=key.json
gcloud container clusters get-credentials simplecluster --zone $MY_ZONE

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpath
spec:
  containers:
  - name: hostpath
    image: google/cloud-sdk:latest
    command: ["/bin/bash"]
    args: ["-c", "tail -f /dev/null"]
    volumeMounts:
    - mountPath: /rootfs
      name: rootfs
  volumes:
  - name: rootfs
    hostPath:
      path: /
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpath
spec:
  securityContext:
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: hostpath
    image: google/cloud-sdk:latest
    command: ["/bin/bash"]
    args: ["-c", "tail -f /dev/null"]
EOF

kubectl get pod hostpath -o=jsonpath="{ .metadata.annotations.kubernetes\.io/psp }"
