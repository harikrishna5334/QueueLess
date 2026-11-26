# ADR-009: Local Kubernetes First (Kind / k3s)

## Status
**Accepted** (Approved by Project Owner - 2026-11-02)

## Context
QueueLess aims to provide complete DevOps and container orchestration coverage, culminating in Phase 12 (Kubernetes).
However, cloud container orchestration poses significant cost and accessibility challenges:
1. **Cloud Cost Burden:** Provisioning managed Kubernetes clusters such as AWS EKS incurs substantial fixed hourly baseline costs:
   - EKS control plane: ~$0.10/hour (~$73/month) regardless of cluster usage.
   - EC2 worker nodes (e.g., 2 x `t3.medium`): ~$60/month.
   - AWS Network Load Balancers / ALBs, NAT Gateways, and EBS volumes add further monthly overhead.
2. **Project Constitution Mandate:** Sections 43, 44, and 45 of `AGENTS.md` explicitly enforce:
   - "Local Kubernetes is sufficient for all core Kubernetes learning objectives."
   - "Prefer local development and free/low-cost alternatives when they adequately support the learning objective."
   - "Never run Terraform apply or equivalent cloud provisioning without explicit developer approval."
3. **Core Educational Goals:** The core principles of Kubernetes -- declarative manifests, Pods, Deployments, Services, ConfigMaps, Secrets, liveness/readiness probes, resource limits, rolling updates, ingress routing, and Helm packaging -- can be mastered completely without paying cloud provider fees.

## Decision
We mandate a **Local Kubernetes First** architecture for **Phase 12**:

1. **Local Tooling Selection:**
   - The primary development and learning platform is **Kind (Kubernetes in Docker)** or **k3s/k3d**.
   - Kind runs multi-node clusters inside Docker containers on the local machine with zero recurring cloud cost.
2. **Local Deliverables:**
   - Complete declarative manifest repository and Helm charts covering:
     - API Deployment, ReplicaSet, and Horizontal Pod Autoscaler (HPA).
     - Frontend static web server Deployment.
     - ClusterIP and NodePort Services.
     - Ingress Controller configuration using NGINX Ingress Controller or Traefik.
     - ConfigMaps and Secrets for environment parameters.
     - Liveness and readiness probes tied to application health endpoints (`/health/liveness`, `/health/readiness`).
     - Resource requests and limits (CPU, memory) tuned for deterministic scheduling.
3. **Managed Cloud Deployment Policy:**
   - Managed cloud Kubernetes (AWS EKS via Terraform) is strictly optional and secondary.
   - If executed, cloud deployment must:
     - Obtain prior explicit owner approval.
     - Use the smallest practical configuration (single-zone, spot or minimal `t3.small` nodes).
     - Execute verified teardown scripts immediately upon completion of testing to avoid lingering billable resources.

## Alternatives Considered
- **Cloud-First Managed EKS:** Rejected due to non-negotiable monthly baseline costs. Forcing a developer or student to pay ~$150+/month just to test basic Pods and Services contradicts the project philosophy.
- **Minikube:** Evaluated as an alternative; however, Kind and k3s/k3d offer superior multi-node simulation, faster cluster spin-up, and native Docker socket integration without hypervisor overhead.

## Consequences
### Positive
- $0 cloud infrastructure expense for all core Kubernetes learning milestones.
- Extremely fast cluster spin-up, tear-down, and reset cycles during development.
- Offline and local development capabilities without internet connectivity dependencies.
- Helm charts and manifests developed for Kind remain directly portable to EKS, GKE, or AKS with minor ingress annotations adjustments.

### Liabilities
- Local ingress requires manual port forwarding or `cloud-provider-kind` emulation rather than managed cloud ALBs.
- Cloud-specific features such as AWS IAM Roles for Service Accounts (IRSA) or EBS CSI Driver storage classes cannot be fully tested locally and must be learned via local emulation (LocalStack) or short-lived, explicitly budgeted cloud sessions.

