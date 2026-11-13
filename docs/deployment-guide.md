# Deployment guide

How to build the platform from nothing on a Windows laptop with WSL.
Typical time: 20 to 30 minutes, mostly image downloads.

## 1. Prerequisites

| Tool | Version used |
|---|---|
| Windows with WSL 2 (Ubuntu) and Docker Desktop | Docker 29.4 |
| kind | 0.31.0 |
| kubectl | 1.35 |
| Helm | 3.20 |
| Ansible (core) | 2.16 |
| kubeseal | 0.41.2 (must match the controller) |
| Node.js (only to run tests locally) | 24 |

Give WSL enough memory. Create `C:\Users\<you>\.wslconfig`:

    [wsl2]
    memory=10GB

Then run `wsl --shutdown` in PowerShell and reopen Ubuntu. The full stack uses about 4 to 5 GB.

## 2. Build the cluster

    git clone https://github.com/Thormie-Harshey/ecommerce-gitops-config.git
    cd ecommerce-gitops-config
    ansible-playbook -i ansible/inventory.ini ansible/bootstrap.yml

The playbook creates the kind cluster `gitops`, the namespaces, ArgoCD and
the Sealed Secrets controller, then applies the root app. ArgoCD deploys
everything else from this repository.

## 3. Reseal the secrets (new cluster only)

A new cluster has a new Sealed Secrets key, so the sealed files in Git
cannot be decrypted until they are resealed with it.

    # Postgres password, one per environment
    for env in staging production; do
      kubectl create secret generic postgres-credentials --namespace "$env" \
        --from-literal=password="$(openssl rand -hex 24)" --dry-run=client -o yaml \
      | kubeseal --format yaml > "overlays/$env/postgres-sealedsecret.yaml"
    done

    # Grafana login
    kubectl create secret generic grafana-admin --namespace monitoring \
      --from-literal=admin-user=admin \
      --from-literal=admin-password="$(openssl rand -hex 16)" \
      --dry-run=client -o yaml \
    | kubeseal --format yaml > monitoring/secrets/grafana-admin-sealedsecret.yaml

    # Slack webhooks (pasted input is hidden)
    read -rsp "Critical webhook URL: " CRIT; echo
    read -rsp "Warning webhook URL: " WARN; echo
    kubectl create secret generic alertmanager-slack --namespace monitoring \
      --from-literal=critical-url="$CRIT" --from-literal=warning-url="$WARN" \
      --dry-run=client -o yaml \
    | kubeseal --format yaml > monitoring/secrets/alertmanager-slack-sealedsecret.yaml
    unset CRIT WARN

    git add . && git commit -m "Reseal secrets for new cluster" && git push

Postgres starts with whichever password it first receives, so reseal before
first use, or delete the Postgres volumes after resealing.

## 4. Check and promote

    kubectl get applications -n argocd

All applications should become Synced and Healthy within about 10 minutes,
except `ecommerce-production`, which waits as OutOfSync. Promote it by
clicking **Sync** in ArgoCD.

## 5. Open the tools

Each command opens a tunnel; leave it running in its own terminal.

| Tool | Command | Address | Login |
|---|---|---|---|
| App (staging) | `kubectl port-forward -n staging svc/gateway 8080:3000` | http://localhost:8080 | none |
| ArgoCD | `kubectl port-forward -n argocd svc/argocd-server 8443:443` | https://localhost:8443 | `admin`, password below |
| Grafana | `kubectl port-forward -n monitoring svc/kps-grafana 3001:80` | http://localhost:3001 | `admin`, password below |
| Prometheus | `kubectl port-forward -n monitoring svc/kps-kube-prometheus-stack-prometheus 9090:9090` | http://localhost:9090 | none |
| Jaeger | `kubectl port-forward -n monitoring svc/jaeger 16686:16686` | http://localhost:16686 | none |

    # ArgoCD password
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    # Grafana password
    kubectl get secret grafana-admin -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d; echo

## 6. CI setup (app repo, once)

In `ecommerce-gitops-app`, under Settings, Secrets and variables, Actions:

| Name | Type | Value |
|---|---|---|
| `DOCKERHUB_USERNAME` | Variable | Docker Hub username |
| `DOCKERHUB_TOKEN` | Secret | Docker Hub access token, Read and Write, 90-day expiry |
| `CONFIG_REPO_TOKEN` | Secret | Fine-grained GitHub token, Contents read and write on the config repo only, 90-day expiry |

## 7. Release, promote and roll back

- **Release:** push to `main` in the app repo. CI tests, scans and pushes the
  images, then writes the new tag into `overlays/staging`. ArgoCD deploys it.
- **Promote:** copy the tag into `overlays/production/kustomization.yaml`,
  push, then click **Sync** on `ecommerce-production`.
- **Roll back:** `git revert <commit>` in the config repo and push
  (and Sync, for production).

Always run `git pull` in the config repo before editing it, because CI also
commits to it.

## 8. Tear down

    kind delete cluster --name gitops

Then `wsl --shutdown` in PowerShell to release the memory.
