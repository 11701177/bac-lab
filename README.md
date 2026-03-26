# CI/CD Lab: Build, Push & Deploy Nginx on OpenShift

In this lab you will fork this repo, customise an Nginx container, let GitHub Actions build and push the image automatically, and deploy it on OpenShift via ArgoCD.

**Pipeline overview:**
```
git push → GitHub Actions → docker build → push to benhari/nginx-lab → ArgoCD syncs → live on OpenShift
```

---

## Repo structure

```
bac-lab/
├── .github/
│   └── workflows/
│       └── build.yml          # GitHub Actions — builds & pushes the image
├── nginx/
│   ├── Dockerfile             # Builds the nginx container
│   ├── nginx.conf             # Nginx config (port 8080, OpenShift-safe)
│   └── index.html             # ✏️ Customise this (optional)
├── argocd/
│   ├── deployment.yaml        # Kubernetes Deployment
│   ├── service.yaml           # Kubernetes Service
│   └── route.yaml             # OpenShift Route (HTTPS)
├── .gitignore
└── README.md
```

---

## Prerequisites

- A GitHub account
- Access to the OpenShift cluster and ArgoCD (instructor provides URL + credentials)
- Docker installed locally (optional, for local testing)

---

## Step 1 — Fork & clone the repo

Fork this repository to your own GitHub account, then clone it:

```bash
git clone https://github.com/YOUR_GITHUB_USERNAME/bac-lab
cd bac-lab
```

---

## Step 2 — Customise your Nginx page *(optional)*

Open `nginx/index.html` and replace `YOUR_NAME` and `YOUR_STUDENT_ID` with your own values. Feel free to change colours or add content.

Test it locally before pushing:

```bash
docker build -t nginx-lab-test ./nginx
docker run --rm -p 8080:8080 nginx-lab-test
# Open http://localhost:8080
```

---

## Step 3 — Add the GitHub secret

The CI workflow needs a shared token to push to Docker Hub. Your instructor will provide the token value.

Go to your fork → **Settings → Secrets and variables → Actions → New repository secret** and add:

| Name | Value |
|------|-------|
| `REGISTRY_TOKEN` | *(provided by instructor)* |

> Do not share this token outside the lab.

---

## Step 4 — Set your student ID and version

Open `.github/workflows/build.yml` and update the two lines at the top of the `env` block:

```yaml
env:
  STUDENT_ID: "student-00"   # ✏️ replace with your student ID, e.g. student-12
  VERSION: "v1"              # ✏️ bump this every time you push a new image
```

Your image will be pushed as:
```
docker.io/benhari/nginx-lab:student-12-v1
```

**Every time you push a new image, bump the version** (`v1` → `v2` → `v3` etc). Never overwrite a tag you have already deployed.

---

## Step 5 — Push and watch CI run

Commit your changes and push to `master`:

```bash
git add .
git commit -m "feat: customise page and set student ID"
git push origin master
```

Go to the **Actions** tab in your fork and watch the pipeline. Once it completes with a green checkmark, your image is live in the registry.

---

## Step 6 — Update the Kubernetes manifests

Replace all placeholders in the `argocd/` folder:

| File | What to change |
|------|---------------|
| `deployment.yaml` | `YOUR_NAME` in name/labels, `YOUR_STUDENT_ID` in namespace, set `image:` to your tag |
| `service.yaml` | `YOUR_NAME` in name and labels, `YOUR_STUDENT_ID` in namespace |
| `route.yaml` | `YOUR_NAME` in name and labels, `YOUR_STUDENT_ID` in namespace |

The image line in `deployment.yaml` should look like:

```yaml
image: docker.io/benhari/nginx-lab:student-12-v1
```

The namespace in all three files should look like:

```yaml
namespace: bac-lab-student-12
```

Commit and push these changes.

---

## Step 7 — Create your ArgoCD application

Open the ArgoCD UI (instructor provides the URL) and log in with your credentials.

Click **+ New App** and fill in the form:

| Field | Value |
|-------|-------|
| Application Name | `nginx-lab-YOUR_NAME` |
| Project | `default` |
| Sync Policy | `Manual` |
| Repository URL | your fork's GitHub URL |
| Revision | `master` |
| Path | `argocd` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `bac-lab-YOUR_STUDENT_ID` (e.g. `bac-lab-student-12`) |

Click **Create**, then click **Sync** → **Synchronize**.

Wait until the ArgoCD UI shows **Synced** and **Healthy**.

> **GitOps principle:** You never `kubectl apply` your deployment directly. ArgoCD watches your repo and keeps the cluster in sync with whatever is in Git.

---

## Step 8 — Verify your deployment

```bash
# Check your pod is running
oc get pods -n bac-lab-YOUR_STUDENT_ID

# Get your public URL
oc get route nginx-lab-YOUR_NAME -n bac-lab-YOUR_STUDENT_ID
```

Open the URL in your browser — you should see your custom page.

**Bonus:** Make another change to `index.html`, bump to `v2` in `build.yml`, update the image tag in `deployment.yaml`, push everything, and watch ArgoCD redeploy automatically.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Pipeline fails at "Log in" | Missing secret | Check `REGISTRY_TOKEN` in Settings → Secrets |
| Wrong image tag pushed | Forgot to update `STUDENT_ID` | Update the `env` block in `build.yml` |
| Pod stays in `Pending` | Resource quota | `oc describe pod <name>` — check Events |
| `CrashLoopBackOff` | Port/permission error | `oc logs <pod>` — nginx must listen on port 8080 |
| ArgoCD shows `OutOfSync` | Manifest not updated | Update `image:` in `deployment.yaml` to match the tag you pushed |
| Route returns 503 | Pod not ready | `oc describe pod <name>` — check readiness probe |

---

## Cheatsheet

```bash
# Docker (local testing only)
docker build -t myimage ./nginx
docker run -p 8080:8080 myimage

# OpenShift
oc get pods -n NAMESPACE
oc logs POD_NAME
oc describe pod POD_NAME
oc get route -n NAMESPACE
oc rollout restart deployment/DEPLOYMENT_NAME
```
