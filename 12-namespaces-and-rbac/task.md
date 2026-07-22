# Task 12 - Isolate with namespaces and lock down with RBAC

## Goal

Deploy into a dedicated namespace, then create a ServiceAccount with a Role that
allows reading Pods but not deleting them, and prove the boundary holds.

## Expected outcome

- A `demo` namespace contains a copy of the app.
- A `pod-reader` Role and RoleBinding grant a ServiceAccount read-only access to
  Pods in `demo`.
- `kubectl auth can-i` confirms the ServiceAccount can list Pods but cannot
  delete them.

## How to check

```sh
kubectl get namespace demo
kubectl get pods -n demo                 # the app runs here

kubectl auth can-i list pods -n demo \
  --as=system:serviceaccount:demo:app-reader
# yes

kubectl auth can-i delete pods -n demo \
  --as=system:serviceaccount:demo:app-reader
# no

kubectl auth can-i list pods -n default \
  --as=system:serviceaccount:demo:app-reader
# no  (the Role is scoped to demo only)
```

## Steps

1. Create the `demo` namespace and deploy the app into it (reuse a manifest with
   `-n demo`, or your Helm chart with `--namespace demo --create-namespace`).
2. Create the `app-reader` ServiceAccount in `demo`.
3. Write and apply `rbac.yaml` with a `pod-reader` Role and a RoleBinding tying
   the Role to the ServiceAccount.
4. Use `kubectl auth can-i --as=...` to confirm: can list Pods, cannot delete
   Pods, cannot list Pods in `default`.
5. Optional: apply a small ResourceQuota to `demo`, then try to scale the app
   beyond it and read the rejection message.

## Hints

- The `--as=system:serviceaccount:<namespace>:<name>` form is how you test as a
  ServiceAccount. The pattern is fixed; get the namespace and name right.
- A RoleBinding only grants access in its own namespace. That is why the `demo`
  Role gives nothing in `default`, which is the isolation you are demonstrating.
- If `can-i` says `yes` when you expected `no`, you may still be testing as
  yourself (a cluster admin). The `--as` flag is what switches identity.
- For cluster-wide read access you would use a ClusterRole and
  ClusterRoleBinding instead. Try converting the example as an extra exercise.
