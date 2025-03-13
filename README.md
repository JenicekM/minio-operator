### MINIO Operator
https://min.io/docs/minio/kubernetes/upstream/operations/installation.html

```
kubectl apply -k "github.com/minio/operator?ref=v7.0.0"

kubectl apply -k "github.com/minio/operator?ref=v7.0.0" --dry-run=client -o yaml >> minio-operator.yaml
```
if 
```
4m          Warning   FailedCreate        replicaset/minio-operator-fcbcfb669   Error creating: pods "minio-operator-fcbcfb669-" is forbidden: unable to validate against any security context constraint: [provider "anyuid": Forbidden: not usable by user or serviceaccount, provider restricted-v2: .containers[0].runAsUser: Invalid value: 1000: must be in the ranges: [1000740000, 1000749999], provider "restricted": Forbidden: not usable by user or serviceaccount, provider "nonroot-v2": Forbidden: not usable by user or serviceaccount, provider "nonroot": Forbidden: not usable by user or serviceaccount, provider "noobaa-db": Forbidden: not usable by user or serviceaccount, provider "noobaa": Forbidden: not usable by user or serviceaccount, provider "noobaa-endpoint": Forbidden: not usable by user or serviceaccount, provider "hostmount-anyuid": Forbidden: not usable by user or serviceaccount, provider "machine-api-termination-handler": Forbidden: not usable by user or serviceaccount, provider "hostnetwork-v2": Forbidden: not usable by user or serviceaccount, provider "hostnetwork": Forbidden: not usable by user or serviceaccount, provider "hostaccess": Forbidden: not usable by user or serviceaccount, provider "rook-ceph": Forbidden: not usable by user or serviceaccount, provider "node-exporter": Forbidden: not usable by user or serviceaccount, provider "rook-ceph-csi": Forbidden: not usable by user or serviceaccount, provider "privileged": Forbidden: not usable by user or serviceaccount]
```
use 
```
oc adm policy add-scc-to-user privileged  system:serviceaccount:minio-operator:minio-operator
```

https://pet2cattle.com/2022/09/openshift-assign-securitycontextconstraints-serviceaccount
or 
```
oc apply -f minio-operator.yaml
oc apply -f scc-minio-operator.yaml
```

```
oc get pods -n minio-operator
NAME                             READY   STATUS    RESTARTS   AGE
minio-operator-fcbcfb669-2gjm8   1/1     Running   0          143m
minio-operator-fcbcfb669-mcmgx   1/1     Running   0          143m
```
Deploy a MinIO Tenant using Kustomize

https://min.io/docs/minio/kubernetes/upstream/operations/install-deploy-manage/deploy-minio-tenant.html#minio-k8s-deploy-minio-tenant

```
kubectl kustomize https://github.com/minio/operator/examples/kustomization/base/ > tenant-base.yaml

vi  tenant-base.yaml

servers: 2 #4  #minimal is 2
volumesPerServer: 2 #4
volumeClaimTemplate.spec.storageClassName: ocs-external-storagecluster-ceph-rbd #standard
volumeClaimTemplate.spec.resources.requests.storage: 1Ti
```

```
oc adm policy add-scc-to-user privileged  system:serviceaccount:minio-tenant:myminio-sa
```
route: 
```
oc create route passthrough --service myminio-console
```

or 
```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myminio-console
  namespace: minio-tenant
spec:
  host: myminio-console-minio-tenant.apps.cluster-k2j4h.dynamic.redhatworkshops.io #modify hostname
  port:
    targetPort: https-console
  tls:
    termination: passthrough
  to:
    kind: Service
    name: myminio-console
    weight: 100
  wildcardPolicy: None
status:  {}
```

or 
```
oc apply -f tenant-base.yaml
oc apply -f scc-myminio-sa.yaml
oc apply -f route.yaml
```

Create bucket:
```
oc exec -it myminio-pool-0-0 -- mc alias set myminio https://localhost:9000 minio minio123 --insecure
oc exec -it myminio-pool-0-0 -- mc mb myminio/mybucket --insecure
```

