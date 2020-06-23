
## Import base configmaps

These are the shared configurations available for deployed instances. ConfigMaps can be combined.

```
kubectl apply -f kubernetes/template/configmaps.yml
```

## Import a block snapshot

The `blocks` and `chainstate` archive all nodes in the deployment will start with.

```
kubectl create -f kubernetes/tekton/tasks/import-block-snapshot-minio.yml
```

## Import zcash params

Cache zcash params for faster node start times.

```
kubectl create -f kubernetes/tekton/tasks/import-zcash-params.yml 
```

## EDIT CONFIGMAP FOR DEPLOY

The test deployment will use the values of `kubernetes/template/configmaps-testing.yml`.

Customize for the build artifact and block snapshot names in mino.

Then, apply the deployment.


```
kubectl apply -f kubernetes/template/zcash-script-deploy-miner.yml
```