# Runbook: HighErrorRate

**Severity:** critical (posted to #all-alerts-critical)
**Fires when:** more than 5% of a service's requests return a 5xx error for 5 minutes, in staging or production. The alert names the namespace and service.

## What it means
Customers are getting failures. 5xx means the server failed, not the customer (4xx errors such as 404 are not counted).

## Check
1. Open the **Service Overview (RED)** dashboard and set **Environment** to the alert's namespace. Note which services are erroring. If gateway and order fail but product does not, suspect what order depends on.
2. Check for a recent change, the most common cause: in ArgoCD, open the application's **History**, or run `git log --oneline -5` in the config repo.
3. Check the pods: `kubectl get pods -n <namespace>`
4. Read the errors in **Application Logs**, or in Grafana Explore with Loki: `{namespace="<namespace>"} | json | level >= 50`
5. Open a failing trace in Jaeger (service `gateway`, tag `error=true`), or copy a `trace_id` from an error log line into **Lookup by Trace ID**. The failing span shows which service broke.

## Common causes and fixes
Fix through Git, not `kubectl`. In staging, self-heal undoes manual changes within seconds; in production, they cause drift from Git.

- **A bad config or release commit.** Revert it in the config repo: `git revert <commit>` then `git push`. Staging syncs automatically; for production, click **Sync** in ArgoCD.
  Example, November 13, 2026 (alert drill): a commit pointed staging's order service at a missing product address. Order failed 50% of requests and gateway 24%. `git revert` resolved the alert within about 10 minutes.
- **A dependency is down** (product, Postgres or Redis). Check `kubectl get pods -n <namespace>` and `kubectl logs deploy/<name> -n <namespace> --tail=50`. Fix the cause in Git rather than restarting by hand.
- **Database unreachable.** Check `kubectl get pods -n <namespace> -l app=postgres`. Order and product log `ECONNREFUSED ...:5432` when Postgres is not answering.

## Confirm it is fixed
The error rate panel drops below 5%, and Slack posts **[RESOLVED] HighErrorRate**.

## Escalate
If not resolved within 15 minutes, escalate to the application owner.
