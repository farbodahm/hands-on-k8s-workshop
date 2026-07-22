# Task 13 - Clean up and self-assess

## Goal

Tear down what you built, confirm the cluster is clean or gone, and check your
own understanding against the vocabulary and commands from the whole path.

## Expected outcome

- All workloads you created are removed (or the cluster is deleted).
- You can explain each core term and pick the right command for common tasks
  without looking them up.

## How to check

```sh
# if you kept the cluster, it should be empty of your work:
kubectl get all --all-namespaces | grep -E 'hello|postgres|heartbeat|migrate'
# no output

# or, if you deleted the cluster:
k3d cluster list
# learn is gone
```

## Steps

1. Uninstall the Helm release and delete any remaining manifests and the `demo`
   namespace.
2. Decide whether to keep the cluster as a sandbox (`k3d cluster stop learn`) or
   remove it (`k3d cluster delete learn`).
3. Self-assess with the questions below. If any answer is fuzzy, reread that
   module's README and redo its task.

## Self-assessment questions

Answer these out loud or in writing. They cover the whole path.

- What is the difference between a Pod and a Deployment, and why rarely create a
  bare Pod?
- A Service has no endpoints. What are the two most likely causes?
- A client on your Mac cannot reach the app. Walk the request path (Ingress,
  Service, Pod) and name what to check at each hop.
- When does a ConfigMap change take effect in a running Pod, and how do you force
  it?
- Why does a database use a StatefulSet and a PVC instead of a plain Deployment?
- What is the difference between a liveness and a readiness probe, and what does
  each do on failure?
- The HPA shows a target of `<unknown>`. Name two causes.
- Which command tells you why a Pod will not start, and which section of its
  output do you read first?
- What does `helm rollback` operate on that `kubectl rollout undo` does not?
- A ServiceAccount can do more than you intended. Which command confirms its
  permissions without performing any action?

## Hints

- If you cannot answer a question quickly, that module is where to spend more
  time. The tasks are repeatable; run them again on the fresh cluster.
- Keeping the cluster stopped rather than deleted lets you jump back in without
  rebuilding images, at the cost of some disk. Deleting is the truly clean slate.
