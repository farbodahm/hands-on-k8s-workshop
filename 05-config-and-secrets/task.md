# Task 05 - Configure the app without rebuilding

## Goal

Drive the app's behavior from a ConfigMap and a Secret instead of hardcoded
values, without changing the image.

## Expected outcome

- A ConfigMap sets the app's `PORT` (and optionally a greeting).
- A Secret holds a username and password.
- The running Pod's environment contains the values from both.

## How to check

```sh
kubectl get configmap hello-config
kubectl get secret db-credentials

# confirm the values reached the container:
POD=$(kubectl get pod -l app=hello -o jsonpath='{.items[0].metadata.name}')
kubectl exec "$POD" -- env | grep -E 'PORT|POSTGRES_USER'
# PORT and POSTGRES_USER appear with the configured values

# decode a secret value:
kubectl get secret db-credentials -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```

## Steps

1. Write and apply `configmap.yaml` with at least a `PORT` key.
2. Create `db-credentials` as a Secret with `kubectl create secret generic`
   (see README) so the plaintext never lands in a file.
3. Edit the Deployment's container spec to pull env vars from both the ConfigMap
   and the Secret with `envFrom`. Apply it.
4. `kubectl rollout restart deployment hello` so new Pods pick up the config.
5. Exec into a Pod and confirm the env vars are present.
6. Change a ConfigMap value, apply, exec into the Pod, and observe it is
   unchanged. Then rollout restart and confirm it updates. This shows why the
   restart matters.

## Hints

- If the app fails to start after wiring in `PORT`, check the value is a string
  in quotes. YAML would otherwise read `8080` as a number and the env var
  injection can complain.
- `kubectl exec <pod> -- env` is the ground-truth check for what the container
  actually sees. Trust it over reading the manifest.
- Never commit real secrets to a repo. Creating them with `kubectl create
  secret` from literals (or from a file you gitignore) keeps them out of source.
- You reuse the `db-credentials` Secret in module 06 for Postgres, so pick
  values you will remember.
