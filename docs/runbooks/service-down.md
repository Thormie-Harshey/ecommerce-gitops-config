# Runbook: ServiceDown

**Severity:** critical (posted to #all-alerts-critical)
**Fires when:** Prometheus cannot reach gateway, order or product in staging or production for 1 minute.

## What it means
A service is not answering at all, not even on /metrics. If it is the gateway, customers cannot reach anything.

## Check
1. `kubectl get pods -n <namespace> -l app=<service>` Is the pod Running and Ready?
2. In ArgoCD, is the application **Healthy**? A **Degraded** status often shows the reason, such as `ImagePullBackOff`.
3. `kubectl describe pod <pod> -n <namespace>` and read the **Events** at the bottom.
4. In Prometheus, **Status → Target health**, filter `ecommerce`: the target's error message says why the scrape failed.

## Common causes and fixes
Fix through Git; ArgoCD applies the change.

- **The image cannot be pulled** (`ImagePullBackOff`): the tag in the overlay does not exist on Docker Hub, often because CI failed before pushing. Check the latest CI run, then `git revert` the tag change.
- **The pod keeps crashing.** Follow the PodCrashLooping runbook.
- **Scaled to zero or deleted by hand.** In staging, self-heal restores it within seconds. In production, click **Sync** in ArgoCD to return it to what Git says.

## Confirm it is fixed
The target shows **UP** in Prometheus, and Slack posts **[RESOLVED] ServiceDown**.

## Escalate
Immediately, if the gateway is down in production and not restored within 5 minutes.
