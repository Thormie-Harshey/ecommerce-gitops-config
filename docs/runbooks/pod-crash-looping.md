# Runbook: PodCrashLooping

**Severity:** critical (posted to #all-alerts-critical)
**Fires when:** a container in staging or production restarts more than 3 times in 1 hour.

## What it means
A container keeps dying and Kubernetes keeps restarting it, waiting longer each time (CrashLoopBackOff). CrashLoopBackOff is the symptom, not the cause.

## Check
1. Find the pod: `kubectl get pods -n <namespace>`
2. See why it last stopped: `kubectl describe pod <pod> -n <namespace>`, under **Last State**, read the **Reason** (for example `OOMKilled` or `Error`) and the **Exit Code**.
3. Read the logs from the crash before the restart: `kubectl logs <pod> -n <namespace> --previous`

## Common causes and fixes
Fix through Git; ArgoCD applies the change.

- **A dependency is down at startup.** The log says `Startup failed` with `ECONNREFUSED ...:5432`. Seen on this platform on November 13, 2026: order and product restarted 2 to 3 times while Postgres was still starting, then recovered on their own. If Postgres or Redis stays down, fix that first.
- **OOMKilled** (needed more memory than its limit). In the earlier observability project (November 11, 2026), Grafana crash looped with a 256 MiB limit. Fix: raise the limit in the values file or `base/<service>/deployment.yaml`, then push.
- **Missing setting.** The services exit on purpose if a setting is missing; the log says `Missing environment variables`. Fix the `env` section in Git and push.
- **A bad image.** `git revert` the commit that changed the tag, then push (and **Sync** in ArgoCD for production).

Note: the readiness probe (`/health`) never restarts a container, and the liveness probe only checks that the port answers. So restarts come from the app exiting or being killed, not from a database outage failing a health check.

## Confirm it is fixed
The pod shows `1/1 Running` and its restart count stops rising. Slack posts **[RESOLVED] PodCrashLooping**.

## Escalate
If the cause is unclear after reading the previous logs, escalate to the application owner with the `describe` and `logs --previous` output.
