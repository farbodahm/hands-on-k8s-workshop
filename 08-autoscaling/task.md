# Task 08 - Autoscale on CPU

## Goal

Create an HPA for the app, generate load, and watch it add and remove replicas
automatically.

## Expected outcome

- `kubectl top pods` reports usage (metrics-server works).
- An HPA targets the `hello` Deployment between 1 and 5 replicas.
- Under load, replicas increase; after load stops, they decrease.

## How to check

```sh
kubectl top pods              # prints CPU/memory per pod, no error

kubectl get hpa hello
# shows TARGETS (e.g. 5%/50%), MINPODS 1, MAXPODS 5, current REPLICAS

# with load running (see steps), in another terminal:
kubectl get hpa hello -w
# TARGETS rises above 50% and REPLICAS climbs toward 5

# after stopping load, within a few minutes:
kubectl get deployment hello
# READY drops back toward 1
```

## Steps

1. Confirm `kubectl top pods` works. If not, wait for metrics-server or check
   its Pod in `kube-system`.
2. Confirm the Deployment has CPU requests (from module 07). Without them the
   HPA cannot compute a percentage.
3. Create the HPA (command or manifest) targeting 50% CPU, min 1, max 5.
4. Start one or more load generators (see README). Watch the HPA scale up.
5. Stop the load. Watch replicas scale back down after the cooldown.
6. Run `kubectl describe hpa hello` and read the events section to see the
   scaling decisions it recorded.

## Hints

- HPA target shows `<unknown>` when it cannot read metrics. Two usual causes:
  metrics-server not ready, or missing CPU requests on the container.
- Scale-up is quick; scale-down is deliberately slow to avoid flapping. Give it
  a few minutes after load stops before expecting fewer replicas.
- If the cheap `/` handler will not push CPU high enough, add a CPU-burning
  endpoint to the app (a loop that hashes for a bit) and hit that instead. This
  is a realistic reason apps expose load-test endpoints.
- The HPA respects the Deployment's own replica field loosely; once an HPA owns
  a Deployment, manage scale through the HPA, not `kubectl scale`.
