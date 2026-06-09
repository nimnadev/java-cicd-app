# Java CI/CD Pipeline — Step by Step Guide
## Jenkins + SonarQube + ArgoCD + Helm + Kubernetes

---

## 📁 Project Structure

```
java-cicd-app/                        ← Push this to GitHub Repo 1
├── src/
│   ├── main/java/com/example/app/
│   │   ├── Application.java
│   │   └── HelloController.java
│   ├── test/java/com/example/app/
│   │   └── HelloControllerTest.java
│   └── main/resources/
│       └── application.properties
├── helm-chart/                       ← Push this to GitHub Repo 2 (ArgoCD watches this)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── deployment.yaml
│       └── service.yaml
├── Dockerfile
├── Jenkinsfile
├── pom.xml
└── .gitignore
```

---

## STEP 1 — Create Two GitHub Repositories

You need **2 separate repos**:

| Repo | Purpose |
|------|---------|
| `java-cicd-app` | Source code (Java + Jenkinsfile + Dockerfile) |
| `helm-charts-repo` | Helm chart only (ArgoCD watches this) |

### Create repos on GitHub:
1. Go to https://github.com → New Repository
2. Create `java-cicd-app` (public or private)
3. Create `helm-charts-repo` (public or private)

---

## STEP 2 — Push Source Code to GitHub

```bash
# Go into the project folder
cd java-cicd-app

# Initialize git
git init
git add .
git commit -m "initial commit: java cicd app"

# Add your GitHub remote (replace YOUR_USERNAME)
git remote add origin https://github.com/YOUR_USERNAME/java-cicd-app.git

# Push
git branch -M main
git push -u origin main
```

---

## STEP 3 — Push Helm Chart to Second Repo

```bash
# Copy helm chart to a separate folder
cp -r helm-chart ../helm-charts-repo
cd ../helm-charts-repo

git init
git add .
git commit -m "initial commit: helm chart"
git remote add origin https://github.com/YOUR_USERNAME/helm-charts-repo.git
git branch -M main
git push -u origin main
```

---

## STEP 4 — Update Placeholders in Jenkinsfile

Open `Jenkinsfile` and update these values:

```
DOCKER_IMAGE   → your-dockerhub-username/java-cicd-app
SONAR_URL      → http://your-sonarqube-ip:9000
HELM_REPO_URL  → https://github.com/YOUR_USERNAME/helm-charts-repo
```

Also update `helm-chart/values.yaml`:
```
repository: your-dockerhub-username/java-cicd-app
```

---

## STEP 5 — Install SonarQube (if not installed)

```bash
# Run SonarQube via Docker
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  sonarqube:lts-community

# Open browser: http://localhost:9000
# Default login: admin / admin
```

---

## STEP 6 — Jenkins Setup

### Install Jenkins on Kubernetes:
```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --create-namespace \
  --set controller.serviceType=NodePort
```

### Get Jenkins password:
```bash
kubectl exec --namespace jenkins -it svc/jenkins \
  -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```

### Install Jenkins Plugins:
Go to **Manage Jenkins → Plugins → Available** and install:
- Git Plugin
- Maven Integration Plugin
- Pipeline Plugin
- SonarQube Scanner Plugin
- Docker Pipeline Plugin

---

## STEP 7 — Add Credentials to Jenkins

Go to **Manage Jenkins → Credentials → Global → Add Credential**

| ID | Type | What to add |
|----|------|-------------|
| `dockerhub-creds` | Username/Password | DockerHub username + password |
| `github-token` | Secret text | GitHub Personal Access Token |

### Create GitHub Token:
1. GitHub → Settings → Developer Settings → Personal Access Tokens
2. Generate token with `repo` scope
3. Copy and add to Jenkins

---

## STEP 8 — Configure SonarQube in Jenkins

1. **Jenkins → Manage Jenkins → System**
2. Find **SonarQube Servers** section
3. Add server:
   - Name: `SonarQube`
   - URL: `http://your-sonarqube-ip:9000`
   - Token: (generate from SonarQube → My Account → Security → Generate Token)

---

## STEP 9 — Create Jenkins Pipeline Job

1. Jenkins → **New Item**
2. Name: `java-cicd-pipeline`
3. Type: **Pipeline**
4. Under **Pipeline** section:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/YOUR_USERNAME/java-cicd-app`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
5. Click **Save**

---

## STEP 10 — Set Up ArgoCD Application

```bash
# Login to ArgoCD CLI
argocd login localhost:8080 --username admin --password YOUR_PASSWORD --insecure

# Create ArgoCD app pointing to helm-charts-repo
argocd app create java-cicd-app \
  --repo https://github.com/YOUR_USERNAME/helm-charts-repo \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Verify
argocd app list
argocd app get java-cicd-app
```

---

## STEP 11 — Set Up GitHub Webhook (Auto-trigger Jenkins)

1. Go to your `java-cicd-app` GitHub repo
2. **Settings → Webhooks → Add webhook**
3. Payload URL: `http://YOUR_JENKINS_IP:8080/github-webhook/`
4. Content type: `application/json`
5. Events: **Just the push event**
6. Click **Add webhook**

---

## STEP 12 — Enable Auto-trigger in Jenkins Job

1. Open `java-cicd-pipeline` job
2. **Configure → Build Triggers**
3. Check ✅ **GitHub hook trigger for GITScm polling**
4. Save

---

## ✅ Full Flow — How It Works Now

```
Developer pushes code to GitHub
        ↓
GitHub Webhook triggers Jenkins
        ↓
Jenkins: Checkout → Build → Test → SonarQube
        ↓
Jenkins: Build Docker Image → Push to DockerHub
        ↓
Jenkins: Update image tag in helm-charts-repo
        ↓
ArgoCD detects change in helm-charts-repo
        ↓
ArgoCD deploys new version to Kubernetes ✅
```

---

## 🔧 Troubleshooting

| Issue | Fix |
|-------|-----|
| Jenkins can't reach SonarQube | Check SonarQube is running, firewall ports open |
| Docker push fails | Verify dockerhub-creds in Jenkins |
| ArgoCD not syncing | Run `argocd app sync java-cicd-app` manually |
| Helm deploy fails | Check `kubectl get pods` and `kubectl describe pod` |
| Webhook not triggering | Verify Jenkins URL is accessible from internet |

---

## 📞 Quick Commands Reference

```bash
# Check pods
kubectl get pods -n argocd
kubectl get pods -n jenkins
kubectl get pods -n default

# ArgoCD
argocd app list
argocd app sync java-cicd-app
argocd app logs java-cicd-app

# Jenkins logs
kubectl logs -n jenkins deployment/jenkins

# SonarQube
docker logs sonarqube
```
