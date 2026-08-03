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

## What is a headless Service?

`clusterIP: None` on the `postgres` Service makes it headless. The "head" is the
virtual IP a normal Service gets, and here you are asking Kubernetes not to
create one.

A ClusterIP Service (module 03) gets an address like `10.43.12.7`. No Pod owns
that address and no network interface has it. It is a fiction maintained by
iptables rules that kube-proxy writes on every node. DNS returns that single
address, and the rules rewrite each connection to one of the backing Pods.

A headless Service skips all of it. No virtual IP is allocated and kube-proxy
writes no rules. CoreDNS returns the Pod addresses themselves, one A record per
ready Pod, and the client connects straight to a Pod.

```sh
kubectl get svc
# hello      ClusterIP   10.43.12.7   80/TCP
# postgres   ClusterIP   None         5432/TCP

kubectl run dns-test --rm -it --image=busybox:1.36 --restart=Never -- \
  sh -c 'nslookup hello; nslookup postgres'
# hello resolves to the fake 10.43.x.x, postgres to a real 10.42.x.x pod IP
```

The payoff is per-Pod DNS names. With a headless Service and `serviceName:
postgres`, every Pod in the StatefulSet gets a permanent address of its own:

```
postgres-0.postgres.default.svc.cluster.local
postgres-1.postgres.default.svc.cluster.local
```

That name survives restarts, because a StatefulSet always recreates `postgres-0`
as `postgres-0`. Through a ClusterIP you could never ask for a specific replica;
the virtual IP would send you somewhere random.

Databases need that. In a replicated Postgres, one instance is the primary that
accepts writes and the rest are read replicas, so the client has to name the one
it wants. Spreading writes across all three would fail two times out of three.

With a single replica the two kinds behave identically. You write it headless
because that is what the pattern looks like once there is more than one Pod.

## Why not put the app and the database under one Service?

A Service is not a box you put things in. It is three things at once: a label
selector, a stable name, and load balancing across every Pod the selector
matches. Grouping related components is not one of its jobs.

Cover both with one Service and the selector is the first problem. Selector
terms are ANDed, so `app: hello` plus `app: postgres` matches nothing. You would
have to invent a shared label.

Suppose you did. The endpoint list would look like this:

```sh
kubectl get endpoints combined
# 10.42.0.5:8080, 10.42.0.6:8080, 10.42.0.7:5432
```

Every connection to that name now lands on a random member. Sometimes a request
reaches the Go app, sometimes it opens a socket to Postgres and gets binary wire
protocol back. Nothing routes HTTP to the HTTP one. Load balancing is blind to
what sits behind it.

The two also want opposite exposure. `hello` is published to the outside world
through your Ingress. `postgres` has to stay reachable only from inside the
cluster. One Service means one policy for both.

One Service per role, and roles find each other by DNS name. The next question
takes this one level down: why not one Pod either?

## What goes in one Pod and what gets its own?

Things share a Pod only when they cannot do their job apart. Everything else
gets its own Pod, which in practice means almost everything does.

Containers in a Pod are welded together in four ways: they scale together, they
land on the same node together, they restart and die together, and they share
one IP. That gives you four questions to ask about any two components:

1. Do they scale together, always, exactly? If you could ever want 5 of one and
   2 of the other, separate Pods.
2. Can they be deployed independently? Same Pod means every deploy of one
   restarts the other.
3. Do they have different lifecycles? A database that must survive an app
   redeploy cannot live in the app's Pod.
4. Would one crashing justify restarting the other?

Any "no" means separate Pods. This module is the sharpest example: put Postgres
in the app's Pod and `replicas: 2` gives you two Go apps and two disconnected
databases with two separate volumes, each request hitting one or the other at
random.

Take a bigger business to see the pattern: an API service, an orders service, a
payments service, Postgres, Redis, and Kafka.

| Workload | Packaging | Which question fails if combined |
|---|---|---|
| API service | Own Deployment | Scales on HTTP traffic (1) |
| Orders service | Own Deployment | Ships on its own schedule (2) |
| Payments service | Own Deployment | Same (2) |
| Postgres | Own StatefulSet | Must outlive app deploys (3) |
| Redis | Own StatefulSet | Different lifecycle than any app (3) |
| Kafka brokers | Own StatefulSet, one Pod per broker | Individually addressable, upgraded one at a time (1, 3) |

One Pod template per thing that scales as a unit, and the Pods find each other
through Services by DNS name. The architecture diagram and the list of
Deployments and StatefulSets come out nearly one-to-one.

Kafka stress-tests the rule from both sides. Three brokers are three Pods from
one StatefulSet, not three containers in one Pod: they must land on different
nodes to survive a node failure, and clients address a specific broker, which is
why Kafka uses a headless Service just like Postgres here. And its coordination
layer (Zookeeper, or KRaft controllers) is a separate StatefulSet even though
Kafka cannot run without it, because you run 3 of those and maybe 9 brokers, so
question 1 fails.

So when do containers share a Pod? When the second container is per-instance
glue for the first: a log shipper tailing the app's files from a shared volume,
a service-mesh proxy that must share the app's IP to intercept its traffic, or
an init container that waits for the database before the app starts. Ten app
replicas need exactly ten log shippers, one glued to each. Scaling is inherently
1:1, so all four questions pass. The smell test: if the helper could conceivably
serve two replicas of the main container, it does not belong in the Pod.

If you come from docker-compose, the trap is mapping one compose file to one
Pod. The right mapping is one compose service to one Deployment or StatefulSet.
A Pod is not the grouping mechanism; namespaces, labels, and Helm charts are
(modules 11 and 12). Pods are the scaling unit, and putting two things in one
Pod declares they scale, ship, and die together forever.
