# 12 - Namespaces and RBAC

Everything so far lived in the `default` namespace with full admin access. Real
clusters divide work into namespaces and restrict who can do what. This module
covers both.

## Namespaces

A namespace is a partition of the cluster. Resource names must be unique within
a namespace, not across the whole cluster, so two teams can each have a `hello`
Deployment without colliding. Namespaces are also the unit for quotas and access
rules.

```sh
kubectl create namespace demo
kubectl get namespaces
kubectl apply -f deployment.yaml -n demo     # create in a specific namespace
kubectl get pods -n demo
kubectl get pods --all-namespaces            # everything, everywhere
```

Set a default namespace for the current context so you stop typing `-n`:

```sh
kubectl config set-context --current --namespace=demo
```

Cross-namespace DNS uses the full name: a Service `postgres` in namespace `data`
is reachable as `postgres.data.svc.cluster.local` from other namespaces.

## Resource quotas and limits

A namespace can cap total resource use with a ResourceQuota, and set default
container limits with a LimitRange. These stop one workload from starving the
cluster and are how multi-tenant clusters stay fair. Worth knowing they exist;
you can experiment by applying a small ResourceQuota to your `demo` namespace and
watching a Deployment get rejected for exceeding it.

## RBAC: who can do what

RBAC (role-based access control) governs which identities may perform which
actions on which resources. Four objects combine to express it:

- **Role**: a set of permissions (verbs like `get`, `list`, `create` on
  resources like `pods`) within one namespace.
- **ClusterRole**: the same, but cluster-wide or for cluster-scoped resources.
- **RoleBinding**: grants a Role to a subject within a namespace.
- **ClusterRoleBinding**: grants a ClusterRole cluster-wide.

A subject is a user, a group, or a ServiceAccount.

## ServiceAccounts

A ServiceAccount is an identity for a process running in a Pod (as opposed to a
human user). Every Pod runs as one; if you do not specify it, the namespace's
`default` ServiceAccount is used. When a Pod needs to talk to the Kubernetes API
(to list Pods, for example), it authenticates as its ServiceAccount, and RBAC
decides what it may do.

```sh
kubectl create serviceaccount app-reader -n demo
```

## A worked example: a read-only role

Grant a ServiceAccount permission to read Pods in `demo`, nothing more.

`rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: demo
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: demo
  name: read-pods
subjects:
  - kind: ServiceAccount
    name: app-reader
    namespace: demo
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

## Testing permissions

`kubectl auth can-i` checks whether a subject may do something, without actually
doing it.

```sh
kubectl auth can-i list pods -n demo \
  --as=system:serviceaccount:demo:app-reader
# yes

kubectl auth can-i delete pods -n demo \
  --as=system:serviceaccount:demo:app-reader
# no
```

This is the fastest way to confirm an RBAC rule does what you intended.
