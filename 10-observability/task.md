# Task 10 - Inspect and debug the cluster

## Goal

Build fluency with the inspection commands by deliberately breaking things and
diagnosing them from cluster output alone.

## Expected outcome

You can, for any Pod, find its status, the events explaining it, its logs
(including after a crash), and its resource usage, and you can get a shell into
the running system even for the shell-less app image.

## How to check

This module is about skill, so the check is a set of questions you should be
able to answer using only kubectl:

```sh
# 1. Which node is each hello pod on, and how many restarts has it had?
kubectl get pods -l app=hello -o wide

# 2. Why did a specific pod land where it did, or fail to start?
kubectl describe pod <pod>        # read the Events section

# 3. What did the app log, and what did it log before its last crash?
kubectl logs <pod>
kubectl logs <pod> --previous

# 4. How much CPU and memory is each pod using right now?
kubectl top pods

# 5. Get a shell into the running app pod even though its image has no shell.
kubectl debug -it <pod> --image=busybox --target=hello
```

## Steps

1. Break a Deployment on purpose: set the image to a tag that does not exist and
   apply. Use `get`, then `describe`, and find the `ImagePullBackOff` event.
   Fix it.
2. Break it another way: set the liveness probe to a wrong path so the container
   restarts. Read `logs --previous` and the restart count. Fix it.
3. Use `kubectl get events --sort-by=.lastTimestamp` to see the whole story of
   what just happened, in order.
4. Use `kubectl debug` to open a busybox shell targeting the distroless app Pod,
   and from inside it curl `localhost:8080`.
5. Answer all five questions in the check section without looking anything up
   except with kubectl.

## Hints

- The Events section of `describe` is where most answers live. Read it first,
  every time.
- `--previous` only works if the container actually restarted. On a healthy Pod
  it errors, which is expected.
- `kubectl debug` needs `--target` to share the app container's namespaces; omit
  it and you get an isolated container that cannot see the app's localhost.
- Keep a scratch terminal running `kubectl get pods -w` while you break things.
  Watching state change in real time builds intuition faster than polling.
