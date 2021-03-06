# Deploy TNB

TNB (testnet-in-a-box) will deploy a collection of zcashd instances to a Kubernetes cluster.

The deployment bundle will consist of a tar archive containing:
- An archive of binaries from a TNB build (zcashd, zcash-cli, zcash-gtest, zcash-tx) see the task for more info: [./kubernetes/tekton/tasks/build-binary-tnb.yml](./kubernetes/tekton/tasks/build-binary-tnb.yml)
- `zcash.conf` used for miners starting the testnet
- An archive of a block snapshot from the regular testnet

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

This command will wait 5 minutes (should only take 2) for the minio container to be marked "healthy" and we can proceed.

You will get output like
```
pod/minio-757c6f5468-mv9cp condition met
```

Then create some buckets.

```
kubectl create -f kubernetes/tekton/tasks/create-minio-bucket.yml
kubectl create -f kubernetes/tekton/tasks/create-cache-bucket.yml
```

## Deploy monitoring

The monitoring statefulset will collect metrics about the deployed nodes.

```
kubectl apply -f kubernetes/monitoring/configmap.yml
kubectl apply -f kubernetes/monitoring/service.yml
kubectl apply -f kubernetes/monitoring/serviceaccount.yml
kubectl apply -f kubernetes/monitoring/statefulset.yml
```

## Create the configmaps

```
kubectl apply -f kubernetes/template/configmaps-tnb.yml
```

## Import zcash params bundle

This task caches the zcashd parameters on mino for nodes to use.

```
kubectl create -f kubernetes/tekton/tasks/import-zcash-params.yml
```

## Import a tnb bundle

```
kubectl create -f kubernetes/tekton/tasks/import-zcash-tnb-bundle.yml 
```

## Review the configuration

All configuration is done through Kubernetes ConfigMaps.

Review the contents of `kubernetes/template/configmaps-tnb.yml`.
This will contain the the bundle name to use and `zcash.conf` contents.

It will also determine the number of replicas (miners) to run.

```
kubectl apply -f kubernetes/template/zcash-tnb-bundle-deploy.yml
```

## Peer the pods

**STILL A WORK IN PROGRESS**

Get the pods IPs. These values won't be available until the containers are started.

```
kubectl get pods -l app=zcash-with-exporter  -o jsonpath="{.items[*].status.podIP}"
```

You'll get something like

```
10.244.0.57 10.244.0.58
```

Get the pod name and put them in your env.

```
export pod1=$(kubectl get pods -l app=zcash-with-exporter  -o jsonpath="{.items[0].metadata.name}")
export pod2=$(kubectl get pods -l app=zcash-with-exporter  -o jsonpath="{.items[1].metadata.name}")
```

Use this name to verify the pod IP

```
$ kubectl exec  $pod1 -- ip a
Defaulting container name to zcashd-script.
Use 'kubectl describe pod/zcash-tnb-bundle-944fd6b5b-g4wdh -n default' to see all of the containers in this pod.
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0@if58: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether ba:31:cf:99:d1:f7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.244.0.57/24 brd 10.244.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::b831:cfff:fe99:d1f7/64 scope link 
       valid_lft forever preferred_lft forever
```

## Follow the logs of the miners to make sure they start up ok

```
kubectl logs -f $pod1 -c zcashd-script
```

## Add the peer node

Exec into pod1

```
kubectl exec -ti $pod1 -c zcashd-script -- bash
```

Peer with the other miner

```
${HOME}/workspace/source/src/zcash-cli -rpcpassword=${ZCASHD_RPCPASSWORD} addnode "10.244.0.58:18233" "add"
```


## View the cluster in grafana

Get the grafana admin password
```
kubectl get secrets monitoring-grafana-admin -o jsonpath="{.data.password}" | base64 -d
```

Open a tunnel to grafana

```
kubectl port-forward svc/monitoring 3000:3000
```

Open a browser to http://localhost:3000

Login as `admin` with the generated password secret.

From the left side menu, select `Dashboards`, then `Manage`.

Select `Import` from the top right.

In the text box labeld `Import via grafana.com` enter `12561` and click `Load`.

Select the prometheus and loki sources and import!

### TODO
Naming on the bundle isn't right
Manual "addnode" to get the peers talking.
