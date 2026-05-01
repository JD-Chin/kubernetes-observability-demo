# kubernetes-observability-demo

A full-stack observability project deploying a 12-service microservices application on Kubernetes, hosted on AWS, and monitored with Dynatrace OneAgent. Built to demonstrate hands-on cloud infrastructure, container orchestration, and observability platform deployment.

---

## Project Overview

This project provisions a cloud environment from scratch, deploys a production-grade microservices application, and instruments it with an enterprise observability platform — replicating the kind of environment Dynatrace monitors in real enterprise deployments.

**What this demonstrates:**
- Cloud infrastructure provisioning on AWS
- Kubernetes cluster setup and management
- Multi-service application deployment via `kubectl`
- Real-time observability with Dynatrace OneAgent
- CI/CD pipeline automation with GitHub Actions
- Infrastructure security configuration
- Systematic troubleshooting and root cause analysis

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    AWS EC2                          │
│              (m7i-flex.large)                       │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │              minikube (Kubernetes)           │   │
│  │                                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ frontend │  │   cart   │  │ checkout │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘    │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ payment  │  │shipping  │  │  email   │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘    │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │currency  │  │ product  │  │   ads    │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘    │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │  redis   │  │recommend │  │  load    │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘    │   │
│  │                                              │   │
│  │  ┌──────────────────────────────────────┐    │   │
│  │  │     Dynatrace OneAgent + Operator    │    │   │
│  │  └──────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Cloud Provider | AWS (EC2 — m7i-flex.large) |
| Operating System | Ubuntu 22.04 LTS |
| Container Runtime | Docker 27.5.1 |
| Orchestration | Kubernetes via minikube v1.38.1 |
| Sample Application | Google Online Boutique (12 microservices) |
| Observability | Dynatrace OneAgent + Kubernetes Operator |
| CI/CD | GitHub Actions |
| Uptime Monitoring | UptimeRobot |
| Package Manager | Helm v3.20.2 |

---

## Prerequisites

- AWS account with EC2 access
- Dynatrace free trial account
- GitHub account
- SSH key pair (.pem file)
- Basic Linux/bash familiarity

---

## Setup & Deployment

### 1. Provision EC2 Instance

Launch an EC2 instance with the following configuration:

- **AMI:** Ubuntu 22.04 LTS
- **Instance type:** m7i-flex.large (2 vCPU, 8GB RAM)
- **Storage:** 20GB gp3
- **Security group inbound rules:**
  - Port 22 (SSH)
  - Port 80 (HTTP)
  - Port 443 (HTTPS)
  - Port 8080 (Custom TCP — application access)

> **Why m7i-flex.large:** Running 12 microservices alongside Dynatrace OneAgent and the Kubernetes operator requires a minimum of 6GB RAM. Smaller instance types (t3.small, t3.medium) were tested and caused resource contention — see Lessons Learned.

### 2. Connect to Your Instance

```bash
# Copy your .pem key to the Linux SSH directory
cp /path/to/your-key.pem ~/.ssh/your-key.pem
chmod 600 ~/.ssh/your-key.pem

# SSH into the instance
ssh -i ~/.ssh/your-key.pem ubuntu@YOUR_PUBLIC_IP
```

### 3. Install Docker

```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker ubuntu
newgrp docker
docker --version
```

### 4. Install minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

### 5. Install kubectl

```bash
sudo snap install kubectl --classic
```

### 6. Start Kubernetes Cluster

```bash
minikube start --driver=docker --cpus=2 --memory=6000
kubectl get nodes
```

Expected output:
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   32s   v1.35.1
```

### 7. Deploy Online Boutique

```bash
# Clone the repository
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo

# Download the official release manifest
curl -LO https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/main/release/kubernetes-manifests.yaml

# Remove CPU limits to accommodate single-node cluster
sed -i '/cpu:/d' kubernetes-manifests.yaml

# Deploy all 12 microservices
kubectl apply -f kubernetes-manifests.yaml

# Monitor deployment in real time
kubectl get pods --watch
```

> **Pro tip:** Use `--watch` instead of repeatedly running `kubectl get pods`. It automatically updates as pod statuses change — a more efficient and professional approach to monitoring deployments.

Wait until all pods show `1/1 Running` before proceeding.

### 8. Expose the Application

Open a second terminal and SSH into the instance, then run:

```bash
minikube tunnel
```

In the first terminal, forward the port:

```bash
kubectl port-forward service/frontend-external 8080:80 --address 0.0.0.0
```

Access the application at `http://YOUR_EC2_PUBLIC_IP:8080`

### 9. Install Dynatrace OneAgent

```bash
# Download the installer (replace with your environment-specific command from Dynatrace UI)
wget -O Dynatrace-OneAgent-Linux.sh "https://YOUR_ENV.live.dynatrace.com/api/v1/deployment/installer/agent/unix/default/latest?arch=x86" \
--header="Authorization: Api-Token YOUR_API_TOKEN"

# Install with full-stack monitoring
sudo /bin/sh Dynatrace-OneAgent-Linux.sh \
--set-monitoring-mode=fullstack \
--set-app-log-content-access=true
```

### 10. Install Dynatrace Kubernetes Operator

```bash
# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install the Dynatrace Operator
helm install dynatrace-operator oci://public.ecr.aws/dynatrace/dynatrace-operator \
--create-namespace \
--namespace dynatrace \
--atomic

# Apply your cluster configuration (download dynakube.yaml from Dynatrace UI)
kubectl apply -f dynakube.yaml

# Monitor operator deployment
kubectl get pods -n dynatrace --watch
```

---

## Screenshots

| # | Screenshot | Description |
|---|---|---|
| 1  | [EC2 Instance](screenshots/EC2%20instance.t3.small.%20screenshot.1.jpg) | EC2 instance running on AWS |
| 2  | [Security Group Rules](screenshots/Security.Inbound.Rules%20screenshot.2.jpg) | Inbound rules — SSH, HTTP, HTTPS, 8080 |
| 3  | [minikube Startup](screenshots/kubernetes%20milestones.%20screenshot.3.jpg) | Kubernetes cluster initializing |
| 4  | [kubectl get nodes](screenshots/kubectl.get.nodes.Ready.screenshot4.jpg) | Node showing Ready status |
| 5  | [All 12 Pods Running](screenshots/All.12.pods.running.Screenshot.5.jpg) | Online Boutique fully deployed |
| 6  | [minikube Tunnel](screenshots/minikube.tunnel.running.no.errors.Screenshot.6.jpg) | Network tunnel active, no errors |
| 7  | [Security Group Port 8080](screenshots/Updated.security.group.with.port8080.Screenshot.7.jpg) | Port 8080 opened for application access |
| 8  | [Online Boutique Live](screenshots/online.btq.live.screenshot8.jpg) | Application running in browser |
| 9  | [Dynatrace OneAgent](screenshots/Dynatrace.OneAgent.live.Screenshot.9.jpg) | OneAgent connected and fully operational |
| 10 | [Problems Dashboard](screenshots/Problem.Dashboard.Screenshot.10.jpg) | Davis AI detecting active problems |
| 11 | [Root Cause Analysis](screenshots/Davis.AI.root.cause.Screenshot.11.jpg) | CPU saturation — automated problem triage |
| 12 | [Dynatrace Operator](screenshots/Dynatrace.operator.live.screenshot12.jpg) | Kubernetes Operator deployed successfully |

---

## Lessons Learned

### 1. Instance Sizing Matters — Right-Size Before Deploying

The project was initially deployed on a `t3.small` (2 vCPU, 2GB RAM). Running 12 microservices alongside Dynatrace OneAgent and the Kubernetes operator consumed all available memory, causing `kubectl` commands to time out and SSH connections to drop.

**Diagnosis:** Used `free -h` to confirm only 148MB of 1.9GB RAM was available.

**Resolution:** Upgraded to `m7i-flex.large` (2 vCPU, 8GB RAM), allocating 6GB to minikube and reserving 2GB for the OS and monitoring tools.

**Takeaway:** Always baseline resource requirements before deploying. In production, resource planning happens at the architecture stage — not after deployment failures.

---

### 2. CPU Limits in Kubernetes Manifests Can Block Single-Node Deployments

The official Online Boutique `kubernetes-manifests/` folder uses local image references and CPU limits optimized for multi-node clusters. On a single-node minikube cluster, the total CPU requests across all 12 services exceeded the available capacity, leaving several pods stuck in `Pending`.

**Diagnosis:** Used `kubectl describe pod <pod-name>` which showed `Insufficient CPU` in the Events section.

**Resolution:** Used `sed -i '/cpu:/d'` to remove CPU limit constraints from the manifest file — the correct approach for single-node development clusters.

**Takeaway:** Kubernetes resource limits are designed for production multi-node environments. Development clusters require adjusted configurations. Always read the Events section of a `kubectl describe` output when a pod won't start.

---

### 3. Host-Level and Kubernetes-Level Monitoring Agents Cannot Coexist

Installing Dynatrace OneAgent directly on the EC2 host and then deploying the Dynatrace Kubernetes Operator created a conflict. The operator's OneAgent pod entered a `CrashLoopBackOff` because a host-level agent was already registered.

**Diagnosis:** Retrieved crash logs using `kubectl logs <pod-name> -n dynatrace --previous` which showed `"Agent was installed directly on the host and must be uninstalled before proceeding."`

**Resolution:** Uninstalled the host-level OneAgent using the official uninstall script before proceeding with the operator deployment.

**Takeaway:** In Kubernetes environments, the operator-managed OneAgent is the preferred approach. It provides deeper container-level visibility and is designed specifically for orchestrated environments. Host-level agents are for non-containerized workloads.

---

### 4. SSH Key Permissions Behave Differently Across Filesystems

When using Windows Subsystem for Linux (WSL), `.pem` key files stored in the Windows filesystem (`/mnt/c/...`) cannot have their permissions changed with `chmod`. SSH requires key files to have `600` permissions and will refuse to use a key that is too permissive.

**Resolution:** Copy the `.pem` file into the Linux filesystem (`~/.ssh/`) where `chmod` works correctly.

```bash
cp /mnt/c/Users/username/Downloads/key.pem ~/.ssh/key.pem
chmod 600 ~/.ssh/key.pem
```

**Takeaway:** WSL mounts Windows filesystems with fixed permissions controlled by Windows, not Linux. Always move security-sensitive files into the native Linux filesystem when working in WSL.

---

### 5. minikube Tunnel Is Required for LoadBalancer Services

Kubernetes `LoadBalancer` services show `EXTERNAL-IP: <pending>` on minikube because there is no cloud load balancer provisioning external IPs. In a real cloud Kubernetes cluster (EKS, GKE, AKS), this happens automatically.

**Resolution:** Running `minikube tunnel` in a separate terminal emulates a cloud load balancer and assigns an external IP to LoadBalancer services.

**Takeaway:** Understanding the difference between local and cloud Kubernetes behavior is critical for SE demonstrations. When showing Dynatrace to a customer on a local cluster, this step is required. In production deployments it happens automatically.

---

## Security Practices

All screenshots in this repository have been sanitized before publishing:

- AWS Account IDs redacted
- EC2 Instance IDs redacted
- Public and private IP addresses redacted
- Security group rule IDs redacted
- Dynatrace environment IDs redacted
- API tokens and authentication credentials never committed to the repository
- `.pem` key files excluded via `.gitignore`
- `dynakube.yaml` (contains Dynatrace tokens) excluded via `.gitignore`

---

## CI/CD Pipeline

A parallel CI/CD project (`observability-sandbox`) was built alongside this deployment to demonstrate automated pipeline concepts:

- GitHub Actions workflow triggers on every push to `main`
- Validates application files automatically
- Deploys to GitHub Pages with zero manual steps
- Monitored with UptimeRobot (5-minute interval uptime checks)

---

## Author

**GitHub:** [JD-Chin](https://github.com/JD-Chin)

---

*This project was built as a hands-on demonstration of cloud infrastructure, container orchestration, and enterprise observability platform deployment.*
