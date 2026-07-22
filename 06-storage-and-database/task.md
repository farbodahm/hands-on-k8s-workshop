# Task 06 - Run Postgres and connect the app

## Goal

Run a single Postgres instance with persistent storage as a StatefulSet, then
have your Go app connect to it over the cluster network.

## Expected outcome

- A StatefulSet `postgres` runs one Pod with a bound PVC.
- Data written to Postgres survives deleting and recreating the Pod.
- The Go app's `/db` endpoint returns a value fetched from Postgres.

## How to check

```sh
kubectl get statefulset postgres      # READY 1/1
kubectl get pvc                        # data-postgres-0, STATUS Bound
kubectl get pv                         # a provisioned volume, Bound

# persistence check:
kubectl exec -it postgres-0 -- psql -U app -d appdb -c \
  "CREATE TABLE t (x int); INSERT INTO t VALUES (42);"
kubectl delete pod postgres-0
kubectl wait --for=condition=ready pod postgres-0 --timeout=60s
kubectl exec -it postgres-0 -- psql -U app -d appdb -c "SELECT * FROM t;"
# returns 42 after the restart

# app-to-db check (after adding the /db endpoint):
kubectl port-forward deployment/hello 8080:8080 &
curl localhost:8080/db
# returns a value from the database, e.g. the current time or "1"
```

## Steps

1. Confirm the `db-credentials` Secret from module 05 still exists.
2. Write and apply `postgres.yaml` (headless Service plus StatefulSet with a
   volumeClaimTemplate). Confirm the PVC and PV are Bound.
3. Verify persistence: create a table and row, delete `postgres-0`, wait for it
   to return, and confirm the row is still there.
4. Extend `main.go` with a `/db` endpoint that connects using `DB_HOST`,
   `DB_NAME`, and the credentials from the Secret, runs a simple query, and
   returns the result. Rebuild as `hello-go:0.3` and import it.
5. Update the `hello` Deployment with the DB env vars and the new image tag.
   Apply and rollout.
6. Curl `/db` and confirm the app read from Postgres.

## Hints

- An unbound PVC (`Pending`) usually means no StorageClass. Check
  `kubectl get storageclass`; `local-path` should be marked default on K3s.
- The app cannot reach the database until the `postgres` Service has an
  endpoint. Check `kubectl get endpoints postgres` before debugging the app.
- Connection refused from the app often means the DB host or port is wrong, or
  Postgres has not finished starting. `kubectl logs postgres-0` shows when it is
  ready to accept connections.
- Build the connection string from env vars, not hardcoded values. That is the
  whole reason you wired up the Secret and ConfigMap in module 05.
- If the app Pod starts before Postgres is ready and crashes, that is a preview
  of module 07: readiness probes and restart behavior handle exactly this.
