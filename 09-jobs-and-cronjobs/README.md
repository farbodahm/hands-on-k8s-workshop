# 09 - Jobs and CronJobs

Not everything runs forever. Some work runs once and finishes (a database
migration, a batch import), and some runs on a schedule (a nightly cleanup). A
Deployment is wrong for these because it restarts Pods that exit. Jobs and
CronJobs are built for work that completes.

## Job

A Job runs one or more Pods until a set number complete successfully, then stops.
If a Pod fails, the Job retries it up to a limit.

`job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migrate
spec:
  backoffLimit: 3          # retries before the Job is marked failed
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: postgres:16
          envFrom:
            - secretRef:
                name: db-credentials
          command:
            - sh
            - -c
            - >
              psql -h postgres -U app -d appdb
              -c "CREATE TABLE IF NOT EXISTS visits (id serial primary key, at timestamptz default now());"
```

`restartPolicy: Never` (or `OnFailure`) is required for Jobs; a Job Pod must be
allowed to terminate.

```sh
kubectl apply -f job.yaml
kubectl get jobs                 # COMPLETIONS goes to 1/1
kubectl logs job/migrate         # the command output
```

This Job creates a table your app could use, a realistic migration task tied to
the module 06 database.

## Key Job fields

- **completions**: how many successful Pods make the Job done (default 1).
- **parallelism**: how many Pods run at once.
- **backoffLimit**: how many failures before the Job gives up.
- **activeDeadlineSeconds**: a hard time limit for the whole Job.

Completed Jobs stick around so you can read their logs. Delete them with
`kubectl delete job migrate`, or set `ttlSecondsAfterFinished` to clean up
automatically.

## CronJob

A CronJob creates a Job on a schedule, using cron syntax.

`cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: heartbeat
spec:
  schedule: "*/1 * * * *"        # every minute
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: heartbeat
              image: busybox
              command: ["sh", "-c", "date; echo tick"]
```

The five schedule fields are minute, hour, day-of-month, month, day-of-week.
`*/1 * * * *` runs every minute, which is convenient for seeing it fire without
waiting.

```sh
kubectl apply -f cronjob.yaml
kubectl get cronjob heartbeat        # shows LAST SCHEDULE
kubectl get jobs -w                  # a new job appears each minute
```

## Managing CronJobs

- **suspend**: set `spec.suspend: true` to pause scheduling without deleting.
- **concurrencyPolicy**: `Allow`, `Forbid` (skip if the previous run is still
  going), or `Replace`.
- **successfulJobsHistoryLimit** and **failedJobsHistoryLimit**: how many old
  Jobs to keep for inspection.

Stop a noisy CronJob when you are done experimenting:

```sh
kubectl delete cronjob heartbeat
```
