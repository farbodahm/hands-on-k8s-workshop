# 13 - Cleanup and next steps

You built an application and grew it into a small system: replicated, routed,
configured, backed by a database, health-checked, autoscaled, scheduled,
observable, packaged, and access-controlled. This module tears it down cleanly
and points at what to study next.

## Tearing down

Remove workloads first, then the cluster.

```sh
# by manifest, if you kept the files:
kubectl delete -f <file>.yaml

# by Helm release:
helm uninstall hello

# a whole namespace and everything in it:
kubectl delete namespace demo

# stop the cluster but keep it for later:
k3d cluster stop learn

# delete the cluster entirely:
k3d cluster delete learn
```

Deleting the cluster reclaims all Docker resources. Images you imported are gone
with it; you rebuild and re-import when you next need them.

## What you now know

The vocabulary you should be comfortable with: cluster, node, control plane, Pod,
Deployment, ReplicaSet, Service (ClusterIP, NodePort, LoadBalancer), Ingress and
ingress controller, ConfigMap, Secret, PersistentVolume, PersistentVolumeClaim,
StorageClass, StatefulSet, probe (liveness, readiness, startup), resource
requests and limits, rolling update and rollback, Horizontal Pod Autoscaler, Job,
CronJob, namespace, ServiceAccount, Role, RoleBinding, and RBAC.

The commands you should reach for without thinking: `kubectl get`, `describe`,
`logs`, `apply`, `delete`, `exec`, `port-forward`, `rollout`, `scale`, `top`,
`auth can-i`, plus the k3d and helm lifecycle commands.

## Where to go next

A few directions, roughly in order of usefulness for most people:

- **Declarative everything and GitOps.** Keep all manifests in a Git repo and
  have a tool (Argo CD or Flux) apply them automatically. This is how teams run
  clusters in practice.
- **Networking depth.** Study `NetworkPolicy` to restrict Pod-to-Pod traffic, and
  read how the cluster DNS and Service proxy actually route packets.
- **Stateful workloads for real.** Run a multi-replica database with an operator,
  and learn how backups and failover work under a StatefulSet.
- **Observability stack.** Install Prometheus for metrics and Grafana for
  dashboards, and add structured logging aggregation. `kubectl top` is the floor,
  not the ceiling.
- **Security.** Pod security standards, image scanning, and tighter RBAC. Revisit
  Secrets and look at encryption at rest and external secret stores.
- **A managed cluster.** Deploy the same app to a cloud provider's Kubernetes
  (GKE, EKS, or AKS). The manifests barely change, which is the payoff of
  learning on K3s: the skills transfer directly.

## Keep the sandbox

The K3s cluster is cheap to recreate, so treat it as a scratchpad. When you read
about a new resource type, write a manifest and apply it here. Breaking things in
a local cluster is the fastest way to understand them, and nothing here costs
money or affects anyone else.
