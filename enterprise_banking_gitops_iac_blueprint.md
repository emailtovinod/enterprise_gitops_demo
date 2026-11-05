# Enterprise Prescriptive Blueprint: End-to-End Infrastructure & Application GitOps in a Regulated Banking Environment

**Document Classification:** Tier-1 Enterprise Architecture & Security Governance Blueprint  
**Target Stack:** GitHub Enterprise, ArgoCD, Terraform / OpenTofu (Terragrunt), Crossplane, HashiCorp Vault, Kyverno  


## 1. Executive Summary & Architectural Philosophy

In a Tier-1 banking environment, deploying infrastructure and applications requires uncompromising adherence to **Segregation of Duties (SoD)**, **Cryptographic Non-Repudiation**, **Zero-Trust Execution**, and **Blast-Radius Containment**. 

This blueprint establishes a unified, end-to-end GitOps Operating Model across both **Infrastructure as Code (IaC)** and **Application Workloads**. It eliminates human write access to cloud consoles and Kubernetes clusters ("ClickOps" and manual `kubectl apply`), replacing them with an auditable, deterministic, pull-request-driven reconciliation loop.

### Core Security Tenets
1. **Maker-Checker Enforcement:** No individual engineer can author and approve their own infrastructure or application change in UAT or Production.
2. **Zero Static Credentials:** Neither human engineers nor CI/CD pipelines possess static cloud IAM keys, database passwords, or long-lived Kubernetes `kubeconfig` files.
3. **Immutable Promotion:** Infrastructure modules, container images, and execution plans are versioned, cryptographically signed, and promoted by digest/hash across environments.
4. **Continuous Drift Remediation:** Unauthorized out-of-band mutations in the cloud control plane or Kubernetes cluster are automatically detected, alerted to the SIEM, and reverted to the Git-declared state.

---

## 2. Two-Layer GitOps Boundary: Infrastructure vs. Applications

To prevent monolithic state failures and privilege escalation, the bank's architecture is bifurcated into two distinct operational layers connected by an automated hand-off bridge:

| Architectural Layer | Scope & Managed Assets | Tooling Stack | Execution Engine | Reconciliation Cadence |
| :--- | :--- | :--- | :--- | :--- |
| **Layer 1: Foundational Infrastructure ("Day-0 / Day-1")** | Cloud Landing Zones, Hub-Spoke VPCs/VNets, DirectConnect/MPLS, Firewalls, HSM/KMS Keys, Shared Core Databases, and Base K8s Clusters (EKS/AKS/OCP). | **Terraform / OpenTofu** orchestrated via **Terragrunt** | **GitHub Actions** (Self-Hosted Hardened Runners in Mgmt Zone) | **PR-Driven Plan/Apply** + Scheduled 4-Hour Drift Detection |
| **Layer 2A: Platform Baseline & Add-ons ("Day-1.5")** | Namespaces, NetworkPolicies, Istio Service Mesh, Kyverno, External Secrets Operator, Falco/CrowdStrike, Crossplane Providers. | **Helm + Kustomize** via ArgoCD `ApplicationSet` | **ArgoCD** (Hub-and-Spoke Control Plane) | **Continuous Pull Loop** (Every 3 mins + Webhook) |
| **Layer 2B: Application Workloads & App-Scoped Infra ("Day-2")** | Microservice Deployments, HPA, ConfigMaps, and App-specific Cloud Resources (S3 Buckets, SQS Queues, App RDS via Crossplane). | **Kustomize / Helm** + **Crossplane Claims (XRDs)** | **ArgoCD** (Scoped per Business Tribe `AppProject`) | **Continuous Pull Loop** (Real-time `selfHeal` & `prune`) |

### The "Terraform-to-ArgoCD" Zero-Touch Hand-Off Bridge
When Layer 1 Terraform provisions or updates a Kubernetes cluster, its final action is to register the cluster's API endpoint and mTLS credentials as a Kubernetes `Secret` inside the Hub ArgoCD namespace, enriched with metadata labels:
```yaml
metadata:
  labels:
    argocd.argoproj.io/secret-type: cluster
    bank.com/environment: "prod-pci"
    bank.com/region: "ap-south-1"
    bank.com/dc-role: "primary"
    bank.com/business-unit: "payments"
```
ArgoCD's **Cluster Generator (`ApplicationSet`)** immediately detects these labels and bootstraps all Layer 2A security agents, admission controllers, and namespace guardrails without human intervention.

---

## 3. Enterprise Five-Tier Repository Topology

Never mix application source code, Kubernetes manifests, and Terraform infrastructure definitions in a single repository. Doing so violates Segregation of Duties and complicates audit scoping. Enforce a **Five-Tier Repository Model**:

| Repo Tier | Naming Convention | Contents | Primary Owner |
| :--- | :--- | :--- | :--- |
| **1. IaC Golden Modules** | `iac-modules-registry` | Hardened, reusable Terraform modules with mandatory KMS encryption, private endpoints, and logging baked in. | Cloud Security & Platform Architecture |
| **2. Live Foundational IaC** | `iac-live-foundation` | Terragrunt environment configurations (`00-network`, `10-security`, `20-data`, `30-compute-k8s`). | Cloud Platform / Network / SecOps |
| **3. Platform GitOps Baseline** | `platform-gitops-baseline` | ArgoCD `AppProjects`, `ApplicationSets`, Cluster Add-ons, Kyverno Policies, Crossplane `Composition` templates. | Platform SRE & Kubernetes SecOps |
| **4. Application Source Code** | `app-source-<tribe>-<svc>` | Java/Go/Node source code, unit tests, `Dockerfile`. **Zero K8s environment manifests.** | Application Engineering Tribes |
| **5. Application GitOps Config** | `app-config-<tribe>` | Base Kustomize manifests, environment overlays (`dev`, `uat`, `prod`), and Crossplane Infrastructure Claims. | App Tech Leads, Release Managers, CAB |

---

## 4. Branching & Promotion Strategy (Trunk-Based Directory Promotion)

**Mandatory Standard:** Long-lived environment branches (`dev`, `uat`, `prod` branches) are **strictly prohibited** across both IaC and Application Config repositories. Merging branches across environments causes configuration drift and accidental overwrites of environment-specific parameters (such as HSM partition IDs, firewall CIDRs, or replica counts).

All state repositories (`iac-live-foundation`, `platform-gitops-baseline`, and `app-config-*`) operate on a **Single Protected Trunk Branch (`main`)** using **Directory-Based Isolation**.

### A. Infrastructure Repository Layout (`iac-live-foundation`)
Every directory represents an **isolated Terraform state file (`terraform.tfstate`)** and an isolated blast radius:
```text
iac-live-foundation/
├── _envcommon/                        # Shared Terragrunt base configurations
├── non-prod/
│   └── ap-south-1-mumbai/
│       ├── dev/
│       │   ├── 00-network/            # State 1: VPC, Subnets, Transit Gateway
│       │   ├── 10-security/           # State 2: KMS, HSM links, WAF
│       │   ├── 20-data-shared/        # State 3: Shared Kafka/MSK, Enterprise Redis
│       │   └── 30-compute-k8s/        # State 4: EKS/AKS/OCP Cluster & NodeGroups
│       └── uat/
│           └── ...
└── prod/
    ├── ap-south-1-mumbai-primary/
    │   └── prod-pci/
    │       ├── 00-network/
    │       ├── 10-security/
    │       ├── 20-data-shared/
    │       └── 30-compute-k8s/
    └── ap-south-2-hyderabad-dr/
        └── prod-pci/
            └── ...
```

### B. Application GitOps Repository Layout (`app-config-payments`)
```text
app-config-payments/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── environments/
    ├── dev/
    │   ├── kustomization.yaml         # Pins image SHA256 digest for Dev
    │   └── infra-claims.yaml          # Crossplane DB/Queue claim for Dev
    ├── uat/
    │   ├── kustomization.yaml         # Pins verified image SHA256 digest for UAT
    │   └── patch-resources.yaml
    ├── prod-mumbai-primary/
    │   ├── kustomization.yaml         # Pins CAB-approved SHA256 digest for Prod Primary
    │   └── patch-hsm-mounts.yaml
    └── prod-hyderabad-dr/
        ├── kustomization.yaml         # Mirrors Primary digest for Active-Active/Warm-Standby
        └── patch-hsm-mounts.yaml
```

### C. End-to-End Promotion Workflow
1. **Infrastructure Promotion:**
   * Changes to `iac-modules-registry` are tested and released as a signed semantic version tag (e.g., `v2.4.0`).
   * A PR updates `non-prod/.../dev/30-compute-k8s/terragrunt.hcl` to reference `?ref=v2.4.0`.
   * After validation in `dev` and `uat`, a promotion PR bumps the tag reference in `prod/.../30-compute-k8s/terragrunt.hcl`.
2. **Application Promotion:**
   * A merge to `app-source` triggers CI (GitHub Actions), which runs SAST/SCA scans, builds the immutable container image, generates a CycloneDX SBOM, signs both with **Cosign**, and pushes the image by `@sha256:...` digest to the bank's internal air-gapped container registry.
   * CI automatically opens a PR in `app-config-payments` updating `/environments/dev/kustomization.yaml`.
   * Promoting to `uat` and `prod` consists strictly of opening a PR that promotes the exact `@sha256` digest across directory folders on `main`.

---

## 5. Unified Enterprise RBAC & Segregation of Duties (SoD) Matrix

All authentication across GitHub Enterprise, ArgoCD, Cloud Portals, and HashiCorp Vault is federated via **Corporate IdP (Entra ID / Okta) using OIDC/SAML** with phishing-resistant MFA (FIDO2/YubiKey).

### A. End-to-End Persona Access Matrix

| Banking Persona | `iac-live-foundation` (GitHub) | `app-config-*` (GitHub) | ArgoCD Access (UI / CLI / API) | Cloud Console & Direct `kubectl` Access |
| :--- | :--- | :--- | :--- | :--- |
| **Application Developer (Maker)** | `None` | `Write` (Create branches & PRs; blocked from merging to `main`) | `role:readonly` in `dev`/`uat`; `None` in `prod` | `View` pod logs in `dev` only; **Zero access** in `uat`/`prod` |
| **App Tech Lead / QA Lead (Non-Prod Checker)** | `None` | `Maintain` (`CODEOWNERS` for `/environments/dev/` & `/uat/`) | `role:nonprod-deployer` (`get`, `sync` on `dev`/`uat` projects) | **Zero write access** (Read-only telemetry via Grafana/SIEM) |
| **Network Security Engineer** | `Write` + `CODEOWNERS` on `*/00-network/*` | `None` | `None` | **Zero write access** (Read-only VPC/Firewall diagnostics) |
| **Cloud SecOps / IAM Engineer** | `Write` + `CODEOWNERS` on `*/10-security/*` & `iac-modules` | `Read` | `role:secops-auditor` (`get` across all projects) | **Zero write access** (Manages KMS/Vault policies via GitOps) |
| **Platform / SRE Engineer** | `Write` + `CODEOWNERS` on `*/30-compute-k8s/*` | `Maintain` on `/base/` | `role:platform-admin` (Manages `AppProjects`, clusters, add-ons) | **JIT Break-Glass Only** via CyberArk/PIM with session recording |
| **FinOps Governance Lead** | `CODEOWNERS` (Triggered on high cost delta PRs) | `Read` | `None` | Read-only Billing & Cost Explorer dashboards |
| **CAB / Prod Release Manager** | `CODEOWNERS` on `/prod/**` | `CODEOWNERS` on `/environments/prod-*/` | `role:prod-operator` (`get`, `sync` during active Change Windows) | **None** |
| **Internal / External Auditor (RBI / PCI)** | `Read` (Full access to code, PRs, Action runs) | `Read` | `role:auditor` (Read-only across all apps, logs, events) | Read-only access to SIEM, CloudTrail, and Compliance Dashboards |

### B. Enforcing Granular Folder-Level SoD in GitHub (`CODEOWNERS`)

GitHub Branch Protection Rules on `main` must be configured with:
* **Require a pull request before merging** (Minimum 2 approvals for Prod paths).
* **Require review from Code Owners.**
* **Dismiss stale pull request approvals when new commits are pushed.**
* **Require approval of the most recent reviewable push** (Strictly prevents the PR author or last pusher from approving their own changes).
* **Require signed commits.**

#### 1. `.github/CODEOWNERS` for `iac-live-foundation`
```text
# Global fallback - Platform Architecture & SecOps
*                                       @bank-org/platform-architects @bank-org/cloud-secops

# Layer-Specific Segregation of Duties
**/00-network/**                        @bank-org/network-security-leads
**/10-security/**                       @bank-org/cloud-secops-leads @bank-org/cryptography-team
**/20-data-shared/**                    @bank-org/enterprise-dba-leads @bank-org/platform-sre
**/30-compute-k8s/**                    @bank-org/platform-sre-leads

# Production Mandatory Dual-Gate (Requires Domain Owner + CAB Release Management)
/prod/**/00-network/**                  @bank-org/network-security-leads @bank-org/cab-infra-approvers
/prod/**/10-security/**                 @bank-org/cloud-secops-leads @bank-org/ciso-delegates
/prod/**/20-data-shared/**              @bank-org/enterprise-dba-leads @bank-org/cab-infra-approvers
/prod/**/30-compute-k8s/**              @bank-org/platform-sre-leads @bank-org/cab-infra-approvers
```

#### 2. `.github/CODEOWNERS` for `app-config-payments`
```text
# Base templates require Tech Lead & Platform SRE review
/base/**                                @bank-org/payments-tech-leads @bank-org/platform-sre

# Environment-Specific Approvals
/environments/dev/**                    @bank-org/payments-senior-devs @bank-org/payments-tech-leads
/environments/uat/**                    @bank-org/payments-qa-leads @bank-org/payments-tech-leads
/environments/prod-**                   @bank-org/payments-vp-engineering @bank-org/cab-release-managers
```

---

## 6. Infrastructure as Code (IaC) Security Guardrails & Execution Pipeline

### A. PR-Driven Execution & Policy-as-Code Gate
Human engineers cannot execute `terraform apply` from local workstations. All execution happens inside ephemeral, air-gapped GitHub Actions Runners:

1. **PR Opened (`terraform plan`):**
   * Runner authenticates to the Cloud Provider via **short-lived OIDC token** mapped to a **Read-Only `Plan` IAM Role** (incapable of mutating state or infrastructure).
   * Runs **Checkov / Trivy** for static misconfiguration analysis.
   * Generates `tfplan.binary` and converts it to `tfplan.json`.
   * Evaluates `tfplan.json` against **Open Policy Agent (OPA)** guardrails:
     ```rego
     package bank.terraform.guardrails

     # Block deletion of stateful production databases or encryption keys
     deny[msg] {
       resource := input.resource_changes[_]
       resource.Mode == "managed"
       resource.type == {"aws_db_instance", "aws_rds_cluster", "aws_kms_key", "azurerm_mssql_database"}[_]
       resource.change.actions[_] == "delete"
       msg := sprintf("CRITICAL POLICY VIOLATION: Deletion of stateful resource '%v' is blocked via CI. Requires Break-Glass Change Procedure.", [resource.address])
     }

     # Enforce mandatory banking compliance tags on all resources
     mandatory_tags := {"CostCenter", "DataClassification", "BusinessUnit", "PCI-Scope", "GitRepo"}
     deny[msg] {
       resource := input.resource_changes[_]
       resource.change.actions[_] != "delete"
       provided_tags := {tag | resource.change.after.tags_all[tag]}
       missing := mandatory_tags - provided_tags
       count(missing) > 0
       msg := sprintf("COMPLIANCE VIOLATION: Resource '%v' is missing mandatory tags: %v", [resource.address, missing])
     }
     ```
   * **Infracost Gate:** Posts the monthly cost delta to the PR. If the delta exceeds threshold, triggers a mandatory review from `@bank-org/finops-approvers`.
   * Cryptographically hashes `tfplan.binary` (SHA256) and uploads it to immutable artifact storage tied to the PR commit SHA.

2. **Merge to `main` (`terraform apply`):**
   * Requires passing GitHub Environment Protection Rules (CAB ServiceNow webhook verification + `CODEOWNERS` sign-off).
   * Runner authenticates via **OIDC** to a layer-scoped **Read-Write `Apply` IAM Role** (e.g., the `30-compute-k8s` runner role only has IAM permissions for EKS/EC2/IAM-NodeRoles; it cannot modify VPC route tables or KMS keys).
   * Downloads the exact approved `tfplan.binary`, verifies its SHA256 hash, and executes `terraform apply tfplan.binary`.

### B. Tier-0 State File Protection
Terraform state files contain sensitive resource metadata and must be protected as **Tier-0 Confidential Data**:
* **Dedicated State Backends per Environment & Layer:** Non-Prod and Prod never share an S3 bucket / Azure Storage Account.
* **HSM-Backed Encryption:** State buckets enforce Server-Side Encryption using Customer-Managed Keys (SSE-KMS) backed by FIPS 140-3 Level 3 HSMs.
* **State Locking & WORM Versioning:** Enable DynamoDB/Blob Lease locking, Object Versioning, and **Object Lock (Governance/Compliance WORM mode)** on access audit logs.
* **Network Isolation:** State bucket policies enforce `aws:SourceVpce` (VPC Endpoint restriction), rejecting any access attempt outside the dedicated GitHub Runner subnet.

### C. Self-Service App Infrastructure via Crossplane (Layer 2B)
To prevent application teams from touching Terraform state while still enabling fast provisioning of app-scoped cloud resources (S3 buckets, SQS queues, dedicated RDS databases):
1. Platform Engineering publishes hardened **Crossplane `CompositeResourceDefinitions` (XRDs)** in `platform-gitops-baseline` (pre-configured with KMS encryption, private endpoints, and backup policies).
2. Application teams commit a declarative Kubernetes manifest (`BankPostgreSQLInstance` or `BankObjectBucket`) inside their `app-config-<tribe>` repo.
3. ArgoCD syncs the claim to the cluster, and Crossplane provisions the cloud resource using least-privilege workload identity.

---

## 7. ArgoCD Hardening, Multi-Tenancy & Runtime Guardrails

### A. Hub-and-Spoke Air-Gapped Topology
* **Environment Isolation:** Deploy physically separate ArgoCD Control Plane instances for **Non-Production** and **Production/PCI-DSS**. Never connect a Non-Prod ArgoCD instance to a Production Kubernetes cluster.
* **Hardened Execution:** Disable the ArgoCD `admin` local user (`admin.enabled: "false"` in `argocd-cm`), disable `exec` web terminal in Production (`exec.enabled: "false"`), and run ArgoCD Repo-Server with read-only root filesystems and restricted egress.

### B. Production ArgoCD RBAC (`argocd-rbac-cm.yaml`)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:none # Zero-Trust default: explicit deny
  scopes: '[groups, email]'
  policy.csv: |
    # Global Internal/External Auditor Role (Read-Only)
    p, role:auditor, applications, get, */*, allow
    p, role:auditor, projects, get, *, allow
    p, role:auditor, repositories, get, *, allow
    p, role:auditor, logs, get, */*, allow
    g, "IdP-Group-Bank-Auditors", role:auditor
    g, "IdP-Group-Cloud-SecOps", role:auditor

    # Payments Tribe - Non-Prod Deployer
    p, role:payments-nonprod, applications, get, payments-nonprod/*, allow
    p, role:payments-nonprod, applications, sync, payments-nonprod/*, allow
    g, "IdP-Group-Payments-TechLeads", role:payments-nonprod

    # Payments Tribe - Production CAB Operator (Sync only; no delete or parameter override)
    p, role:payments-prod-op, applications, get, payments-prod/*, allow
    p, role:payments-prod-op, applications, sync, payments-prod/*, allow
    p, role:payments-prod-op, applications, delete, payments-prod/*, deny
    p, role:payments-prod-op, applications, override, payments-prod/*, deny
    p, role:payments-prod-op, exec, create, payments-prod/*, deny
    g, "IdP-Group-CAB-Release-Managers", role:payments-prod-op

    # Platform SRE Admin
    g, "IdP-Group-Platform-SRE-Leads", role:admin
```

### C. Production Multi-Tenant Boundary (`AppProject` Manifest)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: "Tier-1 Production Payments Workloads"
  # 1. Restrict allowed Git repositories
  sourceRepos:
    - 'https://github.internal.bank.com/payments-tribe/app-config-payments.git'
  
  # 2. Enforce Cryptographic Commit Verification (Only sync GPG-signed commits)
  signatureKeys:
    - keyID: '4AEE18F83AFDEB23' # GitHub Enterprise Web-Flow Signing Key
    - keyID: 'B5690EEEBB952194' # Payments Release Bot Hardware Key

  # 3. Restrict target clusters and namespaces
  destinations:
    - namespace: 'payments-prod-*'
      server: 'https://prod-mumbai-k8s.internal.bank.com:6443'
    - namespace: 'payments-prod-*'
      server: 'https://prod-hyderabad-k8s.internal.bank.com:6443'

  # 4. Deny all cluster-scoped resources to prevent tenant privilege escalation
  clusterResourceWhitelist: []

  # 5. Strictly whitelist permitted namespace-scoped resources
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: 'apps'
      kind: StatefulSet
    - group: ''
      kind: Service
    - group: ''
      kind: ConfigMap
    - group: 'autoscaling'
      kind: HorizontalPodAutoscaler
    - group: 'policy'
      kind: PodDisruptionBudget
    - group: 'networking.k8s.io'
      kind: NetworkPolicy
    - group: 'external-secrets.io'
      kind: ExternalSecret
    - group: 'platform.internal.bank.com'
      kind: BankPostgreSQLInstance

  # 6. Enforce Change Windows (Sync only allowed Tue/Thu 01:00 - 04:00 IST)
  syncWindows:
    - kind: allow
      schedule: '0 1 * * 2,4'
      duration: 3h
      applications:
        - '*'
      manualSync: false
      timeZone: "Asia/Kolkata"
    - kind: deny
      schedule: '0 0 25-31 * *' # Month-end / Quarter-end financial settlement freeze
      duration: 168h
      applications:
        - '*'
      manualSync: false
      timeZone: "Asia/Kolkata"
```

---

## 8. Cryptographic Supply Chain, Secret Zero & Runtime Admission Control

### A. Zero-Secret GitOps (HashiCorp Vault + External Secrets Operator)
* **Strict Prohibition:** Raw Kubernetes `Secret` objects, base64 passwords, API keys, or TLS private keys must **never** be committed to Git (blocked via pre-commit hooks and GitHub Secret Scanning Push Protection).
* **Architecture:** Applications commit `ExternalSecret` manifests containing only non-sensitive Vault path references. The **External Secrets Operator (ESO)** authenticates to **HashiCorp Vault Enterprise** (backed by HSM Auto-Unseal) using short-lived Kubernetes ServiceAccount tokens and injects credentials at runtime with automated rotation.

### B. Kubernetes Admission Control (`Kyverno` ClusterPolicy)
Deploy Kyverno in `Enforce` mode across all clusters (via Layer 2A `platform-gitops-baseline`) to verify container signatures and pod security posture before etcd persists any pod:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: bank-zero-trust-pod-guardrails
spec:
  validationFailureAction: Enforce
  background: true
  webhookTimeoutSeconds: 15
  rules:
    # Rule 1: Restrict registries & require immutable SHA256 digest
    - name: require-internal-registry-and-digest
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Images must originate from registry.internal.bank.com and use an immutable @sha256 digest."
        pattern:
          spec:
            containers:
              - image: "registry.internal.bank.com/*@sha256:*"

    # Rule 2: Verify Cosign cryptographic image signature & attestation
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "registry.internal.bank.com/*"
          attestors:
            - count: 1
              entries:
                - keys:
                    kms: "awskms:///arn:aws:kms:ap-south-1:111122223333:alias/bank-ci-signing-key"

    # Rule 3: Enforce non-root execution and read-only root filesystem
    - name: enforce-restricted-security-context
      match:
        any:
          - resources:
              kinds: ["Pod"]
      validate:
        message: "Pods must run as non-root with a read-only root filesystem and drop ALL capabilities."
        pattern:
          spec:
            securityContext:
              runAsNonRoot: true
              seccompProfile:
                type: RuntimeDefault
            containers:
              - securityContext:
                  allowPrivilegeEscalation: false
                  readOnlyRootFilesystem: true
                  capabilities:
                    drop: ["ALL"]
```

---

## 9. Continuous Drift Detection, ITSM Integration & Break-Glass Protocol

### A. Automated ITSM / CAB Integration (ServiceNow / Jira)
1. Opening a PR against `/prod/**` in `iac-live-foundation` or `/environments/prod-*/` in `app-config-*` triggers a GitHub Action that automatically creates a Change Request (CR) in **ServiceNow**.
2. The Action attaches the exact Git diff, `terraform plan` output, Checkov/Trivy vulnerability scan report, Cosign verification log, and exact rollback commit SHA to the ServiceNow ticket.
3. A **GitHub Custom Deployment Protection Rule** blocks the merge button until ServiceNow fires an encrypted webhook confirming CAB approval (`state == "Scheduled/Implement"`).
4. Upon merge and ArgoCD/Terraform completion, the pipeline posts the final reconciliation status and audit hash back to ServiceNow and closes the ticket.

### B. Dual-Layer Drift Detection & Self-Healing
* **Layer 2 (ArgoCD):** Configured with `automated: { prune: true, selfHeal: true }`. Any unauthorized manual change on the cluster is overwritten within seconds, and ArgoCD Notifications sends a high-severity alert to the bank's **SIEM (Splunk / Microsoft Sentinel)**.
* **Layer 1 (Terraform):** A scheduled GitHub Action executes `terragrunt run-all plan -detailed-exitcode -lock=false` every **4 hours**. If exit code `2` (drift detected) is returned:
  * A P1 Security & Compliance Incident is automatically generated in ServiceNow/SIEM.
  * The drift diff is correlated with CloudTrail / Azure Activity Logs to identify the rogue principal.

### C. Emergency "Break-Glass" Procedure (P0 Production Incident)
1. **Hotfix via Accelerated GitOps (Preferred):** In a P0 incident, the engineer opens an Emergency PR. Checking OUT of standard CAB queues is achieved via a dedicated `@bank-org/incident-commanders` GitHub Team that can satisfy the emergency branch protection gate alongside an automated P0 PagerDuty/ServiceNow incident ID validator.
2. **Direct Cluster/Cloud Break-Glass (Extreme Edge Case):**
   * If GitHub or ArgoCD itself is degraded, an authorized SRE requests a time-bound (60-minute) **Break-Glass Role** via **CyberArk / Entra ID PIM**, requiring dual-peer approval and an active P0 Incident Ticket ID.
   * For ArgoCD workloads, the SRE executes an audited break-glass script that applies a temporary `deny` SyncWindow with `manualSync: true` to pause self-healing while diagnostics occur.
   * Every command executed during the session is recorded and streamed to immutable WORM storage.
   * **Mandatory Post-Incident Reconciliation:** All break-glass changes must be back-ported into `iac-live-foundation` or `app-config-*` within 24 hours as a mandatory gate of the Post-Incident Review (PIR).

---

## 10. Multi-Region Disaster Recovery (DR) & Active-Active Architecture

In alignment with **RBI Business Continuity & DR mandates** (requiring regular switches between Primary Data Centre [e.g., Mumbai] and Disaster Recovery Data Centre [e.g., Hyderabad] across different seismic zones):

1. **Infrastructure Symmetry via Terragrunt:** Both `ap-south-1-mumbai-primary` and `ap-south-2-hyderabad-dr` consume the exact same versioned modules from `iac-modules-registry`. DR readiness is verified continuously by ensuring zero diff between primary and DR module versions.
2. **ArgoCD Control Plane HA & DR:**
   * Run the Primary Production ArgoCD Control Plane in a Dedicated Management Cluster in Mumbai, with a Warm-Standby ArgoCD Control Plane in Hyderabad.
   * Since **all ArgoCD state (Applications, AppProjects, RBAC, Repo Configs) is stored declaratively in `platform-gitops-baseline`**, the Hyderabad ArgoCD instance is continuously synced from the same Git repository ("ArgoCD managing ArgoCD").
3. **Workload Traffic Shifting:** Application manifests are deployed simultaneously to both Mumbai and Hyderabad clusters via ArgoCD `ApplicationSets`. Traffic failover during DR drills is executed via a GitOps commit to the DNS/GSLB route weighting config or Service Mesh Gateway without requiring emergency container redeployments.

---

## 11. Regulatory Compliance Traceability Matrix (CISO & Audit Sign-Off)

| Regulatory Framework | Control Requirement | Prescriptive GitOps / IaC Implementation | Audit Evidence Generated |
| :--- | :--- | :--- | :--- |
| **RBI Master Direction on IT Governance & Cybersecurity** | **Maker-Checker & Change Management:** All changes to production systems must be authorized, tested, and segregated from development duties. | • 5-Tier Repo Separation (`app-source` vs `app-config` vs `iac-live`).<br>• GitHub `CODEOWNERS` blocking self-approval.<br>• ServiceNow CAB Webhook gating PR merges. | Immutable Git commit graph, PR approval signatures, and linked ServiceNow Change Ticket IDs. |
| **RBI Cyber Resilience & DR Drills** | **BCP/DR Readiness:** Periodic validation of recovery time objectives (RTO/RPO) across geographically separated sites. | • Symmetrical Terragrunt topologies (`prod-mumbai` & `prod-hyderabad`).<br>• ArgoCD `ApplicationSets` maintaining warm/active multi-cluster parity. | Git history of DR traffic-shift commits and zero-drift Terragrunt plan logs across regions. |
| **PCI-DSS v4.0 (Req 6.4 & 6.5)** | **Secure Software Development & Change Control:** Separate development/test environments from production; trace all changes to known authors. | • Physically isolated Non-Prod and Prod ArgoCD instances.<br>• Mandatory hardware-backed GPG/SSH commit signing enforced on `main` and verified by ArgoCD. | Cryptographically verifiable GPG signatures on every Git commit and Cosign attestations on every container image. |
| **PCI-DSS v4.0 (Req 7 & 8)** | **Restrict Access by Business Need-to-Know & Zero Standing Privileges:** Limit system component access to least privilege. | • Zero human write access to K8s API (`kubectl`) and Cloud Consoles in Prod.<br>• OIDC ephemeral tokens for Terraform runners.<br>• Locked-down ArgoCD `AppProject` resource whitelists. | Kubernetes Audit Logs and CloudTrail showing `argocd-application-controller` and `github-actions-oidc` as the sole mutating principals. |
| **SOC2 Type II (CC6.1 & CC8.1)** | **Logical Access & Unauthorized Change Prevention:** Detect and prevent unauthorized infrastructure or configuration modifications. | • ArgoCD `selfHeal: true` and `prune: true` (3-min loop).<br>• Scheduled 4-hour Terraform drift detection pipeline.<br>• Kyverno runtime admission enforcement. | SIEM logs of blocked admission requests, automated drift remediation events, and OPA policy evaluation reports on every PR. |

---

## 12. 90-Day Executive Implementation Roadmap

### Phase 1: Foundation, Identity & State Security (Weeks 1–3)
* Provision the 5-Tier Repository structure on GitHub Enterprise with standardized branch protection and `.github/CODEOWNERS`.
* Configure OIDC federation between GitHub Enterprise, Cloud IAM (AWS/Azure/GCP), and Corporate IdP (Entra ID / Okta).
* Bootstrap Tier-0 encrypted Terraform state backends (HSM-backed KMS, VPC endpoint lockdown, DynamoDB/Blob state locking).
* Enforce hardware-backed GPG/SSH commit signing for all engineering and platform teams.

### Phase 2: Guardrails, Secret Zero & Layer-1 IaC Pipeline (Weeks 4–6)
* Implement the PR-driven Terragrunt GitHub Actions pipeline with Checkov, Infracost, and OPA/Rego policy gates.
* Deploy isolated Non-Prod and Prod Hub ArgoCD Control Planes using the "Terraform-to-ArgoCD" Cluster Secret bridge.
* Integrate HashiCorp Vault with Kubernetes via the External Secrets Operator (ESO) and deploy Kyverno admission policies in `Enforce` mode.

### Phase 3: Pilot Onboarding, Crossplane & ITSM Automation (Weeks 7–9)
* Publish initial Crossplane `CompositeResourceDefinitions` (XRDs) for self-service databases, queues, and storage buckets.
* Onboard 2 non-critical banking applications and 1 core payment microservice end-to-end (`dev` $\rightarrow$ `uat` $\rightarrow$ `prod`).
* Activate the automated ServiceNow CAB Change Request webhook integration on Production GitHub Environments.
* Execute a live **Security & Resilience GameDay**: simulate unauthorized `kubectl`/Console drift, test CyberArk PIM Break-Glass access, and execute a Mumbai-to-Hyderabad DR failover drill.

### Phase 4: Audit Sign-Off & Enterprise Factory Scale (Weeks 10–12)
* Conduct an end-to-end evidence walkthrough with Internal Audit, Risk, and the CISO office using the Regulatory Traceability Matrix.
* Revoke remaining legacy standing human write permissions on Cloud Portals and Kubernetes clusters.
* Publish self-service repository onboarding templates (via Backstage or Terraform GitHub Provider) to scale onboarding across all banking tribes.
