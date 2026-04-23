# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal DevOps lab that deploys a portfolio site (https://shengjunye.me) via a full GitOps pipeline on AWS. The goal is repeated destroy/rebuild cycles to build operational muscle memory — not a production system.

Each subdirectory is its own git repository with its own remote on GitHub (`Illumidragui/`).

## Repository map

| Directory | Purpose | Own CLAUDE.md |
|---|---|---|
| `devsecops-infra/` | Terraform — provisions AWS + deploys ArgoCD | Yes |
| `devsecops-helm/` | Helm charts — source of truth for ArgoCD apps | No |
| `argocd-app-of-apps/` | ArgoCD App of Apps — points ArgoCD at `devsecops-helm` | No |
| `devsecops-v5/` | Docusaurus portfolio site — built into a Docker image | Yes |

## End-to-end architecture

```
devsecops-v5/          devsecops-infra/
(Docusaurus site)  ──► (Terraform: VPC + EC2 + ArgoCD)
        │                        │
  Docker image                   │ ArgoCD bootstrapped via Helm
  docker.io/                     ▼
  illumidragui/        argocd-app-of-apps/
  syesite:<tag>    ◄── (ArgoCD watches this repo)
                                 │
                    devsecops-helm/ (Helm charts)
                    ├── cert-manager-clusterissuer
                    ├── ingress-nginx
                    ├── tailscale-operator
                    └── syesite-chart  ← references Docker image tag
```

**Change propagation for a site update:**
1. Edit content in `devsecops-v5/`, push to GitHub
2. CI builds a new Docker image tagged `ga-<date>-<run>` and pushes to Docker Hub
3. Update `devsecops-helm/syesite-chart/values.yaml` with the new image tag
4. ArgoCD detects the Helm chart change and rolls out the new pod automatically

## Key architectural constraints

**Two-phase Terraform deployment** — the `helm-argocd` module cannot be applied in the same run as `vpc` + `ec2` because the Helm/Kubernetes providers need a kubeconfig that only exists after k3s installs itself. The `create = bool` flag on each module is the sequencing mechanism (not Terraform workspaces):

```bash
# Phase 1: set helm-argocd create = false in main.tf
terraform apply -target=module.vpc -target=module.ec2 -var-file=terraform.tfvars

# Copy kubeconfig once k3s is up (get Tailscale IP from admin panel first):
TAILSCALE_IP=<ip>
scp ec2-user@${TAILSCALE_IP}:/etc/rancher/k3s/k3s.yaml ~/.kube/devsecops-config
sed -i "s/127.0.0.1/${TAILSCALE_IP}/g" ~/.kube/devsecops-config

# Phase 2: set helm-argocd create = true in main.tf
terraform apply -var-file=terraform.tfvars
```

**Helm/Kubernetes providers are scoped inside `helm-argocd` module** — `depends_on` cannot be used on that module (Terraform limitation with provider-containing modules); ordering is handled via the `create` flag instead.

**kubectl and Helm only work over Tailscale** — k3s API port 6443 is blocked from the public internet. HTTP/HTTPS (80/443) are open publicly for the website. kubeconfig lives at `~/.kube/devsecops-config`.

**ArgoCD manages everything after bootstrap** — once Phase 2 is applied, ArgoCD reads `argocd-app-of-apps/values.yaml` from GitHub and syncs all four applications with `automated: prune: true, selfHeal: true`. Manual `helm install` or `kubectl apply` should not be used for managed apps.

## Sub-project commands

### devsecops-v5 (portfolio site)
```bash
cd devsecops-v5
npm start          # dev server at localhost:3000
npm run build      # production build → build/
npm run clear      # clear Docusaurus cache
```

### devsecops-infra (Terraform)
```bash
cd devsecops-infra
terraform init
terraform validate
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars
terraform fmt -recursive
terraform destroy -var-file=terraform.tfvars
```

### Helm charts (devsecops-helm)
```bash
# Lint a chart
helm lint devsecops-helm/syesite-chart

# Dry-run template render
helm template syesite devsecops-helm/syesite-chart --values devsecops-helm/syesite-chart/values.yaml
```

### ArgoCD App of Apps
```bash
# Validate the Helm chart
helm template argocd-apps argocd-app-of-apps --values argocd-app-of-apps/values.yaml
```

## Adding a new application to the cluster

1. Create a Helm chart directory in `devsecops-helm/` with `Chart.yaml`, `values.yaml`, and `templates/`
2. Add an entry to `argocd-app-of-apps/values.yaml` pointing at the new path in `devsecops-helm`
3. Push both repos — ArgoCD picks up the new application automatically

## State and secrets

- Terraform remote state: S3 bucket `devsecops-infra-tfstate` (survives `terraform destroy`)
- `devsecops-infra/terraform.tfvars` is gitignored — contains `tailscale_authkey`, `tailscale_oauth_clientid`, `tailscale_oauth_secret`, `ssh_public_key`, `argocd_github_repo`
- Tailscale pre-auth keys expire (default 90 days) — generate a fresh key before each deploy
- kubeconfig at `~/.kube/devsecops-config` is local-only, regenerated after each destroy/rebuild
