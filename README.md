# TiDB Helm

This is just a toy to operate TiDB cluster in Kubernetes.

## Basic Usage

Before you start, you need to deploy a Kubernetes, and create a PV.

```bash
# Start PD cluster
helm install pd --name pd 

# Start TiKV cluster
helm install tikv --name tikv 

# Start TiDB cluster
helm install tidb --name tidb 
```

In another shell, you can watch the progress, like:

```
kubectl get pods -w -l cluster=tidb-cluster
NAME      READY     STATUS    RESTARTS   AGE
pd-0      1/1       Running   0          30s
pd-1      1/1       Running   0          29s
pd-2      1/1       Running   2          27s
```

After all Pods running, you can login TiDB and operate the cluster, like:

```bash
# Another shell, forward 4000 port
kubectl port-forward svc/tidb 4000

# Use following to operate TiDB
mysql -h 127.0.0.1 -P 4000 -u root  -p
mysql> use test;
mysql> create table t1 (id int primary key, name varchar(1024));
mysql> insert into t1 values (1, "a");
Query OK, 1 row affected (0.06 sec)

mysql> select * from t1;
+----+------+
| id | name |
+----+------+
|  1 | a    |
+----+------+
```

## Scale-up

For scale-up, very easy, just do:

```bash
# Use 5 replicas for PD
kubectl patch statefulset/pd -p '{"spec":{"replicas": 5}}'

# Use 5 nodes for TiKV
# Notice: here we just use 5 nodes, not 5 replicas for a region in TiKV.
# If you want to use 5 replicas for a region, you should use `pd-ctl config set max-replicas 5`
kubectl patch statefulset/tikv -p '{"spec":{"replicas": 5}}'
```

You can also use `scale` sub-command, like:

```bash
kubectl scale deployment tidb --replicas=5
```

## Scale-down

TiDB server is stateless, it is easy to do scale down, like:

```bash
kubectl scale deployment tidb --replicas=3
``` 

Scaling down PD and TiKV is hard.

## Scale-down PD

List pods and pvc for PD.

```bash
kubectl get pods,pvc -l app=pd
NAME       READY     STATUS    RESTARTS   AGE
pod/pd-0   1/1       Running   0          21m
pod/pd-1   1/1       Running   0          20m
pod/pd-2   1/1       Running   1          20m

NAME                                 STATUS    VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/datadir-pd-0   Bound     local-pv-857a68f4   3724Gi     RWO            shared-nvme    21m
persistentvolumeclaim/datadir-pd-1   Bound     local-pv-2b676486   3724Gi     RWO            shared-nvme    20m
persistentvolumeclaim/datadir-pd-2   Bound     local-pv-d714dd92   3724Gi     RWO            shared-nvme    20m
```

List the PD members and delete "pd-2".

```bash
kubectl exec -ti pd-0 -- sh -c '/pd-ctl -d -u pd-0.pd:2379 member list | jq -c ".members[] | {name, member_id}"'
{"name":"pd-1","member_id":7019544700859125000}
{"name":"pd-0","member_id":15488656168250106000}
{"name":"pd-2","member_id":17965809889215412000}

kubectl exec -ti pd-0 -- sh -c '/pd-ctl -d -u pd-0.pd:2379 member delete name pd-2'

kubectl exec -ti pd-0 -- sh -c '/pd-ctl -d -u pd-0.pd:2379 member list | jq -c ".members[] | {name, member_id}"'
{"name":"pd-1","member_id":7019544700859125000}
{"name":"pd-0","member_id":15488656168250106000}
```

Change PD replicas and remove the pvc, you must delete associated pvc here to later scale-up, maybe we can improve it later.

```bash
kubectl patch statefulset/pd -p '{"spec":{"replicas": 2}}'
statefulset.apps "pd" patched

kubectl delete pvc datadir-pd-2
persistentvolumeclaim "datadir-pd-2" deleted

kubectl get pods,pvc -l app=pd
NAME       READY     STATUS    RESTARTS   AGE
pod/pd-0   1/1       Running   0          26m
pod/pd-1   1/1       Running   0          26m

NAME                                 STATUS    VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/datadir-pd-0   Bound     local-pv-857a68f4   3724Gi     RWO            shared-nvme    26m
persistentvolumeclaim/datadir-pd-1   Bound     local-pv-2b676486   3724Gi     RWO            shared-nvme    26m
``` 

Scale up again to check the member ID of "pd-2" is different from the old

```bash
kubectl patch statefulset/pd -p '{"spec":{"replicas": 3}}'
statefulset.apps "pd" patched

kubectl exec -ti pd-0 -- sh -c '/pd-ctl -d -u pd-0.pd:2379 member list | jq -c ".members[] | {name, member_id}"'
{"name":"pd-1","member_id":7019544700859125000}
{"name":"pd-2","member_id":10581645324753740000}
{"name":"pd-0","member_id":15488656168250106000}
```

## Scale-down TiKV

To scale down TiKV, you can do the same as PD.

List pods and pvc for TiKV.
```bash
kubectl get pods,pvc -l app=tikv
NAME         READY     STATUS    RESTARTS   AGE
pod/tikv-0   1/1       Running   0          1m
pod/tikv-1   1/1       Running   0          1m
pod/tikv-2   1/1       Running   0          1m
pod/tikv-3   1/1       Running   0          39s
pod/tikv-4   1/1       Running   0          34s

NAME                                   STATUS    VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/datadir-tikv-0   Bound     local-pv-857a68f4   3724Gi     RWO            shared-nvme    1m
persistentvolumeclaim/datadir-tikv-1   Bound     local-pv-7a7eafcb   3724Gi     RWO            shared-nvme    1m
persistentvolumeclaim/datadir-tikv-2   Bound     local-pv-50b561fd   3724Gi     RWO            shared-nvme    1m
persistentvolumeclaim/datadir-tikv-3   Bound     local-pv-d714dd92   3724Gi     RWO            shared-nvme    39s
persistentvolumeclaim/datadir-tikv-4   Bound     local-pv-47a7d73b   3724Gi     RWO            shared-nvme    34s
```

List the TiKV members and delete "tikv-4".
```bash
kubectl exec -ti pd-0 -- sh -c '/pd-ctl -d -u pd-0.pd:2379 store | jq ".stores[] | {id: .store.id, addr: .store.address, state: .store.state_name}" -c'
{"id":5,"addr":"tikv-1.tikv:20160","state":"Up"}
{"id":8,"addr":"tikv-3.tikv:20160","state":"Up"}
{"id":9,"addr":"tikv-4.tikv:20160","state":"Up"}
{"id":1,"addr":"tikv-0.tikv:20160","state":"Up"}
{"id":4,"addr":"tikv-2.tikv:20160","state":"Up"}

# The ID of tikv-4 is 9
kubectl exec -ti pd-0 -- sh -c '/pd-ctl -d -u pd-0.pd:2379 store delete 9'

kubectl exec -ti pd-0 -- sh -c '/pd-ctl -d -u pd-0.pd:2379 store | jq ".stores[] | {id: .store.id, addr: .store.address, state: .store.state_name}" -c'
{"id":4,"addr":"tikv-2.tikv:20160","state":"Up"}
{"id":5,"addr":"tikv-1.tikv:20160","state":"Up"}
{"id":8,"addr":"tikv-3.tikv:20160","state":"Up"}
{"id":1,"addr":"tikv-0.tikv:20160","state":"Up"}
```

Change TiKV replicas and remove the pvc, you must delete associated pvc here to later scale-up, maybe we can improve it later.

```bash
kubectl patch statefulset/tikv -p '{"spec":{"replicas": 4}}'
statefulset.apps "tikv" patched

kubectl delete pvc datadir-tikv-4
persistentvolumeclaim "datadir-tikv-4" deleted
```

Scale up again to check the ID of "tikv-4" is different from the old

```bash
kubectl patch statefulset/tikv -p '{"spec":{"replicas": 5}}'
statefulset.apps "tikv" patched

kubectl exec -ti pd-0 -- sh -c '/pd-ctl -d -u pd-0.pd:2379 store | jq ".stores[] | {id: .store.id, addr: .store.address, state: .store.state_name}" -c'
{"id":10,"addr":"tikv-4.tikv:20160","state":"Up"}
{"id":1,"addr":"tikv-0.tikv:20160","state":"Up"}
{"id":4,"addr":"tikv-2.tikv:20160","state":"Up"}
{"id":5,"addr":"tikv-1.tikv:20160","state":"Up"}
{"id":8,"addr":"tikv-3.tikv:20160","state":"Up"}
```

## Clean up

```bash
helm delete tidb --purge
helm delete tikv --purge 
helm delete pd --purge 

kubectl delete pvc -l cluster=tidb-cluster
```
