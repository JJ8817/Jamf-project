```markdown
# Hello World Flask - GitOps Deployment with ArgoCD

A simple Flask application demonstrating GitOps principles using ArgoCD, Helm, and Kubernetes. This project deploys a "Hello World" Flask app to a local Minikube cluster with automated continuous deployment.

## Architecture Overview

```
GitHub Repository (Source of Truth)
    ↓
ArgoCD (GitOps Controller)
    ↓
Minikube Kubernetes Cluster
    ↓
Flask Application Pod
```

### How It Works Together

1. **Application Code**: Simple Flask app (`app.py`) serves "Hello world" on port 5000
2. **Containerization**: Application is containerized and pushed to GHCR as `ghcr.io/jj8817/hello-world-flask`
3. **Helm Chart**: Kubernetes manifests templated in `helm/hello-world/` directory
4. **ArgoCD Application**: Monitors GitHub repository and automatically syncs changes to Kubernetes
5. **GitOps Workflow**: Any changes pushed to GitHub trigger automatic deployment through ArgoCD

## Project Structure

```
.
├── app.py                              # Flask application
├── test_app.py                         # Unit tests
├── requirements.txt                    # Python dependencies
├── helm/
│   └── hello-world/                    # Helm chart directory
│       ├── Chart.yaml                  # Chart metadata
│       ├── values.yaml                 # Default values
│       ├── values-staging.yaml         # Staging environment overrides
│       ├── values-prod.yaml            # Production environment overrides
│       └── templates/                  # Kubernetes manifest templates
│           ├── deployment.yaml
│           ├── service.yaml
│           └── _helpers.tpl
└── argocd/
    └── hello-world-staging.yaml        # ArgoCD app definition for staging
```

## Prerequisites

- **Homebrew** (macOS package manager)
- **Docker Desktop** (for container runtime)
- **Minikube** (local Kubernetes cluster)
- **kubectl** (Kubernetes CLI)
- **ArgoCD CLI** (GitOps deployment tool)
- **Python 3.x** (for local testing)
- **GitHub CLI** (for GHCR authentication)

## Setup Instructions

### 1. Install Required Tools

```
# Install tools via Homebrew
brew install minikube kubectl argocd gh

# Verify installations
minikube version
kubectl version --client
argocd version
gh version
```

### 2. Start Minikube Cluster

```
# Start Minikube with adequate resources
minikube start --memory=4096 --cpus=2

# Verify cluster is running
minikube status
kubectl cluster-info
```

### 3. Install ArgoCD

```
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready (takes 2-3 minutes)
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

# Verify installation
kubectl get pods -n argocd
```

### 4. Access ArgoCD UI

```
# In a separate terminal, expose ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Login via CLI
argocd login localhost:8080 --username admin --password <password-from-above> --insecure
```

Access the UI at **https://localhost:8080**
- Username: `admin`
- Password: (from command above)

### 5. Deploy Application

```
# Apply the ArgoCD Application manifest
kubectl apply -f argocd/hello-world-staging.yaml

# Check application status
argocd app list
argocd app get hello-world-staging

# Manually sync if not auto-syncing
argocd app sync hello-world-staging
```

### 6. Access the Application

**⚠️ macOS AirPlay uses port 5000 by default - use port 5001!**

```
# Method 1: kubectl port-forward (use port 5001 to avoid AirPlay conflict)
kubectl port-forward -n hello-staging svc/hello-world-staging 5001:80
# Then access at http://127.0.0.1:5001/

# Method 2: Minikube service (recommended - use port 5001)
minikube service hello-world-staging -n hello-staging --url --port 5001
# Click the provided URL (usually http://192.168.49.2:5001)

# Test the endpoint
curl http://127.0.0.1:5001/
# Expected output: Hello world
```

## Local Development & Testing

### Run Flask App Locally

```
# Install dependencies
pip3 install -r requirements.txt

# Run the application
python3 app.py

# Test in another terminal (port 5001 to avoid AirPlay)
curl http://127.0.0.1:5001/
```

### Run Unit Tests

```
# Install pytest
pip3 install pytest

# Run tests
pytest test_app.py -v
# Expected output: test_app.py::test_hello PASSED
```

## Common Pain Points & Solutions

### 1. **macOS Port 5000 403 Forbidden Error** ⭐ **MOST COMMON**

**Problem**: Browser shows "403 Forbidden" with `Server: AirTunes` header

**Cause**: macOS AirPlay receiver hijacks port 5000

**Solution**:
```
# Use port 5001 instead of 5000
kubectl port-forward -n hello-staging svc/hello-world-staging 5001:80
minikube service hello-world-staging -n hello-staging --url --port 5001

# Or disable AirPlay receiver (one-time)
sudo /System/Library/CoreServices/AirPlayXPCHelper --disable-receiver
```

### 2. YAML Parsing Errors

**Problem**: `error converting YAML to JSON: yaml: line X: could not find expected ':'`

**Solution**: Use these exact commands to recreate files:
```
# deployment.yaml (exact copy-paste)
cat > helm/hello-world/templates/deployment.yaml << 'EOF'
[perfect YAML content here]
EOF
```

### 3. Service Not Found

**Problem**: `Service 'hello-world' was not found`

**Solution**: Service is named `hello-world-staging`:
```
kubectl get svc -n hello-staging
# Use: hello-world-staging (not hello-world)
minikube service hello-world-staging -n hello-staging --url --port 5001
```

### 4. Multi-Platform Docker Build Errors

**Problem**: `Multi-platform build is not supported for the docker driver`

**Solution**: Add these steps to `.github/workflows/ci.yaml`:
```
- uses: docker/setup-qemu-action@v3
- uses: docker/setup-buildx-action@v3
- name: Build and push
  uses: docker/build-push-action@v6
  with:
    platforms: linux/amd64,linux/arm64
```

### 5. Git Push Rejected (non-fast-forward)

**Problem**: `rejected... tip of your current branch is behind`

**Solution**:
```
git pull --rebase origin main
git push origin main
```

## GitOps Workflow

### Making Changes

1. **Update code** in your local repository
2. **Commit and push** to GitHub:
   ```
   git pull --rebase origin main
   git add .
   git commit -m "Update application"
   git push origin main
   ```
3. **ArgoCD automatically detects** changes (3 minutes)
4. **Application syncs** automatically

## Verification Commands

```
# ArgoCD status
argocd app get hello-world-staging

# Kubernetes resources
kubectl get all -n hello-staging

# Pod logs
kubectl logs -n hello-staging -l app=hello-world

# Test endpoint
curl $(minikube service hello-world-staging -n hello-staging --url --port 5001 | tail -1)
```

## Cleanup

```
argocd app delete hello-world-staging
kubectl delete namespace hello-staging
kubectl delete namespace argocd
minikube delete
```

## Troubleshooting Tips

1. **Port 5001, not 5000** - macOS AirPlay conflict
2. **Service name**: `hello-world-staging` (not `hello-world`)
3. **Namespace**: Always use `-n hello-staging`
4. **Check logs first**: `kubectl logs`, `kubectl describe`
5. **Validate YAML**: `helm template helm/hello-world -f values-staging.yaml`

🎉 **Your GitOps pipeline is production-ready!**
```