# MINIO Operator
https://min.io/docs/minio/kubernetes/upstream/operations/installation.html

```
kubectl apply -k "github.com/minio/operator?ref=v7.0.0"
kubectl apply -k "github.com/minio/operator?ref=v7.0.0" --dry-run=client -o yaml >> minio-operator.yaml
```
### error: 
```
4m          Warning   FailedCreate        replicaset/minio-operator-fcbcfb669   Error creating: pods "minio-operator-fcbcfb669-" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount, provider restricted-v2: .containers[0].runAsUser: Invalid value: 1000: must be in the ranges: [1000740000, 1000749999], provider "restricted": Forbidden: not usable by user or serviceaccount, provider "nonroot-v2": Forbidden: not usable by user or serviceaccount, provider "nonroot": Forbidden: not usable by user or serviceaccount, provider "noobaa-db": Forbidden: not usable by user or serviceaccount, provider "noobaa": Forbidden: not usable by user or serviceaccount, provider "noobaa-endpoint": Forbidden: not usable by user or serviceaccount, provider "hostmount-anyuid": Forbidden: not usable by user or serviceaccount, provider "machine-api-termination-handler": Forbidden: not usable by user or serviceaccount, provider "hostnetwork-v2": Forbidden: not usable by user or serviceaccount, provider "hostnetwork": Forbidden: not usable by user or serviceaccount, provider "hostaccess": Forbidden: not usable by user or serviceaccount, provider "rook-ceph": Forbidden: not usable by user or serviceaccount, provider "node-exporter": Forbidden: not usable by user or serviceaccount, provider "rook-ceph-csi": Forbidden: not usable by user or serviceaccount, provider "privileged": Forbidden: not usable by user or serviceaccount]
```
### use 
```
oc adm policy add-scc-to-user privileged  system:serviceaccount:minio-operator:minio-operator
```

https://pet2cattle.com/2022/09/openshift-assign-securitycontextconstraints-serviceaccount

## Deploy with prepared yamls:
```
oc apply -f minio-operator.yaml
oc apply -f scc-minio-operator.yaml
```
### Check if operator pods are running:
```
oc get pods -n minio-operator
NAME                             READY   STATUS    RESTARTS   AGE
minio-operator-fcbcfb669-2gjm8   1/1     Running   0          143m
minio-operator-fcbcfb669-mcmgx   1/1     Running   0          143m
```
### Deploy a MinIO Tenant using Kustomize

https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-minio-tenant.html#minio-k8s-deploy-minio-tenant

```
kubectl kustomize https://github.com/minio/operator/examples/kustomization/base/ > tenant-base.yaml                  # Kubernetes
kubectl kustomize https://github.com/minio/operator/examples/kustomization/tenant-openshift/ > tenant-openshift.yaml # Openshift


vi  tenant-base.yaml

servers: 2 #4  #minimal is 2
volumesPerServer: 2 #4
volumeClaimTemplate.spec.storageClassName: ocs-external-storagecluster-ceph-rbd #standard
volumeClaimTemplate.spec.resources.requests.storage: 1Ti
```

```
oc adm policy add-scc-to-user privileged  system:serviceaccount:minio-tenant:myminio-sa
```
### route: 
```
oc create route edge --service myminio-console
```
#### or 
```
oc apply -f route.yaml
```

## Deploy with yamls:
```
oc apply -f tenant-base.yaml      # Kubernetes
oc apply -f tenant-openshift.yaml # Openshift
oc apply -f scc-myminio-sa.yaml
oc apply -f route.yaml 
```
### pasword access_key in tenant-base.yaml or:
```
cat tenant-base.yaml
  config.env: |-
    export MINIO_ROOT_USER="minio"
    export MINIO_ROOT_PASSWORD="minio123"

oc get secret -n minio-tenant storage-user -o yaml
data:
  CONSOLE_ACCESS_KEY: Y29uc29sZQ==
  CONSOLE_SECRET_KEY: Y29uc29sZTEyMw==

oc get routes
NAME              HOST/PORT                                                                    PATH   SERVICES          PORT            TERMINATION   WILDCARD
myminio-console   myminio-console-minio-tenant.apps.cluster-k2j4h.dynamic.redhatworkshops.io          myminio-console   https-console   passthrough   None

```
### to reach Minio API test IP or service to have propper endpoint:
```
oc get pod -o wide
NAME                   READY   STATUS    RESTARTS   AGE     IP            NODE                     NOMINATED NODE   READINESS GATES
myminio-standalone-0   2/2     Running   0          2m14s   10.135.0.58   worker-cluster-p77z5-1   <none>           <none>

oc get all

NAME                       READY   STATUS     RESTARTS   AGE
pod/myminio-standalone-0   0/2     Init:0/1   0          3s

NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/minio             ClusterIP   172.31.102.100   <none>        443/TCP    4s
service/myminio-console   ClusterIP   172.31.213.236   <none>        9443/TCP   4s
service/myminio-hl        ClusterIP   None             <none>        9000/TCP   4s


curl 10.135.0.58:9000
curl http://myminio-hl.minio-tenant.svc.cluster.local:9000
curl myminio-hl.minio-tenant.svc:9000
curl http://myminio-hl.minio-tenant.svc:9000

Client sent an HTTP request to an HTTPS server. # need to disable requestAutoCert
    export MINIO_STORAGE_CLASS_STANDARD="EC:0" #use for 1 PVC, see https://min.io/docs/minio/linux/reference/minio-server/settings/storage-class.html
	  requestAutoCert: false # if you want to enable you have to modify Loki-stack with tls ca crt

```
## Create bucket and user with access_key and secure_key:
```
  oc exec -it myminio-standalone-0 -n minio-tenant -- mc alias set myminio http://localhost:9000 minio minio123
  oc exec -it myminio-standalone-0 -n minio-tenant -- mc mb myminio/loki-bucket
  oc exec -it myminio-standalone-0 -n minio-tenant -- mc admin user add myminio loki-user loki-pass
  oc exec -it myminio-standalone-0 -n minio-tenant -- mc admin policy attach myminio readwrite --user=loki-user

  #if TLS enabled you can use --insecure
  oc exec -it myminio-pool-0-0 -- mc alias set myminio https://localhost:9000 minio minio123 --insecure
  oc exec -it myminio-pool-0-0 -- mc mb myminio/mybucket --insecure
```
