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
2. **Containerization**: Application is containerized and pushed to Docker Hub as `jj1729/hello-world-flask`
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
│           └── ...
└── argocd/
    ├── hello-world-staging.yaml        # ArgoCD app definition for staging
    └── argo-cd-app.yaml                # ArgoCD app definitions (staging + prod)
```

## Prerequisites

- **Homebrew** (macOS package manager)
- **Docker Desktop** (for container runtime)
- **Minikube** (local Kubernetes cluster)
- **kubectl** (Kubernetes CLI)
- **ArgoCD CLI** (GitOps deployment tool)
- **Python 3.x** (for local testing)

## Setup Instructions

### 1. Install Required Tools

```bash
# Install tools via Homebrew
brew install minikube kubectl argocd

# Verify installations
minikube version
kubectl version --client
argocd version
```

### 2. Start Minikube Cluster

```bash
# Start Minikube with adequate resources
minikube start --memory=4096 --cpus=2

# Verify cluster is running
minikube status
kubectl cluster-info
```

### 3. Install ArgoCD

```bash
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

```bash
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

```bash
# Apply the ArgoCD Application manifest
kubectl apply -f hello-world-staging.yaml

# Check application status
argocd app list
argocd app get hello-world-staging

# Manually sync if not auto-syncing
argocd app sync hello-world-staging
```

### 6. Access the Application

```bash
# Method 1: Port forwarding
kubectl port-forward -n hello-staging svc/hello-world 5000:5000

# Then access at http://localhost:5000

# Method 2: Minikube service (recommended for local testing)
minikube service hello-world -n hello-staging --url

# Test the endpoint
curl http://localhost:5000
# Expected output: Hello world
```

## Local Development & Testing

### Run Flask App Locally

```bash
# Install dependencies
pip3 install -r requirements.txt

# Run the application
python3 app.py

# Test in another terminal
curl http://localhost:5000
```

### Run Unit Tests

```bash
# Install pytest
pip3 install pytest

# Run tests
pytest test_app.py -v

# Expected output:
# test_app.py::test_hello PASSED
```

## Common Pain Points & Solutions

### 1. YAML Parsing Errors

**Problem**: `error converting YAML to JSON: yaml: line X: could not find expected ':'`

**Causes**:
- Incorrect indentation (YAML requires consistent 2-space indentation)
- Extra blank lines breaking YAML structure
- Shell commands mixed with YAML content

**Solution**:
```bash
# Validate YAML syntax before applying
kubectl apply --dry-run=client -f your-file.yaml

# Use proper YAML structure
metadata:      # ← Must have colon
  name: app-name
  namespace: argocd
```

### 2. Chart.yaml Not Found

**Problem**: `error reading helm chart from <path>/Chart.yaml: no such file or directory`

**Cause**: ArgoCD Application `path` field doesn't match actual Helm chart location

**Solution**:
```yaml
# If Chart.yaml is in helm/hello-world/
spec:
  source:
    path: helm/hello-world  # ← Must point to directory containing Chart.yaml

# NOT path: . (unless Chart.yaml is in repo root)
```

### 3. pytest Command Not Found

**Problem**: `zsh: command not found: pytest`

**Solution**:
```bash
# Install pytest
pip3 install pytest

# Or use python module syntax
python3 -m pytest test_app.py
```

### 4. Service Not Found in Namespace

**Problem**: `Service 'hello-staging' was not found in 'default' namespace`

**Cause**: Service is in different namespace, but command didn't specify it

**Solution**:
```bash
# Always specify namespace with -n flag
minikube service hello-world -n hello-staging --url

# Check which namespace your resources are in
kubectl get svc --all-namespaces | grep hello
```

### 5. ArgoCD Application Not Syncing

**Problem**: Application shows "OutOfSync" or "Unknown" status

**Possible Causes & Solutions**:

a) **Repository Access Issues**
```bash
# Verify ArgoCD can access your repo
argocd repo list

# For private repos, add credentials
argocd repo add https://github.com/JJ8817/Hello-world-argo-app.git --username <user> --password <token>
```

b) **Helm Values File Not Found**
```yaml
# Ensure values file exists in repo
spec:
  source:
    helm:
      valueFiles:
        - values-staging.yaml  # ← Must exist in same directory as Chart.yaml
```

c) **Manual Sync Required**
```bash
# Force sync
argocd app sync hello-world-staging --force
```

### 6. Pods in CrashLoopBackOff

**Problem**: Pods keep restarting

**Diagnosis**:
```bash
# Check pod status
kubectl get pods -n hello-staging

# View pod logs
kubectl logs -n hello-staging <pod-name>

# Describe pod for events
kubectl describe pod -n hello-staging <pod-name>
```

**Common Causes**:
- Wrong container image or tag
- Application crashes on startup
- Missing environment variables
- Port conflicts

### 7. Cannot Access Application After Deployment

**Problem**: Port forwarding or service URL doesn't respond

**Checklist**:
```bash
# 1. Verify pods are running
kubectl get pods -n hello-staging
# Should show STATUS: Running

# 2. Check service exists
kubectl get svc -n hello-staging

# 3. Verify pod logs
kubectl logs -n hello-staging <pod-name>

# 4. Test within cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- curl http://hello-world.hello-staging.svc.cluster.local:5000

# 5. Check port-forward is active
# Make sure port-forward command is running in another terminal
```

### 8. Minikube Won't Start

**Problem**: `minikube start` fails or hangs

**Solutions**:
```bash
# Delete and restart cluster
minikube delete
minikube start --memory=4096 --cpus=2

# Check Docker Desktop is running (required for Minikube)

# Try different driver
minikube start --driver=hyperkit
# or
minikube start --driver=virtualbox
```

### 9. ArgoCD Can't Find Kubernetes Resources

**Problem**: ArgoCD shows "ComparisonError" or can't find resources

**Solution**:
```bash
# Refresh ArgoCD's cache
argocd app get hello-world-staging --refresh

# Hard refresh (re-fetch from Git)
argocd app get hello-world-staging --hard-refresh

# Check repository connection
argocd repo get https://github.com/JJ8817/Hello-world-argo-app.git
```

## GitOps Workflow

### Making Changes

1. **Update code** in your local repository
2. **Commit and push** to GitHub
   ```bash
   git add .
   git commit -m "Update application"
   git push origin main
   ```
3. **ArgoCD automatically detects** the change (within 3 minutes)
4. **Application syncs** to match Git state
5. **Kubernetes updates** pods with new version

### Manual Sync (for testing)

```bash
# Trigger immediate sync
argocd app sync hello-world-staging

# Sync and show live logs
argocd app sync hello-world-staging --watch
```

## Verification Commands

```bash
# Check ArgoCD application health
argocd app list
argocd app get hello-world-staging

# View Kubernetes resources
kubectl get all -n hello-staging

# Check pod logs
kubectl logs -n hello-staging -l app=hello-world --tail=50

# Test application endpoint
curl $(minikube service hello-world -n hello-staging --url)
```

## Cleanup

```bash
# Delete ArgoCD application
kubectl delete -f hello-world-staging.yaml

# Or via ArgoCD CLI
argocd app delete hello-world-staging

# Delete ArgoCD installation
kubectl delete namespace argocd

# Stop Minikube
minikube stop

# Delete Minikube cluster (complete cleanup)
minikube delete
```

## Environment Configuration

### Staging Environment
- **Namespace**: `hello-staging`
- **Values File**: `values-staging.yaml`
- **Image Tag**: `latest` (auto-updates)
- **Replicas**: 1

### Production Environment (if configured)
- **Namespace**: `hello-prod`
- **Values File**: `values-prod.yaml`
- **Image Tag**: `prod` (stable release)
- **Replicas**: 2+ (for high availability)

## Troubleshooting Tips

1. **Always check logs first**: `kubectl logs` and `kubectl describe` are your friends
2. **Verify namespace**: Most issues stem from wrong namespace context
3. **Check ArgoCD UI**: Visual representation helps identify sync/health issues
4. **Test locally first**: Run Flask app locally before debugging Kubernetes
5. **YAML validation**: Use `kubectl apply --dry-run=client` to catch syntax errors
6. **Port conflicts**: Ensure no other services are using ports 5000 or 8080

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)

## Repository

GitHub: [https://github.com/JJ8817/Hello-world-argo-app](https://github.com/JJ8817/Hello-world-argo-app)
