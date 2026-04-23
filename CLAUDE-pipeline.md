# CLAUDE.md — Contexto del proyecto GitHub Actions

## ¿Qué estamos construyendo?
Estamos migrando los workflows de GitHub Actions a una arquitectura **Fase 2**:
un repositorio de organización `.github` centralizado que contiene los **Reusable Workflows**
compartidos por todos los repos de la organización `Illumidragui`.

El objetivo es que cada repositorio de aplicación tenga un `ci.yml` mínimo que
solo llame a los workflows centralizados, sin duplicar lógica.

---

## Estructura objetivo

```
# Repositorio: Illumidragui/.github  ← ESTE repo
.github/
└── workflows/
    ├── _reusable.code-security.yml       # Gitleaks + Tests + SonarCloud + Snyk
    ├── _reusable.container-security.yml  # Build Docker + Trivy + ZAP
    ├── _reusable.build-static.yml        # npm build → artifact (Docusaurus)
    ├── _reusable.deploy-cloudflare.yml   # Deploy a Cloudflare Pages
    └── _reusable.deploy-kubernetes.yml   # Promote image + Helm bump + ArgoCD

# Repositorio: Illumidragui/syesite  ← Web (Docusaurus)
.github/workflows/
└── deploy-cloudflare.yml   # Solo llama a reusables, sin lógica propia

# Repositorio: Illumidragui/devsecops-pipeline  ← Flask app
.github/workflows/
└── deploy-kubernetes.yml   # Solo llama a reusables, sin lógica propia
```

---

## Ficheros actuales (punto de partida)

En la carpeta actual existen estos ficheros que son la base del trabajo:

| Fichero | Contenido actual |
|---|---|
| `_build` | Job de build compartido (npm, genera artifact) |
| `deploy` | Lógica de deploy genérica |
| `security` | Pipeline DevSecOps completo: Gitleaks, Tests, SonarCloud, Snyk, Trivy, ZAP |
| `deploy-kubernetes` | Pipeline completo para k8s: security + docker + helm |
| `deploy-cloudflare` | Pipeline para CF Pages: security estática + npm build + wrangler |

La tarea es **refactorizar y separar** estos ficheros en reusable workflows del repo de org.

---

## Reglas de arquitectura

### Separación de responsabilidades
- `_reusable.code-security.yml` → solo analiza código fuente. Sin Docker.
- `_reusable.container-security.yml` → solo analiza la imagen. Requiere que el código ya pasó.
- `_reusable.build-static.yml` → solo compila el site estático. Sin seguridad.
- Los workflows de deploy **nunca** contienen lógica de seguridad inline.

### Orden de ejecución obligatorio
```
code-security → build → container-security → deploy
```
Nunca saltarse pasos. El deploy solo ocurre si TODOS los gates anteriores pasan.

### Convención de naming
- Prefijo `_reusable.` → workflow llamable con `workflow_call`
- Sin prefijo → entry point (lo que dispara el push/tag)
- Los reusables NUNCA tienen `push:` o `pull_request:` como trigger, solo `workflow_call:`

### Secrets
- Los reusables declaran sus secrets explícitamente en la sección `on.workflow_call.secrets`
- Los callers usan `secrets: inherit` cuando es posible, o pasan secrets explícitos
- Nunca hardcodear valores — todo por secrets o variables de entorno

### Tags de imagen Docker
- Tag SHA (`github.sha`) → inmutable, identifica exactamente qué commit produjo la imagen
- Tag `latest` → solo se añade en el job de deploy, después de que todos los scans pasan
- Nunca construir la imagen dos veces — build una vez, escanear esa misma imagen, deployar esa misma imagen

---

## Repos y tecnologías implicadas

| Repo | Stack | Deploy target |
|---|---|---|
| `Illumidragui/syesite` | Docusaurus (Node 18, npm) | Cloudflare Pages |
| `Illumidragui/devsecops-pipeline` | Flask (Python 3.12, pip) | Kubernetes vía ArgoCD + Helm |
| `Illumidragui/devsecops-helm` | Helm charts | Actualizado por el pipeline (Helm bump) |
| `Illumidragui/devsecops-k8s-config` | ArgoCD config | PR automático con nuevo image tag |

---

## Herramientas de seguridad y scope

| Herramienta | Tipo | Aplica a CF Pages | Aplica a k8s |
|---|---|---|---|
| Gitleaks | Secret scan | ✅ | ✅ |
| Tests (pytest) | Unit tests | ✅ | ✅ |
| SonarCloud | SAST | ✅ | ✅ |
| Snyk | SCA (dependencias) | ✅ | ✅ |
| Trivy | Container scan | ❌ | ✅ |
| OWASP ZAP | DAST | ❌ | ✅ |

---

## Variables y secrets conocidos

```
# Cloudflare
CLOUDFLARE_API_TOKEN
CLOUDFLARE_ACCOUNT_ID

# Docker / GHCR
GITHUB_TOKEN        ← automático de GitHub Actions
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN

# Seguridad
SONAR_TOKEN
SNYK_TOKEN

# Helm / ArgoCD
GH_TOKEN            ← PAT con acceso a devsecops-helm
GH_PAT              ← PAT con acceso a devsecops-k8s-config
CONFIG_REPO_TOKEN   ← PAT para abrir PRs en config repo

# Cloudflare project
CF_PROJECT_NAME: website
```

---

## Comportamiento esperado del deploy

### Cloudflare Pages (syesite)
- Trigger: `push` a `main`
- Gate: code-security (Gitleaks + Tests + SonarCloud + Snyk)
- Build: `npm ci && npm run build` con `DOCUSAURUS_CONF_URL=https://shengjunye.me`
- Deploy: `wrangler pages deploy build --project-name=website`
- Sin Docker. Sin Trivy. Sin ZAP.

### Kubernetes (devsecops-pipeline)
- Trigger: `push` a tags `v*` o `workflow_dispatch`
- Gate 1: code-security
- Gate 2: container-security (Build → Trivy → ZAP en paralelo)
- Deploy: promote SHA→latest en GHCR + bump tag en Helm chart + PR en config repo
- El deploy requiere que los dos gates pasen. Si cualquier scanner falla, se bloquea.

---

## Lo que NO hacer

- No poner lógica de negocio en los callers — solo llamadas a reusables
- No duplicar steps entre workflows — si algo se repite, es un reusable
- No usar `latest` como tag de imagen en el scan — siempre SHA
- No omitir `if: always()` en los steps de reporte/upload de artefactos
- No mergear PRs del config repo automáticamente — ArgoCD detecta el PR, un humano aprueba
