# Task 11 - Package the app as a Helm chart

## Goal

Turn your hand-written app manifests into a Helm chart, install it as a release,
and upgrade and roll back through Helm.

## Expected outcome

- A chart installs the `hello` Deployment and Service from templated values.
- Changing `replicaCount` through `helm upgrade` changes the running replicas.
- `helm rollback` reverts to a previous release revision.

## How to check

```sh
helm list
# a release (e.g. hello) with STATUS deployed

helm template hello ./hello-chart | head -40
# rendered manifests with your values substituted in

helm upgrade hello ./hello-chart --set replicaCount=4
kubectl get deployment hello
# READY 4/4

helm history hello
# multiple revisions listed

helm rollback hello 1
kubectl get deployment hello
# replica count matches revision 1
```

## Steps

1. Scaffold a chart with `helm create hello-chart` and read the generated
   `deployment.yaml` and `values.yaml`.
2. Adjust `values.yaml` to your app: image repository `hello-go`, tag, port
   8080, replica count. Make sure `imagePullPolicy` is `IfNotPresent` so the
   imported image is used.
3. Delete the plain `hello` Deployment and Service from earlier modules (Helm
   will manage them now) to avoid a name clash.
4. `helm install hello ./hello-chart`. Confirm the Pods and Service come up.
5. `helm upgrade --set replicaCount=4` and confirm the change. Check
   `helm history`.
6. `helm rollback` to an earlier revision and confirm the replica count follows.
7. Optional: `helm uninstall` and reinstall to see a clean create/destroy cycle.

## Hints

- `helm template` renders locally and applies nothing, so use it freely to check
  your changes before installing.
- A release name clashes if plain manifests with the same resource names still
  exist. Delete those first, or the install errors on ownership.
- If a template fails to render, the error names the file and line. `--debug`
  adds detail.
- Keep the chart in this module's directory. It is a reference you can copy and
  adapt for other apps later.
