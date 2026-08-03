# 14 - Operators

Every resource you have used so far (Deployment, Service, StatefulSet, Job) is
built into Kubernetes, and each one has a controller in the control plane that
makes reality match your YAML. Operators extend that machinery: they add new
resource types to your cluster and ship the controller that knows what to do
with them.

If you deleted the `learn` cluster in module 13, recreate it first (the module
04 command with the port mapping) so you have something to install into.

## The pattern you already know

When you apply a Deployment asking for 3 replicas, nothing about your `kubectl`
command creates Pods. The Deployment controller does. It runs a loop: watch
Deployment objects, compare desired state to actual state, act to close the
gap. If you kill a Pod, the loop notices and replaces it. You saw this in
module 02 and have relied on it ever since.

That watch-and-reconcile loop is the core mechanic of Kubernetes. The control
plane is a collection of such loops, one per resource type. The insight behind
operators is that this mechanic is not reserved for built-in types. You can add
your own.

## Custom resources

A CustomResourceDefinition (CRD) teaches the API server a new resource kind. A
CRD is itself just a resource you apply, carrying the name and schema of the
new kind:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: backups.demo.example.com
spec:
  group: demo.example.com
  names:
    kind: Backup
    plural: backups
  scope: Namespaced
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                database:
                  type: string
                schedule:
                  type: string
```

Apply that and your cluster has a `Backup` kind. Everything you know works on
it immediately: `kubectl get backups`, `kubectl describe backup nightly`,
`kubectl apply`, `kubectl explain backup.spec`, RBAC rules from module 12,
YAML in git. The API server stores the objects and serves them like any other
resource.

But nothing happens. A CRD alone is a database table with no application
reading it. You can create `Backup` objects all day and they just sit there.

## What makes it an operator

An operator is the combination of:

- one or more CRDs, defining the new kinds, and
- a custom controller, an ordinary Deployment running in the cluster, watching
  those kinds and doing the work.

The controller runs the same reconcile loop as the built-in ones: see a
`Backup` object, check whether the backup CronJob it implies exists, create or
fix it if not. From the outside the new type behaves exactly like a native
one, which is the point.

The name is literal. An operator encodes what a human operator knows: how to
install the software, configure replication, take backups, upgrade versions,
and handle failure. Built-in controllers know generic things ("keep 3 replicas
running"). Operators know application-specific things ("this Postgres replica
is furthest ahead in replication, promote it").

## Helm charts versus operators

Module 11 might make you wonder why this is not just a chart. The difference
is day one versus day two:

- Helm templates and installs manifests, then it is done until you run it
  again. Nothing watches the cluster in between.
- An operator keeps running. It reacts to failures, drift, and changes in its
  objects continuously.

They compose rather than compete: the usual way to install an operator is with
its Helm chart. Helm delivers the controller; the controller does the ongoing
work.

## A real example: a Postgres operator

In module 06 you wrote a StatefulSet, a headless Service, and a Secret to run
one Postgres instance, and that was the easy part. Replication, failover,
backups, and version upgrades would all be on you. CloudNativePG is an
operator that owns all of it.

Install the operator (one controller per cluster, in its own namespace):

```sh
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system --create-namespace
```

Then a whole replicated database is one manifest, using the `Cluster` kind the
operator installed:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg
spec:
  instances: 3
  storage:
    size: 1Gi
```

Apply it and watch the operator work. It creates three Pods (`pg-1`, `pg-2`,
`pg-3`), initializes one as primary and the others as streaming replicas, and
creates Services split by role: `pg-rw` always routes to the primary, `pg-ro`
to the replicas. An app connects to `pg-rw` and never needs to know which Pod
is primary today.

Delete the primary Pod and the operator promotes a replica in seconds, then
rebuilds the deleted instance. That is the module 06 persistence check, but
with failover handled for you. The task walks through it.

Note what you did not write: no StatefulSet, no Services, no replication
config. Desired state went in, the operational knowledge lives in the
controller.

## Reading a cluster's extensions

Real clusters accumulate operators, so exploring an unfamiliar cluster starts
with asking the API server what it knows:

```sh
# every resource type, built-in and custom (look at the APIVERSION column)
kubectl api-resources

# just the installed CRDs
kubectl get crds

# the schema of a custom kind, same as for built-ins
kubectl explain cluster.spec --api-version=postgresql.cnpg.io/v1
```

Operators you will meet everywhere: cert-manager (its `Certificate` CRD turns
an Ingress annotation into a signed TLS certificate), Prometheus Operator
(`ServiceMonitor` objects declare what to scrape), and database operators for
Postgres, MySQL, and Kafka. Traefik, which routed your traffic since module
04, also installs CRDs; you have been running CRDs since module 01 without
noticing.

## Writing your own

Mostly you install operators; occasionally a platform team writes one. The
standard toolkit is Go with Kubebuilder (or Operator SDK, built on the same
libraries). It scaffolds the CRD and a controller whose heart is a single
`Reconcile` function: given the name of an object that changed, read it,
compare desired to actual, act, and return. The framework handles the
watching, queueing, and retries.

Write one when you operate many copies of the same system and the runbook is
repetitive (ten teams each need a caching cluster). Do not write one to deploy
an ordinary stateless app; modules 02 through 11 already covered everything
that needs.
