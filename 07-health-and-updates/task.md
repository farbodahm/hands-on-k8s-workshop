# Task 07 - Make the app resilient and update it safely

## Goal

Add probes and resource limits, perform a rolling update with no downtime, then
roll back.

## Expected outcome

- The Deployment has readiness and liveness probes and resource requests/limits.
- A rolling update to a new image tag completes while the app stays reachable.
- A rollback returns the app to the previous version.

## How to check

```sh
kubectl describe deployment hello | grep -A3 -E 'Liveness|Readiness|Limits|Requests'
# probes and resources are listed

# rollout with no downtime: run this curl loop in one terminal...
while true; do curl -s localhost:8080 || echo FAIL; sleep 0.3; done
# ...then trigger the update in another and confirm no FAIL lines appear:
kubectl set image deployment/hello hello=hello-go:0.4
kubectl rollout status deployment/hello        # ends with "successfully rolled out"

kubectl rollout history deployment/hello        # shows multiple revisions
kubectl rollout undo deployment/hello
kubectl rollout status deployment/hello
```

## Steps

1. Add readiness and liveness probes pointing at `/healthz`, plus CPU and memory
   requests and limits, to the Deployment. Apply.
2. Set the `RollingUpdate` strategy with `maxUnavailable: 0`.
3. Build a visibly different image (change the greeting), tag it `hello-go:0.4`,
   import it.
4. Start a curl loop through a port-forward, trigger the image update, and watch
   the rollout finish without a failed request.
5. Inspect `rollout history`, then `rollout undo` and confirm the old version
   returns.
6. Break the liveness probe path on purpose, apply, and watch `RESTARTS`
   increase in `kubectl get pods`. Restore it afterward.

## Hints

- If the rollout drops requests, check `maxUnavailable` is 0 and the readiness
  probe is actually passing. A new Pod that never becomes ready stalls the
  rollout, which is the strategy protecting you.
- `kubectl rollout status` blocks until the rollout finishes or fails, which
  makes it useful in scripts.
- `initialDelaySeconds` too low makes a slow-starting app restart in a loop. If
  you see steady restarts right after applying probes, raise the delay or add a
  startup probe.
- Set resource requests low (`50m` CPU, `32Mi` memory). This is a laptop; you do
  not want the scheduler unable to place Pods.
