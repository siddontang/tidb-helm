# TiDB Operator

Refer to https://github.com/operator-framework/operator-sdk/blob/master/doc/helm/user-guide.md, create the tidb-operator with helm

```
operator-sdk new tidb-operator --helm-chart cluster --type helm
```

## Usage

### Create

```bash
kubectl create -f deploy/crds/charts_v1alpha1_cluster_crd.yaml

kubectl create -f deploy/service_account.yaml
kubectl create -f deploy/role.yaml
kubectl create -f deploy/role_binding.yaml
kubectl create -f deploy/operator.yaml

kubectl create -f deploy/crds/charts_v1alpha1_cluster_cr.yaml
```

### Scale

```bash
kubectl get pods -l app=pd
NAME                             READY   STATUS    RESTARTS   AGE
pd-0                             1/1     Running   0          2m
pd-1                             1/1     Running   0          2m
pd-2                             1/1     Running   2          2m
```

Change `deploy/crds/charts_v1alpha1_cluster_cr.yaml`, add:

```yaml
spec:
  # Default values copied from <project_dir>/helm-charts/cluster/values.yaml
  
  # Default values for cluster.
  # This is a YAML-formatted file.
  # Declare variables to be passed into your templates.
  
  pd:
    fullnameOverride: "pd"
    replicas: 5
```

Then:

```bash
kubectl apply -f deploy/crds/charts_v1alpha1_cluster_cr.yaml

kubectl get pods -l app=pd
NAME   READY   STATUS    RESTARTS   AGE
pd-0   1/1     Running   0          5m
pd-1   1/1     Running   0          5m
pd-2   1/1     Running   2          5m
pd-3   1/1     Running   0          2m
pd-4   1/1     Running   1          2m
```

### Cleanup

```bash
kubectl delete -f deploy/crds/charts_v1alpha1_cluster_cr.yaml
kubectl delete -f deploy/operator.yaml
kubectl delete -f deploy/role_binding.yaml
kubectl delete -f deploy/role.yaml
kubectl delete -f deploy/service_account.yaml
kubectl delete -f deploy/crds/charts_v1alpha1_cluster_crd.yaml
```

Notice, you may find that sometimes you cannot delete the customer resource. This problem is caused by the finalizer deadlock. 

The following way may work:

```bash
kubectl patch Cluster/example-cluster -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl patch crd/clusters.charts.helm.k8s.io -p '{"metadata":{"finalizers":[]}}' --type=merge
```

Some links may help you fix this problem:

- https://github.com/kubernetes/kubernetes/issues/60538#issuecomment-369099998
- https://rook.io/docs/rook/v0.8/ceph-teardown.html#troubleshooting

