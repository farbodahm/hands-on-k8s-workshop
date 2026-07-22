# 06 - Storage and a database

Containers are ephemeral: when a Pod dies, everything written inside it is gone.
A database needs its data to survive restarts. This module covers persistent
storage and uses it to run Postgres, then connects your Go app to it.

## The storage vocabulary

- **Volume**: storage attached to a Pod. Some volume types vanish with the Pod;
  others outlive it.
- **PersistentVolume (PV)**: a piece of storage in the cluster, independent of
  any Pod.
- **PersistentVolumeClaim (PVC)**: a request for storage. A Pod references a PVC,
  and the cluster binds it to a PV that satisfies the request.
- **StorageClass**: describes a kind of storage and can create PVs on demand
  (dynamic provisioning). K3s ships with `local-path` as the default, which
  carves storage out of the node's disk. This means you write a PVC and the PV
  appears automatically.

## Why StatefulSet for a database

A Deployment treats its Pods as interchangeable and gives them random names. A
database usually wants a stable identity and its own storage that follows it
across restarts. A StatefulSet provides:

- Stable, ordered Pod names (`postgres-0`, `postgres-1`, ...).
- A `volumeClaimTemplate` that gives each Pod its own PVC automatically.
- A stable network identity through a headless Service.

For a single-instance Postgres you could get away with a Deployment plus one
PVC, but using a StatefulSet teaches the pattern you need for real stateful
workloads.

## Running Postgres

Use the `db-credentials` Secret from module 05 for the username and password.

`postgres.yaml` (headless Service plus StatefulSet):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None          # headless: gives stable DNS to each pod
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          envFrom:
            - secretRef:
                name: db-credentials
          env:
            - name: POSTGRES_DB
              value: appdb
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

`postgres:16` is pulled from Docker Hub automatically, so no image import.

```sh
kubectl apply -f postgres.yaml
kubectl get statefulset postgres
kubectl get pvc                 # a PVC named data-postgres-0, bound
kubectl get pv                  # the PV that local-path provisioned
```

## Connecting the app

Your Go app reaches Postgres by the Service name `postgres` on port 5432, thanks
to cluster DNS. Give the app the connection details through env vars, reusing
the same Secret:

```yaml
env:
  - name: DB_HOST
    value: postgres
  - name: DB_NAME
    value: appdb
envFrom:
  - secretRef:
      name: db-credentials
```

To make the app actually query the database, extend `main.go` with a `/db`
endpoint. Add the `github.com/jackc/pgx/v5` driver and open a connection using
the env vars, then run a trivial query such as `SELECT 1` or `SELECT now()` and
return the result. Rebuild the image as `hello-go:0.3`, import it, and update
the Deployment.

## Proving persistence

The point of a PVC is that data survives Pod restarts. Write data, delete the
Pod, and confirm the data is still there when the StatefulSet recreates it.

```sh
kubectl exec -it postgres-0 -- psql -U app -d appdb -c \
  "CREATE TABLE t (x int); INSERT INTO t VALUES (42);"
kubectl delete pod postgres-0        # StatefulSet recreates it, same PVC
kubectl exec -it postgres-0 -- psql -U app -d appdb -c "SELECT * FROM t;"
# still returns 42
```
