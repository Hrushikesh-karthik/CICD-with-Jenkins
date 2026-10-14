# End-to-End CI/CD Pipeline with Jenkins, Docker, SonarQube, Trivy, Helm, Argo CD and Minikube

## Project Overview

This project automates application delivery using Jenkins and GitOps.

**Workflow:**

1. A developer pushes code to GitHub.
2. A GitHub webhook triggers Jenkins.
3. Jenkins runs automated tests.
4. SonarQube analyzes code quality.
5. Docker builds the application image.
6. Trivy scans the image for vulnerabilities.
7. Jenkins pushes the approved image to Docker Hub.
8. Jenkins updates the image tag in a GitOps repository.
9. Argo CD detects the Git change and synchronizes the Helm release.
10. Minikube runs the updated application.

## Architecture

- Host: Ubuntu on WSL 2
- CI: Jenkins installed directly on Ubuntu
- Containers: Docker
- Code quality: SonarQube
- Image security: Trivy
- Registry: Docker Hub
- Kubernetes: Minikube
- Package manager: Helm
- GitOps CD: Argo CD

No Jenkins Docker container, Docker-in-Docker, or separate Jenkins agent is required.

---

## 1. Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y \
  curl \
  wget \
  git \
  unzip \
  zip \
  gnupg \
  ca-certificates \
  lsb-release \
  fontconfig \
  apt-transport-https \
  software-properties-common
```

Check the Ubuntu version:

```bash
lsb_release -a
```

## 2. Install Java 21

Jenkins requires a supported Java runtime. This project uses OpenJDK 21.

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
```

Verify Java:

```bash
java -version
javac -version
```

Check the Java installation path:

```bash
readlink -f "$(which java)"
```

## 3. Install Jenkins LTS directly on Ubuntu

Add the official Jenkins repository key:

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
```

Add the Jenkins LTS repository:

```bash
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Install Jenkins:

```bash
sudo apt update
sudo apt install -y jenkins
```

Start Jenkins:

```bash
sudo systemctl enable --now jenkins
```

Check its status:

```bash
sudo systemctl status jenkins --no-pager
```

Open Jenkins in your browser:

http://localhost:8080

Retrieve the initial administrator password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Complete the initial setup wizard and install the suggested plugins.

**Important:** These repository commands use the Jenkins LTS repository and its published signing-key URL. If the key URL changes, follow the current official Jenkins installation guide.

Official documentation:
https://www.jenkins.io/doc/book/installing/linux/

## 4. Install Docker Engine

This guide installs Docker Engine inside Ubuntu. It does not use a Jenkins Docker image or Docker-in-Docker.

### 4.1 Configure the Docker repository

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the official repository:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 4.2 Install Docker

```bash
sudo apt update

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### 4.3 Start Docker

```bash
sudo systemctl enable --now docker
```

Test Docker:

```bash
sudo docker run --rm hello-world
```

Check Docker:

```bash
docker --version
sudo docker info
```

### 4.4 Allow your user to run Docker commands

```bash
sudo usermod -aG docker "$USER"
```

Apply the new group membership by closing and reopening the Ubuntu terminal, or run:

```bash
newgrp docker
```

Test:

```bash
docker run --rm hello-world
```

### 4.5 Allow Jenkins to access Docker

Jenkins runs as the `jenkins` Linux user. Add it to the Docker group:

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```

Verify Docker access as Jenkins:

```bash
sudo -u jenkins -H docker info
```

If this fails, troubleshoot Docker permissions and the Docker daemon before proceeding.

**Security note:** Membership in the Docker group effectively grants root-level control over the host. Use this configuration only on a machine you control.

Official documentation:
https://docs.docker.com/engine/install/ubuntu/

## 5. Install kubectl

Add the Kubernetes package repository. This example uses the v1.34 repository; use a supported minor version compatible with your Minikube cluster.

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add the repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install kubectl:

```bash
sudo apt update
sudo apt install -y kubectl
```

Verify:

```bash
kubectl version --client
```

Official documentation:
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

## 6. Install Minikube

Download the latest stable Linux AMD64 binary:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

sudo install minikube-linux-amd64 /usr/local/bin/minikube

rm minikube-linux-amd64
```

Verify:

```bash
minikube version
```

Start Minikube using Docker:

```bash
minikube start --driver=docker
```

Verify the cluster:

```bash
minikube status
kubectl get nodes
kubectl get pods -A
```

Set Docker as the default Minikube driver:

```bash
minikube config set driver docker
```

**WSL note:** This setup assumes Docker Engine works inside WSL 2. If you use Docker Desktop instead, enable WSL integration for your Ubuntu distribution and use Docker Desktop's daemon rather than installing a second Docker Engine.

Official documentation:
https://minikube.sigs.k8s.io/docs/start/

## 7. Install Helm

Download and run the official Helm installation script:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \
  -o get_helm.sh

chmod 700 get_helm.sh
./get_helm.sh

rm get_helm.sh
```

Verify:

```bash
helm version
```

Official documentation:
https://helm.sh/docs/intro/install/

## 8. Install the Argo CD CLI

Download the latest stable Argo CD CLI for Linux AMD64:

```bash
ARGOCD_VERSION=$(curl -fsSL https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION | tr -d '[:space:]')

curl -fL \
  "https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64" \
  -o argocd

sudo install -m 0755 argocd /usr/local/bin/argocd

rm argocd
```

Verify:

```bash
argocd version --client
```

## 9. Install Argo CD into Minikube

Create the namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd \
  --server-side \
  --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for the components to become ready:

```bash
kubectl wait \
  --for=condition=Available \
  --timeout=300s \
  deployment --all -n argocd
```

Check the pods:

```bash
kubectl get pods -n argocd
```

Access the UI through port forwarding:

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Open:

https://localhost:8081

Retrieve the initial admin password:

```bash
argocd admin initial-password -n argocd
```

Log in from another terminal:

```bash
argocd login localhost:8081 \
  --username admin \
  --password '<INITIAL_PASSWORD>' \
  --insecure
```

Change the initial password after logging in.

Official documentation:
https://argo-cd.readthedocs.io/en/stable/getting_started/

## 10. Install Trivy

Add the official Trivy repository:

```bash
sudo apt-get update
sudo apt-get install -y wget gnupg

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | gpg --dearmor \
  | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
```

```bash
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" \
  | sudo tee /etc/apt/sources.list.d/trivy.list
```

Install Trivy:

```bash
sudo apt-get update
sudo apt-get install -y trivy
```

Verify:

```bash
trivy --version
```

Update the vulnerability database and test a scan:

```bash
trivy image alpine:latest
```

Official documentation:
https://trivy.dev/latest/getting-started/installation/

## 11. Run SonarQube

For this local project, SonarQube can run as a persistent Docker container. Jenkins itself remains installed directly on Ubuntu.

Pull the Community Build image:

```bash
docker pull sonarqube:community
```

Create a persistent volume:

```bash
docker volume create sonarqube_data
```

Start SonarQube:

```bash
docker run -d \
  --name sonarqube \
  --restart unless-stopped \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  sonarqube:community
```

Check its status:

```bash
docker ps
docker logs -f sonarqube
```

Open:

http://localhost:9000

Sign in using the initial credentials displayed by the current SonarQube setup, then change the password and create a project token for Jenkins.

**Note:** SonarQube requires sufficient memory and disk space. Its current supported Java runtime, host kernel requirements, and recommended memory depend on the release. Check the official documentation before using it on a resource-constrained laptop.

Official documentation:
https://docs.sonarsource.com/sonarqube-server/

## 12. Install Git and verify the environment

Git should already be installed in Step 1. Verify:

```bash
git --version
```

Configure your Git identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

Verify the major tools:

```bash
java -version
git --version
docker --version
kubectl version --client
minikube version
helm version
argocd version --client
trivy --version
```

Check services:

```bash
sudo systemctl status jenkins --no-pager
sudo systemctl status docker --no-pager
minikube status
kubectl get nodes
kubectl get pods -n argocd
```

---

## 13. Components that require configuration

Installing the tools does not finish the pipeline. Complete these configuration tasks next:

1. Install Jenkins plugins for Pipeline, Git, GitHub integration, and SonarQube integration.
2. Configure Docker Hub credentials in Jenkins.
3. Configure the SonarQube server and project token in Jenkins.
4. Configure a GitHub webhook that can reach Jenkins. A local `localhost` URL is not directly reachable from GitHub; use a secure tunnel or a publicly reachable endpoint.
5. Create the application repository and its automated tests.
6. Create the application's Dockerfile.
7. Create the Helm chart and GitOps repository.
8. Configure the Argo CD application to watch the GitOps repository.
9. Create the Jenkinsfile with the required stages and failure conditions.

## Expected final workflow

GitHub push → Jenkins webhook → automated tests → SonarQube → Docker build → Trivy scan → Docker Hub push → Helm values update in Git → Argo CD synchronization → Minikube deployment.

The application image should use a unique tag, such as the Git commit SHA, instead of relying exclusively on `latest`.
