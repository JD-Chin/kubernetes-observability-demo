# Architecture Decisions

This document captures the system topology, service communication patterns, and key infrastructure decisions made during the deployment of this project. It is intended to provide deeper context beyond the setup guide in the README.

---

## System Topology

```
Internet
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  AWS Security Group                                         │
│  Port 22 (SSH) │ Port 80 (HTTP) │ Port 443 (HTTPS)          │
│  Port 8080 (Application)                                    │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  EC2 Instance: m7i-flex.large                               │
│  OS: Ubuntu 22.04 LTS                                       │
│  2 vCPU │ 8GB RAM │ 20GB gp3 storage                        │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Docker (container runtime)                           │  │
│  │                                                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │  minikube (single-node Kubernetes cluster)      │  │  │
│  │  │  Kubernetes v1.35.1 │ 6GB RAM allocated         │  │  │
│  │  │                                                 │  │  │
│  │  │  ┌─────────────────────────────────────────┐    │  │  │
│  │  │  │  default namespace (Online Boutique)    │    │  │  │
│  │  │  │                                         │    │  │  │
│  │  │  │  frontend (LoadBalancer: port 80/8080)  │    │  │  │
│  │  │  │  cartservice                            │    │  │  │
│  │  │  │  checkoutservice                        │    │  │  │
│  │  │  │  currencyservice                        │    │  │  │
│  │  │  │  emailservice                           │    │  │  │
│  │  │  │  paymentservice                         │    │  │  │
│  │  │  │  productcatalogservice                  │    │  │  │
│  │  │  │  recommendationservice                  │    │  │  │
│  │  │  │  shippingservice                        │    │  │  │
│  │  │  │  adservice                              │    │  │  │
│  │  │  │  redis-cart (database)                  │    │  │  │
│  │  │  │  loadgenerator (synthetic traffic)      │    │  │  │
│  │  │  └─────────────────────────────────────────┘    │  │  │
│  │  │                                                 │  │  │
│  │  │  ┌─────────────────────────────────────────┐    │  │  │
│  │  │  │  dynatrace namespace                    │    │  │  │
│  │  │  │                                         │    │  │  │
│  │  │  │  dynatrace-operator                     │    │  │  │
│  │  │  │  dynatrace-webhook (x2)                 │    │  │  │
│  │  │  │  dynatrace-oneagent-csi-driver          │    │  │  │
│  │  │  │  observability-sandbox-activegate       │    │  │  │
│  │  │  │  observability-sandbox-otel-collector   │    │  │  │
│  │  │  └─────────────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  Dynatrace SaaS Environment                                 │
│  OneAgent telemetry │ Davis AI problem detection            │
│  Kubernetes Operator metrics │ Service topology             │
└─────────────────────────────────────────────────────────────┘
```

---

## Service Communication Map

The 12 Online Boutique microservices communicate via gRPC internally. The frontend is the only service exposed externally.

```
Browser
  │
  ▼
frontend ──────────────────────────────────────────────────┐
  │                                                        │
  ├── cartservice ──── redis-cart (session storage)        │
  │                                                        │
  ├── productcatalogservice                                │
  │                                                        │
  ├── currencyservice                                      │
  │                                                        │
  ├── recommendationservice ──── productcatalogservice     │
  │                                                        │
  ├── adservice                                            │
  │                                                        │
  └── checkoutservice ─────────────────────────────────────┘
        │
        ├── cartservice
        ├── productcatalogservice
        ├── currencyservice
        ├── emailservice
        ├── paymentservice
        └── shippingservice

loadgenerator ──── frontend (synthetic traffic simulation)
```

**Protocol:** All inter-service communication uses gRPC over HTTP/2. The frontend exposes HTTP/1.1 to the browser via a LoadBalancer service on port 80 (forwarded to 8080 externally).

---

## Infrastructure Decisions

### Decision 1: AWS EC2 over Local Machine

**Options considered:**
- Local machine (PC or Mac)
- AWS EC2 instance

**Decision:** AWS EC2

**Reasoning:** Running on a cloud instance replicates real-world enterprise deployment conditions. It demonstrates cloud infrastructure skills directly relevant to Solutions Engineering roles, where customer environments are cloud-hosted. A local machine deployment would not translate to a credible portfolio piece in the same way.

---

### Decision 2: Ubuntu 22.04 LTS over Other OS Options

**Options considered:**
- Amazon Linux 2
- Ubuntu 22.04 LTS
- Ubuntu 24.04 LTS

**Decision:** Ubuntu 22.04 LTS

**Reasoning:** Ubuntu 22.04 LTS is the most widely documented Linux distribution for Kubernetes and Docker deployments. LTS (Long Term Support) means stability and a mature package ecosystem. Ubuntu 24.04 was too new at the time of deployment: some tools including Docker and Dynatrace OneAgent had not fully validated compatibility. Amazon Linux 2 was rejected due to fewer community resources for troubleshooting.

---

### Decision 3: m7i-flex.large Instance Type

**Options considered:**
- t3.micro (2 vCPU, 1GB RAM)
- t3.small (2 vCPU, 2GB RAM)
- t3.medium (2 vCPU, 4GB RAM)
- m7i-flex.large (2 vCPU, 8GB RAM)

**Decision:** m7i-flex.large

**Reasoning:** This decision was driven by empirical testing, not assumption. The project was initially deployed on t3.small. Memory diagnostics (`free -h`) revealed only 148MB of 1.9GB RAM available while running 12 microservices and Dynatrace OneAgent simultaneously: causing SSH timeouts and kubectl command failures.

t3.medium was evaluated but rejected: the t3 family shares CPU credits and has the same vCPU count across all sizes. The bottleneck was memory, not CPU. The m7i-flex.large provides 8GB RAM with a modern Intel processor optimized for general-purpose workloads: giving sufficient headroom for the full stack.

**Memory allocation:**
- 6GB allocated to minikube
- 2GB reserved for Ubuntu OS, Docker overhead, and Dynatrace OneAgent

---

### Decision 4: minikube over Other Kubernetes Distributions

**Options considered:**
- Amazon EKS (managed Kubernetes)
- k3s (lightweight Kubernetes)
- kind (Kubernetes in Docker)
- minikube

**Decision:** minikube

**Reasoning:** minikube is the most documented and widely supported local Kubernetes distribution. It integrates cleanly with Docker as a driver and has strong tooling support including `minikube tunnel` for LoadBalancer emulation. EKS was rejected due to cost and complexity for a single-node demo. k3s and kind were considered but minikube's tooling and documentation depth made it the lowest-friction choice for a portfolio project.

**Known limitation:** minikube running inside Docker on an EC2 instance creates a nested virtualization environment. This caused complications with the Dynatrace Kubernetes Operator's CSI storage driver, which expects direct host filesystem access. This is a known minikube-specific limitation and does not occur in production Kubernetes environments (EKS, GKE, AKS).

---

### Decision 5: Google Online Boutique over Other Sample Applications

**Options considered:**
- Dynatrace EasyTravel (standalone application)
- Dynatrace EasyTrade (Kubernetes application)
- Kona Kart
- Google Online Boutique (microservices-demo)

**Decision:** Google Online Boutique

**Reasoning:** Online Boutique is a production-grade microservices reference architecture maintained by Google. It includes all the characteristics required for a meaningful observability demonstration: a browser-based frontend, HTTP service calls to backend services, database calls via Redis, and a load generator capable of producing errors and traffic patterns. It also uses real-world technologies (gRPC, Docker, Kubernetes manifests) rather than simplified demo tooling: making it more relevant as a portfolio piece.

---

### Decision 6: Dynatrace Kubernetes Operator over Host-Level OneAgent

**Options considered:**
- Dynatrace OneAgent installed directly on EC2 host
- Dynatrace Kubernetes Operator

**Decision:** Kubernetes Operator (after initial host-level installation)

**Reasoning:** The project initially installed OneAgent directly on the EC2 host. This provided basic infrastructure monitoring but conflicted with the Kubernetes Operator deployment: the operator's OneAgent pod entered CrashLoopBackOff because a host-level agent was already registered.

The Kubernetes Operator is the correct approach for containerized environments. It provides: container-level visibility, Kubernetes-native monitoring, automatic service discovery within namespaces, and distributed tracing across microservices. The host-level agent was uninstalled and replaced with the operator-managed approach.

---

### Decision 7: Helm for Operator Installation

**Options considered:**
- kubectl apply with raw YAML
- Helm chart installation

**Decision:** Helm

**Reasoning:** Helm is the industry-standard package manager for Kubernetes applications. Using Helm to install the Dynatrace Operator demonstrates familiarity with real-world Kubernetes tooling. It also provides cleaner version management, atomic installation (automatic rollback on failure via `--atomic`), and is the installation method recommended by Dynatrace's official documentation.

---

## Security Architecture

### Network Security

All traffic enters through AWS Security Group rules configured with the principle of least privilege: only the ports required for the project are open.

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH access for administration |
| 80 | TCP | HTTP traffic |
| 443 | TCP | HTTPS traffic |
| 8080 | TCP (IPv4 + IPv6) | Application access via port-forward |

### Credential Management

No secrets, tokens, or credentials are committed to this repository. The following files are excluded via `.gitignore`:

- `dynakube.yaml`: contains Dynatrace environment ID and API tokens
- `*.pem`: SSH private key files

All screenshots were manually reviewed and sanitized before publishing: AWS account IDs, instance IDs, IP addresses, security group rule IDs, and Dynatrace environment IDs were redacted using solid black overlays.

---

## Observability Layer

Dynatrace monitors the environment at multiple layers simultaneously:

| Layer | What Dynatrace monitors |
|---|---|
| Host | CPU, memory, disk, network on the EC2 instance |
| Container | Individual pod resource consumption, restarts, status |
| Application | Service response times, error rates, throughput |
| Network | Inter-service communication, latency between pods |
| Kubernetes | Pod scheduling, deployment health, namespace status |

**Davis AI** analyzes all telemetry continuously and automatically detects anomalies without manual alert configuration. During this deployment, Davis detected CPU saturation and memory saturation within minutes of OneAgent installation: identifying the affected host and classifying severity automatically.

---

*Last updated: May 2026*
*Author: [JD-Chin](https://github.com/JD-Chin)*
