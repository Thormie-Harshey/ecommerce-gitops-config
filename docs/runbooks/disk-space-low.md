# Runbook: DiskSpaceLow

**Severity:** warning (posted to #alerts-warning)
**Fires when:** less than 10% of the node's /var disk is free, for 5 minutes.

## What it means
/var holds container images, container logs, Postgres data and Loki data. When it fills, logs stop being written, and Kubernetes starts removing pods (eviction).

## Check
1. Open **Infrastructure (Node)**, panel **Disk free on /var**. How fast is it falling? (The flat line at 10% is the alert threshold, not a second disk.)
2. Confirm from the node itself: `docker exec gitops-control-plane df -h /var`
3. See what is using space:
   `docker exec gitops-control-plane du -sh /var/lib/containerd /var/log/pods /var/local-path-provisioner`

## Common causes and fixes
- **Old container images.** `docker exec gitops-control-plane crictl rmi --prune`
- **Large container logs.** Find the noisy pod in **Application Logs** (log volume panel) and reduce its logging.
- **Loki data** (2 GiB volume) or **Postgres data** (1 GiB per environment). Increase the volume size or set retention in `monitoring/loki-values.yaml`, through Git. Jaeger keeps traces in memory, so it uses no disk.
- **The host disk is full.** /var lives on WSL's virtual disk, stored on the Windows C: drive. Check free space on C:.

## Confirm it is fixed
The disk panel rises back above 10%, and Slack posts **[RESOLVED] DiskSpaceLow**.

## Escalate
If free space keeps falling after cleanup, escalate before it reaches 5%.
