# Tenant Onboarding Process Documentation

This document describes the end-to-end flow for adding a new tenant (client) to the Idea 12 multi-tenant, multi-region observability platform.

---

## 1. Prerequisites

-   **Platform Admin** has write access to the platform’s GitOps repository and permission to approve PRs.
-   **Tenant** has an AWS (or other cloud) account and permission to create IAM roles or service principals.
-   **Tenant Operator** and **Control Plane** components (Argo CD, tenant-operator) are deployed in the platform.

---

## 2. Tenant Custom Resource Definition (CRD)

A `Tenant` CRD drives onboarding. It includes cloud-specific fields.

```yaml
apiVersion: platform.io/v1alpha1
kind: Tenant
metadata:
    name: acme-co # Unique tenant identifier
spec:
    cloudType: aws # One of: aws | gcp | azure
    awsAccountId: 123456789012
    region: us-west-2
    externalId: 8f4d2c1a-tenant-acme # Unique External ID for STS
```

Place this YAML in a new Git branch (e.g. `tenant-acme-onboard`) under `k8s-manifests/tenants/`.

---

## 3. IAM Role / Credentials Setup

### AWS

Provide the tenant with a CloudFormation or Terraform snippet to create an IAM role:

```yaml
# CloudFormation example
Resources:
    ObservabilityReadOnlyRole:
        Type: AWS::IAM::Role
        Properties:
            RoleName: ObservabilityReadOnly
            AssumeRolePolicyDocument:
                Version: "2012-10-17"
                Statement:
                    - Effect: Allow
                      Principal:
                          AWS: arn:aws:iam::PLATFORM_ACCOUNT_ID:root
                      Action: sts:AssumeRole
                      Condition:
                          StringEquals:
                              sts:ExternalId: EXTERNAL_ID
            Path: /
            Policies:
                - PolicyName: ReadOnlyMetricsLogs
                  PolicyDocument:
                      Version: "2012-10-17"
                      Statement:
                          - Effect: Allow
                            Action:
                                - cloudwatch:ListMetrics
                                - cloudwatch:GetMetricData
                                - cloudwatch:GetMetricStatistics
                                - logs:DescribeLogGroups
                                - logs:FilterLogEvents
                            Resource: "*"
```

Replace `PLATFORM_ACCOUNT_ID` and `EXTERNAL_ID` with the values from your `Tenant` CR.

### GCP & Azure

-   **GCP**: Tenant creates a Service Account with `roles/monitoring.viewer` and `roles/logging.viewer` and shares the key JSON.
-   **Azure**: Tenant creates a Service Principal with `Monitoring Reader` and `Log Analytics Reader` roles and uploads secret to Vault.

---

## 4. Operator Reconciliation

When the Tenant CR is merged:

1. **tenant-operator** detects the new CR.
2. **Creates** a Kubernetes namespace `tenant-acme` (or `acme-co`).
3. **Deploys** pull-based agents:

    - **Prometheus Agent** Deployment (or Pod) with:

        - `remote_write` to central Thanos
        - AWS/GCP/Azure exporter
        - Credentials via IRSA or mounted secret

    - **Fluent Bit** DaemonSet to collect logs and push to Loki.

4. **Updates** central control plane:

    - Adds Thanos scrape target label `tenant=acme-co`.
    - Creates a Grafana folder and pre-provisioned dashboards for `acme-co`.
    - Configures Alertmanager routing for `acme-co` (Slack/Email webhook).

---

## 5. Credential Management

Agents assume roles or authenticate periodically:

```bash
# AWS example
CREDS=$(aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/ObservabilityReadOnly \
  --external-id 8f4d2c1a-tenant-acme)
export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r .Credentials.AccessKeyId)
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r .Credentials.SecretAccessKey)
export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r .Credentials.SessionToken)
```

Agents refresh tokens automatically (every hour) via sidecar or built-in logic.

---

## 6. Tenant Usage

1. **Login** to Grafana (via Keycloak SSO).
2. **Select** your tenant folder/dashboard.
3. **View** metrics (Prometheus+Thanos) and logs (Loki).
4. **Create** or adjust alerts if permitted.
5. **Access** Vault-backed secrets in your applications via sidecars.

---

## 7. Offboarding

1. **Delete** the `Tenant` CR from Git (or branch).
2. **tenant-operator** cleans up:

    - Namespace
    - Agent Deployments
    - Grafana folder and dashboards
    - Thanos scrape config

3. **Data retention** honoured by Thanos/Loki retention policies.

---

_End of Tenant Onboarding Process_
