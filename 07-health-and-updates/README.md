# 07 - Health checks and updates

This module makes the app resilient and teaches how Kubernetes ships new
versions with zero downtime, plus how to undo a bad release.

## Probes

Kubernetes checks Pod health with probes. There are three, and they answer
different questions.

- **Liveness probe**: is the container still working? If it fails, Kubernetes
  kills and restarts the container. Use it to recover from deadlocks.
- **Readiness probe**: is the container ready to serve traffic? If it fails, the
  Pod is removed from Service endpoints but not restarted. Use it during startup
  and when a Pod is temporarily busy.
- **Startup probe**: has a slow-starting container finished booting? It holds off
  the other probes until the app is up, so a slow start is not mistaken for a
  failure.

Your app already has a `/healthz` endpoint. Wire probes to it:

```yaml
containers:
  - name: hello
    image: hello-go:0.3
    ports:
      - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 2
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

The readiness probe is what fixes the race from module 06: a Pod only receives
traffic once it reports ready.

## Resource requests and limits

You tell Kubernetes how much CPU and memory each container needs.

- **requests**: the amount reserved for the container. The scheduler uses this to
  decide which node has room.
- **limits**: the ceiling. Exceeding the memory limit gets the container killed
  (OOMKilled); exceeding the CPU limit throttles it.

```yaml
resources:
  requests:
    cpu: "50m"        # 50 millicores = 0.05 of a CPU
    memory: "32Mi"
  limits:
    cpu: "200m"
    memory: "64Mi"
```

Requests also matter for module 08: the Horizontal Pod Autoscaler scales on CPU
usage measured against the request.

## Rolling updates

When you change a Deployment's Pod template (a new image tag, for example),
Kubernetes performs a rolling update: it brings up new Pods and takes old ones
down gradually, so the app stays available throughout. The `RollingUpdate`
strategy controls the pace:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0     # never drop below the desired count
      maxSurge: 1           # allow one extra pod during the rollout
```

Trigger and watch a rollout:

```sh
kubectl set image deployment/hello hello=hello-go:0.4
kubectl rollout status deployment/hello      # follows the rollout to completion
kubectl get pods -l app=hello -w             # watch old pods retire, new appear
```

Readiness probes make this safe: a new Pod joins the Service only once it passes
readiness, so users never hit a Pod that is still starting.

## Rollback

If a release is bad, roll back to the previous revision.

```sh
kubectl rollout history deployment/hello     # list revisions
kubectl rollout undo deployment/hello        # back to the previous one
kubectl rollout undo deployment/hello --to-revision=2
```

## Seeing a probe fail

To watch a liveness probe do its job, point it at a path the app does not serve
(say `/broken`) and apply. The container will fail the probe and restart; watch
`RESTARTS` climb in `kubectl get pods`. Then fix it back to `/healthz`.
