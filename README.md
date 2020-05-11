# Zcash network in kubernetes

This project will compile and deploy zcash source code to sandbox environment with some specific code changes.

# STATUS
- [X] Deploy Tekton
- [X] Deploy Minio
- [X] Tekton task to sync block to Minio
- [X] Tekton tasks to create random secret
- [X] Tekton task to build and upload zcash binary to Minio
- [X] Make runner Docker image for deployments
- [ ] Deploy Tekton triggers ?? <-- not yet needed
- [ ] Create deployment for binary
- [ ] Tekton task to deploy built binary miners
- [ ] Tekton task to deploy built binary workers
- [ ] Deploy metrics tools
- [ ] Deploy dashboards
- [ ] Setup CI for runner Dockerfile

## Security

Access to the Kubernetes api allows the operator to make arbitrary changes in the cluster.

Jobs can be restricted in what resources they can access with Kubernetes Role Based Access policies.

No inbound connections are expected or supported.

## Requirements


### Kubernetes cluster


Using `kind`
```
kind create cluster --name zcash-in-a-box
kubectl cluster-info
```

Using GCP

```
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
```

## Deploy Tekon

Tekon is used to compile and deploy software inside the Kubernetes cluster.

```
kubectl apply -f kubernetes/tekton/release/tekton-pipelines-v0.12.0.yml
kubectl apply -f kubernetes/tekton/release/tekton-dashboard-v0.6.1.yml
kubectl apply -f kubernetes/tekton/serviceAccount.yml
```

### Open the dashboard

```
kubectl --namespace tekton-pipelines port-forward svc/tekton-dashboard 9097:9097
```

Navigate to http://localhost:9097

## Deploy Minio with Tekton

Minio will be used as object storage for blocks, binary files, and configurations.

```
kubectl create -f kubernetes/tekton/tasks/create-minio-secret.yml
kubectl create -f kubernetes/tekton/tasks/create-zcashrpc-secret.yml
```

```
kubectl apply -f kubernetes/minio/minio-standalone-pvc.yaml
kubectl apply -f kubernetes/minio/minio-standalone-service.yaml
kubectl apply -f kubernetes/minio/minio-standalone-deployment.yaml
```

** Minio healthcheck needs to be passing
```
kubectl create -f kubernetes/tekton/tasks/create-minio-bucket.yml
kubectl create -f kubernetes/tekton/tasks/create-cache-bucket.yml
```

## Import block snapshot

### Using a Tekton task
```
kubectl create -f kubernetes/tekton/tasks/import-block-snapshot-minio.yml 
```

### From local storage

Using Minio

- Download and install the minio client, `mc` from https://min.io/download#/
- Get the minio auth secret key, put it in an environmental variable.
```
export Z_SECRETACCESSKEY=$(kubectl  get secrets minio-secret-key -o jsonpath="{.data.SECRETACCESSKEY}" | base64 -d)
```
- Expose the minio port locally
```
kubectl port-forward svc/minio 9000:9000
```
- Upload the snapshot
```
mc config host add zcash-in-a-box http://localhost:9000 minio ${Z_SECRETACCESSKEY} --api S3v4
mc cp <BLOCK_SNAPSHOT_FILE_PATH> zcash-in-a-box/cache/
```

## Build zcashd


```
kubectl create -f kubernetes/tekton/tasks/build-binary.yml
```

## Deploy zcashd

## Deploy monitoring
