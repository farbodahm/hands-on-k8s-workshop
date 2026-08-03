# Task 14 - Run a replicated Postgres with an operator

## Goal

Install the CloudNativePG operator, declare a 3-instance Postgres cluster with
one manifest, and prove that the operator handles failover for you. Along the
way, inspect the CRDs and the resources the operator creates, and compare the
result with the Postgres you built by hand in module 06.

## Expected outcome

- The CloudNativePG controller runs as a Deployment in the `cnpg-system`
  namespace, and its CRDs are installed.
- A `Cluster` resource named `pg` runs 3 instances: one primary, two replicas.
- Data written on the primary is readable after the primary Pod is deleted,
  because the operator promotes a replica automatically.

## How to check

```sh
kubectl get deploy -n cnpg-system          # cnpg-cloudnative-pg, READY 1/1
kubectl get crds | grep cnpg               # clusters, backups, poolers, ...

kubectl get cluster pg                     # 3 instances, 3 ready,
                                           # STATUS "Cluster in healthy state"
kubectl get pods -l cnpg.io/cluster=pg     # pg-1, pg-2, pg-3 Running
kubectl get svc                            # pg-rw, pg-ro, pg-r

# who is primary right now:
kubectl get cluster pg -o jsonpath='{.status.currentPrimary}{"\n"}'

# write data on the primary (psql runs as postgres inside the Pod):
kubectl exec -it <primary-pod> -- psql -d app -c \
  "CREATE TABLE t (x int); INSERT INTO t VALUES (42);"

# kill the primary and watch the failover:
kubectl delete pod <primary-pod>
kubectl get cluster pg -w                  # currentPrimary changes within seconds

# the data is still there on the new primary:
kubectl exec -it <new-primary-pod> -- psql -d app -c "SELECT * FROM t;"
# returns 42
```

## Steps

1. If module 13 removed your cluster, recreate it with the module 04 command.
2. Install the operator with Helm into `cnpg-system` (commands in the README).
   Wait for its Deployment to be ready and look at what appeared in
   `kubectl get crds`.
3. Write `pg-cluster.yaml` with a `Cluster` of 3 instances and 1Gi storage, and
   apply it. Watch the Pods appear one by one with
   `kubectl get pods -l cnpg.io/cluster=pg -w`; the first instance initializes
   before the replicas join.
4. List the Services and PVCs the operator created and compare with what you
   wrote by hand in module 06.
5. Find the current primary, create a table with one row on it, delete that
   Pod, and confirm a replica is promoted and still has the row.
6. Optional: point the module 06 app at `pg-rw` instead of your hand-rolled
   `postgres` Service. The app does not care which Pod is primary; that is what
   the role-based Services are for.

## Hints

- The first startup pulls Postgres images from the internet and initializes a
  database, so give it a minute or two. `kubectl describe cluster pg` and the
  events in `kubectl get events --sort-by=.lastTimestamp` show progress.
- `kubectl get cluster` works because of the CRD's short name. If a name ever
  collides with another installed CRD, use the full form:
  `kubectl get clusters.postgresql.cnpg.io`.
- Pods stuck in `Pending` usually mean PVCs are not binding. Same diagnosis as
  module 06: `kubectl get pvc` and `kubectl get storageclass`.
- The operator creates a Secret named `pg-app` with credentials for the `app`
  database, the same role the module 06 Secret played. Look inside it with
  `kubectl get secret pg-app -o yaml` if you do the optional step.
- If the failover seems instant, that is not an error. Promoting a streaming
  replica is fast; rebuilding the instance you deleted takes longer. Watch
  `kubectl get pods -l cnpg.io/cluster=pg` until all three are back.
- To start over, `kubectl delete cluster pg` removes the database Pods and
  Services but keeps the operator. Deleting the PVCs too gives you a truly
  clean slate.
