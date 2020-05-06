# Zcash network in kubernetes

This project will compile and deploy zcash source code to sandbox environment with some specific code changes.

# STATUS
- [X] Deploy Tekton
- [X] Deploy Minio
- [X] Tekton task to sync block to Minio
- [X] Tekton tasks to create random secret
- [X] Tekton task to build and upload zcash binary to Minio
- [ ] Create deployment for binary
- [ ] Tekton task to deploy built binary miners
- [ ] Tekton task to deploy built binary workers
- [ ] Deploy metrics tools
- [ ] Deploy dashboards


## Security

Access to the Kubernetes api allows the operator to make arbitrary changes in the cluster.

Jobs can be restricted in what resources they can access with Kubernetes Role Based Access policies.

No inbound connections are expected or supported.

## Requirements

### Zcash source git repository

https://github.com/zcash/zcash.git

### Kubernetes cluster

export CLUSTERNAME=zcash-in-a-box-ben-v1
export CLUSTERZONE=us-west1-a
export GCP_PROJECT=buildenv

gcloud container \
clusters create ${CLUSTERNAME} \
--project ${GCP_PROJECT} \
--zone ${CLUSTERZONE} \
--machine-type "n1-standard-8" \
--cluster-version="1.15" \
--num-nodes "1" \
--preemptible \
--enable-autoupgrade \
--enable-autorepair \
--enable-ip-alias \
--metadata disable-legacy-endpoints=true \
--enable-autoscaling \
--max-nodes=2


gcloud container clusters get-credentials \
  --project ${GCP_PROJECT} \
  --zone ${CLUSTERZONE} \
  ${CLUSTERNAME}

kubectl create clusterrolebinding cluster-admin-binding \
--clusterrole=cluster-admin \
--user=$(gcloud config get-value core/account)

## Deploy Tekon

Tekon is used to compile and deploy software inside the Kubernetes cluster.


kubectl apply -f kubernetes/tekton/release/tekton-pipelines-v0.12.0.yml
kubectl apply -f kubernetes/tekton/release/tekton-dashboard-v0.6.1.yml
kubectl apply -f kubernetes/tekton/serviceAccount.yml

### Open the dashboard

kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097

Navigate to http://localhost:9097

## Deploy Minio with Tekton

Minio will be used as object storage for blocks, binary files, and configurations.

kubectl create -f kubernetes/tekton/tasks/create-minio-secret.yml
kubectl create -f kubernetes/tekton/tasks/create-zcashrpc-secret.yml

kubectl apply -f kubernetes/minio/minio-standalone-pvc.yaml
kubectl apply -f kubernetes/minio/minio-standalone-service.yaml
kubectl apply -f kubernetes/minio/minio-standalone-deployment.yaml

** Minio healthcheck needs to be passing
kubectl create -f kubernetes/tekton/tasks/create-minio-bucket.yml
kubectl create -f kubernetes/tekton/tasks/create-cache-bucket.yml
kubectl create -f kubernetes/tekton/tasks/import-block-snapshot-minio.yml 

kubectl apply -f kubernetes/template/zcash-inabox-configmap.yml
kubectl create -f kubernetes/tekton/tasks/build-binary.yml

