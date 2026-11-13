# Capstone 2: GitOps Platform with Monitoring

App code and CI: https://github.com/Thormie-Harshey/ecommerce-gitops-app

## How it works

1. A developer pushes code to the app repo on GitHub.
2. The push starts CI in GitHub Actions: lint, test, security scan,
   build, push. If any stage fails, the pipeline stops there.
3. If all stages pass, CI uploads the container image to Docker Hub
   with a version tag, then updates that tag in this config repo.
4. ArgoCD runs inside the cluster (argocd namespace). It compares this
   repo with what is running. When they differ, it updates the cluster
   to match Git.
5. The kubelet on the node pulls the new image from Docker Hub and
   starts the updated pods.
6. The monitoring namespace collects metrics, logs, traces and alerts
   from everything.

ArgoCD pulls changes from Git; CI never touches the cluster.
Nobody deploys by hand. Git is the single source of truth.

## Design decisions

| Decision | Choice | Reason |
|---|---|---|
| Repositories | Two: app repo and config repo | Separates code from deployment settings, matches the brief, and avoids CI triggering itself |
| Registry | Docker Hub | Named in the brief; existing account |
| Cluster | kind on a local laptop | No cloud cost or public exposure; the Ansible playbook can target a cloud server by changing the inventory |
| Namespaces | staging, production, monitoring, argocd | Isolates environments and tools from each other |
| Services | 3 (gateway, order, product) | Meets the 3+ requirement whilst fitting a 16 GB laptop |
| Deployment model | Pull-based (ArgoCD) | CI holds no cluster credentials, reducing attack surface |
| Bootstrapping | Ansible installs only ArgoCD and Sealed Secrets; ArgoCD deploys everything else (app of apps) | Keeps the whole platform in Git |
| Sync policy | Staging: auto-sync, self-heal, prune. Production: manual sync | In ArgoCD, self-heal requires auto-sync, so production relies on OutOfSync detection plus human approval |
| Secrets | Sealed Secrets, one random password per environment | Encrypted secrets are safe in a public repo; only the cluster can decrypt them |
| Data stores | Postgres as a StatefulSet with a 1 GB volume; Redis without a volume | Postgres holds real data; Redis is a disposable cache |

## Bootstrapping the platform (Ansible)

One command builds the platform on a laptop:

    ansible-playbook -i ansible/inventory.ini ansible/bootstrap.yml

The playbook checks the required tools, creates a single-node kind
cluster, creates the namespaces, and installs ArgoCD (chart 10.18.0)
and Sealed Secrets (chart 2.22.0). ArgoCD then deploys everything else
from this repository.

The playbook is idempotent: a second run reports changed=0.
Prerequisite on Windows: `.wslconfig` raises WSL memory to 10 GB.
