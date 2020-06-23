# Zcash network in kubernetes - TZE


### Kubernetes cluster


Using `kind`
```
kind create cluster --name zcash-in-a-box-tze
kubectl cluster-info
```

## Deploy Tekon

Tekon is used to compile and deploy software inside the Kubernetes cluster.

```
kubectl apply -f kubernetes/tekton/release/tekton-pipelines-v0.13.2.yml
kubectl apply -f kubernetes/tekton/release/tekton-dashboard-v0.7.0.yml
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
kubectl create -f kubernetes/tekton/tasks/create-monitoring-grafana-admin-secret.yml
```

```
kubectl apply -f kubernetes/minio/minio-standalone-pvc.yaml
kubectl apply -f kubernetes/minio/minio-standalone-service.yaml
kubectl apply -f kubernetes/minio/minio-standalone-deployment.yaml
```

**Minio healthcheck needs to be passing**
```
kubectl wait --for=condition=ready pods --selector app=minio --timeout=300s
```

This will wait 5 minutes (should only take 2) for the minio container to be marked "healthy" and we can proceed.

You will get output like
```
pod/minio-757c6f5468-mv9cp condition met
```

Then create some buckets.

```
kubectl create -f kubernetes/tekton/tasks/create-minio-bucket.yml
kubectl create -f kubernetes/tekton/tasks/create-cache-bucket.yml
```

## Build zcashd

Create a zcashd binary by running the Tekton job. This will run the build task, grab the output binaries and upload the to internal S3 compatible storage.

```
kubectl create -f kubernetes/tekton/tasks/build-binary-tze.yml
```


