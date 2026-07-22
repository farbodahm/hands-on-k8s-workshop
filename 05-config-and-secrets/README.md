# 05 - Config and secrets

Applications need configuration, and some of it is sensitive. Kubernetes
separates config from the image with two resources: ConfigMap for plain values
and Secret for sensitive ones. Keeping config out of the image means you ship
one image and configure it per environment.

## ConfigMap

A ConfigMap holds non-sensitive key-value data.

`configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-config
data:
  GREETING: "hello from config"
  PORT: "8080"
```

## Two ways to consume config

You inject config into a Pod either as environment variables or as files
mounted into the container.

As environment variables:

```yaml
spec:
  containers:
    - name: hello
      image: hello-go:0.1
      envFrom:
        - configMapRef:
            name: hello-config
```

`envFrom` turns every key in the ConfigMap into an environment variable. To pull
a single key instead, use `env` with `valueFrom`:

```yaml
env:
  - name: PORT
    valueFrom:
      configMapKeyRef:
        name: hello-config
        key: PORT
```

As a mounted file, the ConfigMap keys become filenames under a directory:

```yaml
volumes:
  - name: config
    configMap:
      name: hello-config
containers:
  - name: hello
    volumeMounts:
      - name: config
        mountPath: /etc/hello
```

Files suit larger config (a full config file); env vars suit simple values.

## Secret

A Secret works like a ConfigMap but is meant for sensitive data such as
passwords and API keys. Values are base64-encoded in the stored object. Note
that base64 is encoding, not encryption; it keeps values out of plain sight but
anyone with read access to the Secret can decode them. Treat access to Secrets
as access to the data.

Create a Secret from the command line so you never write the plaintext into a
file:

```sh
kubectl create secret generic db-credentials \
  --from-literal=POSTGRES_USER=app \
  --from-literal=POSTGRES_PASSWORD=s3cret
```

Consume it the same way as a ConfigMap, with `secretRef` or `secretKeyRef`:

```yaml
envFrom:
  - secretRef:
      name: db-credentials
```

## Inspecting

```sh
kubectl get configmap hello-config -o yaml
kubectl get secret db-credentials -o yaml     # values are base64
kubectl get secret db-credentials -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d
```

## Updating a running app

Editing a ConfigMap does not restart Pods. Env vars are read at container start,
so a Pod keeps its old values until it restarts. Mounted files do update in
place after a short delay, but most apps read files at startup too. To pick up
changes reliably, roll the Deployment:

```sh
kubectl rollout restart deployment hello
```

This ties into module 07's rollout mechanics.
