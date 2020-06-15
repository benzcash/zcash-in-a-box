# Zcash network in kubernetes - sec

Abbreviated example of using a secured git repository.


### Kubernetes cluster


Using `kind`
```
kind create cluster --name zcash-in-a-box-sec
kubectl cluster-info
```

## Generate testing key

We'll generate a new key that will be granted access to a secured git repository.

**DO NOT SET A PASSWORD** just hit enter twice.

```
ssh-keygen -t rsa -b 2048 -C 'Testing zcash-in-a-box-sec' -f ./zcash-in-a-box-sec
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in ./zcash-in-a-box-sec
Your public key has been saved in ./zcash-in-a-box-sec.pub
```

## Assign permission for zcash-in-a-box-sec with the public key

This step depends on where your secured git repository is located. Add a new user to be identified with `./zcash-in-a-box-sec.pub`, or add this key to an existing user.


## Deploy Tekon

Tekon is used to compile and deploy software inside the Kubernetes cluster.

```
kubectl apply -f kubernetes/tekton/release/tekton-pipelines-v0.13.0.yml
kubectl apply -f kubernetes/tekton/release/tekton-dashboard-v0.7.0.yml
kubectl apply -f kubernetes/tekton/serviceAccount-sec.yml
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


## Configure secured git repo access

Generate base64 string of git repo's hosts public ssh key.

```
ssh-keyscan github.com  2>/dev/null | base64 -w0 > git_host.pub
```

Generate base64 string of the private ssh key.

```
base64 -w0 < ./zcash-in-a-box-sec > zcash-in-a-box-sec.b64
```

**EDIT** `test-secret.yml`
Read the comments and replace those sections (we'll get something automated later).


Create the secret. The `tekton` service account is configured to use this specific secret, any other changes could cause it to fail.

```
kubectl apply -f test-secret.yml
```

Ref: https://github.com/tektoncd/pipeline/blob/master/docs/auth.md

## Build zcashd

**EDIT** kubernetes/tekton/tasks/build-binary-sec.yml

This section describes the git repository and commit to build from, the url value must match the `known_hosts` used.

```
        resourceSpec:
          type: git
          params:
            - name: revision
              value: tekton-task-build
            - name: url
              value: https://github.com/benzcash/zcash.git
```

Create a zcashd binary by running the Tekton job. This will run the build task, grab the output binaries and upload the to internal S3 compatible storage.

```
kubectl create -f kubernetes/tekton/tasks/build-binary-sec.yml
```


