# Project Idea 12: Multi-Tenant, Multi-Region Observability Platform

## 1. Objective

Design and implement a **multi-tenant**, **multi-region** observability platform on **AWS**, using **Kubernetes** and tools like **Prometheus**, **Grafana**, **Thanos**, **Loki**, **Istio**, **Vault**, **Keycloak**, **OPA/Gatekeeper**, and optionally **Crossplane**. The platform must be GitOps-enabled with **GitHub Actions** and **Argo CD**, and support extensibility through a custom **Kubernetes Operator**.

---

## 2. Key Features

-   **Multi-Tenant Support**: Logical isolation of tenants via Kubernetes namespaces and Istio's tenancy model.
-   **Multi-Region Architecture**: Centralized observability control plane and federated data plane.
-   **GitOps-first**: All infra/app configurations managed via GitHub + Argo CD.
-   **Security**:

    -   Auth via Keycloak.
    -   Secrets via HashiCorp Vault.
    -   Policies via OPA/Gatekeeper.

-   **Cost-optimized Infrastructure as Code** via Terraform and GitHub Actions.
-   **Custom Kubernetes Operator** for tenant lifecycle management.

---

## 3. Tools and Technologies

| Category         | Tool/Service                      |
| ---------------- | --------------------------------- |
| Cloud            | AWS (multi-region)                |
| IaC              | Terraform                         |
| GitOps           | GitHub Actions, Argo CD           |
| Kubernetes       | EKS, KIND (local)                 |
| Observability    | Prometheus, Grafana, Thanos, Loki |
| Service Mesh     | Istio                             |
| AuthN/AuthZ      | Keycloak, OPA/Gatekeeper          |
| Secrets          | HashiCorp Vault                   |
| Policy Control   | OPA (Gatekeeper)                  |
| Storage          | Rook-Ceph                         |
| Networking       | Istio, OpenVPN                    |
| Operator SDK     | Golang, kube-builder              |
| Security Agent   | Wazuh (Optional)                  |
| Network Security | Cilium (Optional)                 |

---

## 4. Architecture Overview

The system has:

-   **Global Control Plane**:

    -   Argo CD
    -   Central Grafana (multi-tenant dashboards)
    -   Thanos Query + Store
    -   Vault & Keycloak
    -   Operator for tenant onboarding

-   **Regional Data Planes (EKS)**:

    -   Prometheus (scrapers per tenant)
    -   Loki + FluentBit
    -   Istio for service mesh and multi-tenant routing
    -   Vault Agent Injectors
    -   Gatekeeper for policies

---

## 5. AWS Infrastructure (Single Cloud Multi-Region)

### Per Region:

-   VPC + Subnets (private/public)
-   EKS Cluster
-   Rook-Ceph Storage
-   NAT Gateway + IGW
-   OpenVPN for secure cluster communication
-   IAM Roles for Service Accounts (IRSA)
-   Route53 Private Zones (optional)

### 🔁 AWS Transit Gateway

-   Central hub-and-spoke architecture connecting VPCs across regions.
-   Configured with:

    -   TGW Attachments for each regional VPC
    -   Route table propagation for dynamic VPC routing
    -   Optional peering with on-prem via Direct Connect

-   Enables secure multi-region observability and tenant access control.

---

## 6. CI/CD Design

-   **CI**: GitHub Actions builds and validates Terraform, Kubernetes YAML, Helm charts.
-   **CD**: Argo CD watches `main/dev/stage` branches and applies infra/app changes.
-   **Envs**: Branch-based isolation:

    -   `main` → Production AWS account
    -   `dev` → Dev AWS account
    -   `stage` → Staging AWS account

---

## 7. GitHub Actions + Terraform

-   Each push to environment-specific branch applies Terraform using GitHub Actions.
-   Uses dynamic secret loading based on branch prefix (e.g., `DEV_AWS_ACCESS_KEY`).
-   On branch deletion, a Terraform Destroy job runs (for non-prod only).
-   Secrets stored in GitHub and loaded per team/env securely.

---

## 8. Argo CD GitOps

-   Argo CD deployed in central EKS control plane.
-   App-of-Apps pattern to bootstrap applications across regions.
-   Separate applications for:

    -   Observability stack (Thanos, Prometheus, Grafana)
    -   Istio config
    -   Vault Agent Injectors
    -   Tenant-specific dashboards

---

## 9. Istio Integration

-   Used for service discovery, traffic routing, and tenant isolation.
-   Ingress Gateway supports multi-tenant routing via JWT claims from Keycloak.
-   mTLS enabled between microservices.
-   Istio AuthorizationPolicies restrict tenant access at the namespace level.

---

## 10. Kubernetes Custom Operator

### Name: `tenant-operator`

#### Functions:

-   Watches `Tenant` CRDs.
-   On new tenant:

    -   Creates namespace
    -   Applies Vault Secrets
    -   Generates Istio Gateway and VirtualService
    -   Adds Prometheus scrape config
    -   Creates Grafana dashboard folder + datasource

-   On tenant deletion:

    -   Tears down all above components

#### Tools:

-   Written in **Go**
-   Uses **Kubebuilder** or **Operator SDK**
-   GitHub Actions CI pipeline for operator

---

## 11. Security

-   **Vault Agent Sidecars** inject secrets into pods.
-   **Keycloak** integrated with GitHub SSO and Istio JWT validation.
-   **OPA/Gatekeeper** enforces:

    -   Image policies
    -   Resource quotas per tenant
    -   Namespace-level RBAC

-   **Cilium** (optional) adds L7-aware network policies.
-   **Wazuh** (optional) agent monitors node security.

---

## 12. Monitoring & Logging

-   **Prometheus** per region with remote-write to Thanos.
-   **Grafana** with folders per tenant, sourced from Thanos.
-   **Loki** for logs via FluentBit.
-   Alerts via **Alertmanager** routed to tenant-specific endpoints.

---

## 13. Additional Enhancements

-   **Crossplane** could optionally be used to provision AWS infrastructure via Kubernetes CRDs (future extension).
-   Add **eBPF** telemetry through Cilium or Pixie (for deep observability).
-   Self-service tenant onboarding via `tenant-operator` and GitHub PR workflow.

---

## 14. Directory Structure Example

```plaintext
repo-root/
├── .github/
│   └── workflows/
│       ├── terraform-dev.yml
│       ├── terraform-prod.yml
│       └── tenant-operator-ci.yml
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── terraform/
│   ├── network/
│   ├── eks/
│   ├── vault/
│   └── observability/
├── helm-charts/
│   └── custom-apps/
├── operators/
│   └── tenant-operator/
└── k8s-manifests/
    ├── argocd/
    ├── istio/
    └── tenants/

```
