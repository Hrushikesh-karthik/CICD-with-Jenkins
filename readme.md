# CI/CD Lab — Installation Guide

End-to-end pipeline: **GitHub → Jenkins → Docker Hub → GitOps repository → Argo CD → Minikube**. The example application will be Python Flask.

## 0. Prerequisites

- Windows 10/11 with WSL2 and Ubuntu
- Docker Desktop for Windows with **Settings → Resources → WSL Integration → Ubuntu** enabled
- GitHub and Docker Hub accounts
- Recommended: 16 GB host RAM; 8 GB may work with reduced allocations. Allow several GB free for Docker and Kubernetes.

Run Windows commands in **PowerShell** and Linux commands in **Ubuntu WSL**, as labeled. Keep project files under `~/devops-cicd` rather than `/mnt/c`.

## 1. Install / verify WSL2 (Windows PowerShell)

```powershell
wsl --install -d Ubuntu
wsl --update
wsl --list --verbose
```

If Ubuntu shows WSL version 1, run `wsl --set-version Ubuntu 2` (adjust the distro name if needed). Restart Windows if prompted. If Ubuntu is already installed with version 2, skip installation.

## 2. Install Docker Desktop (Windows)

Download and install from https://docs.docker.com/desktop/setup/install/windows-install/ . Start Docker Desktop, choose the WSL2 engine, and enable **Resources → WSL Integration → Ubuntu**. Apply and restart.

Verify in **Ubuntu WSL**:

```bash
docker --version
docker info
docker run --rm hello-world
```

Do **not** separately install a second Docker Engine inside WSL when using Docker Desktop integration.

## 3. Install base utilities and Git (Ubuntu WSL)

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl git gnupg unzip jq
mkdir -p ~/devops-cicd
cd ~/devops-cicd
git --version
```

Optional Git identity (use your own details):

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

## 4. Install kubectl (Ubuntu WSL)

```bash
ARCH=$(dpkg --print-architecture)
KVER=$(curl -fsSL https://dl.k8s.io/release/stable.txt)
curl -fLO "https://dl.k8s.io/release/${KVER}/bin/linux/${ARCH}/kubectl"
curl -fLO "https://dl.k8s.io/release/${KVER}/bin/linux/${ARCH}/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

Official documentation: https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

## 5. Install Minikube (Ubuntu WSL)

For a typical x86-64 laptop:

```bash
curl -fLO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

Start with a modest allocation for an approximately 8 GB WSL memory limit:

```bash
minikube start --driver=docker --cpus=3 --memory=4096
kubectl get nodes
kubectl get pods -A
minikube status
```

If Minikube reports insufficient resources, check Docker Desktop and WSL resource limits first. Argo CD plus Jenkins may need more memory than this setup comfortably provides.

Official documentation: https://minikube.sigs.k8s.io/docs/start/

## 6. Install Jenkins LTS (Docker container)

Run Jenkins **outside** Minikube:

```bash
docker volume create jenkins_home
docker run -d \
  --name jenkins \
  --restart unless-stopped \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk21
```

Verify and unlock:

```bash
docker ps
docker logs --tail=80 jenkins
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open http://localhost:8080 in your Windows browser, enter the initial password, install suggested plugins, and create an administrator account. Do not share the password publicly.

**Security:** This Jenkins controller does not have Docker CLI or host socket access by default. Later, configure a separate Jenkins build agent with appropriately restricted Docker build capabilities. Do not mount `/var/run/docker.sock` into the controller as a quick fix. The `50000` port is only needed for certain inbound-agent setups; remove its port mapping if unused. Keep Jenkins local until authentication and network access are secured.

Official documentation: https://www.jenkins.io/doc/book/installing/docker/

## 7. Install Argo CD in Minikube

First check that Minikube is running:

```bash
kubectl get nodes
```

Install Argo CD using the official stable manifests:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=300s
kubectl get pods -n argocd
```

Wait until all required pods are Ready. In a separate terminal, expose the UI locally:

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Open **https://localhost:8081** and accept the local development certificate warning if shown. Username: `admin`. Retrieve the initial password in another Ubuntu terminal:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

Change the initial password after logging in. Do not publish this interface or password. `kubectl port-forward` must stay running while you use the UI.

Official documentation: https://argo-cd.readthedocs.io/en/stable/getting_started/

## 8. Optional: Install Argo CD CLI (Ubuntu WSL)

The web UI is sufficient initially. If you want the CLI:

```bash
ARGO_ARCH=$(uname -m)
case "$ARGO_ARCH" in
  x86_64) ARGO_ARCH=amd64 ;;
  aarch64) ARGO_ARCH=arm64 ;;
  *) echo "Unsupported architecture: $ARGO_ARCH"; exit 1 ;;
esac
curl -fsSL -o argocd-linux "https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-${ARGO_ARCH}"
sudo install -m 555 argocd-linux /usr/local/bin/argocd
rm argocd-linux
argocd version --client
```

## 9. Install Python Flask development dependencies (Ubuntu WSL)

```bash
sudo apt-get install -y python3 python3-venv python3-pip
mkdir -p ~/devops-cicd/flask-app
cd ~/devops-cicd/flask-app
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install flask pytest
python --version
python -m flask --version
```

Later, pin application dependencies in `requirements.txt` and create a `Dockerfile`, `Jenkinsfile`, tests, Kubernetes manifests and an Argo CD Application.

## 10. Final verification

```bash
git --version
docker --version
kubectl version --client
minikube version
minikube status
kubectl get nodes
kubectl get pods -n argocd
docker ps --filter name=jenkins
python3 --version
```

Expected: Minikube is running, Kubernetes node is `Ready`, Argo CD workloads become `Running`/`Ready`, and Jenkins container is running. Jenkins UI: http://localhost:8080; Argo CD UI (while port-forward runs): https://localhost:8081.

## 11. Resource management / troubleshooting

```bash
free -h
docker stats --no-stream
kubectl get pods -A
kubectl describe pod -n argocd <pod-name>
docker logs --tail=100 jenkins
```

If your machine becomes slow, stop services you are not using:

```bash
minikube stop
docker stop jenkins
```

Restart:

```bash
docker start jenkins
minikube start
```

If the Jenkins container already exists, use `docker start jenkins` rather than repeating `docker run`. If port 8080 is occupied, check `docker ps` and `ss -ltnp` before choosing another host port.

## 12. Next: actual CI/CD wiring (not installed automatically)

1. Create Flask app, tests, Dockerfile and `.dockerignore`.
2. Push source to GitHub and create a separate GitOps repository with Kubernetes YAML.
3. Configure a Jenkins build agent and credentials for GitHub and Docker Hub.
4. Create a Jenkins pipeline: checkout → test → build image → push versioned image → update GitOps repository.
5. Create Argo CD Application targeting the GitOps repository; enable auto-sync.
6. Set up a GitHub webhook (requires a securely reachable Jenkins endpoint, or use polling for a local-only lab).
7. Push code and confirm the new version rolls out in Minikube.

**Important:** Do not store credentials in Git repositories. Prefer Jenkins Credentials and short-lived or narrowly scoped tokens. Use immutable image tags such as Git commit SHAs, not only `latest`.
