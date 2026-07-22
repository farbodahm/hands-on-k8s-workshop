# Task 09 - Run one-off and scheduled work

## Goal

Run a Job that performs a database migration once, then a CronJob that fires
repeatedly on a schedule.

## Expected outcome

- A Job completes successfully and its effect is visible (a table exists in
  Postgres).
- A CronJob creates a new Job every minute, each producing log output.

## How to check

```sh
kubectl get job migrate
# COMPLETIONS 1/1

kubectl exec -it postgres-0 -- psql -U app -d appdb -c "\dt"
# the table the Job created is listed

kubectl get cronjob heartbeat
# LAST SCHEDULE shows a recent time

kubectl get jobs
# a new heartbeat-xxxx job appears roughly every minute

kubectl logs job/<one-heartbeat-job-name>
# prints the date and "tick"
```

## Steps

1. Write and apply `job.yaml` that runs a `psql` command to create a table in
   the Postgres from module 06. Watch it reach `COMPLETIONS 1/1`.
2. Read the Job's logs and confirm the table exists via `psql`.
3. Write and apply `cronjob.yaml` scheduled every minute.
4. Watch `kubectl get jobs` for a couple of minutes and confirm new Jobs appear.
5. Read one Job's logs. Then delete the CronJob so it stops firing.
6. Optional: set `concurrencyPolicy: Forbid` and a longer-running command to see
   overlapping runs get skipped.

## Hints

- A Job Pod stuck in `CrashLoopBackOff` is not possible; Jobs use
  `restartPolicy: Never` or `OnFailure`. A failing Job shows retries counting
  toward `backoffLimit`, then a `Failed` status. Read the Pod logs to find why.
- If the migration Job cannot reach Postgres, check the `postgres` Service has
  an endpoint and the credentials Secret name matches.
- CronJob times are UTC by default. Do not be surprised if `LAST SCHEDULE` looks
  offset from your local clock.
- Completed Jobs and their Pods linger on purpose so you can read logs. Clean up
  with `ttlSecondsAfterFinished` or delete them by hand.
