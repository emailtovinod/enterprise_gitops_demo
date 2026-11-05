# Enterprise Prescriptive Blueprint: On-Premises GitOps Operating Model for VMware Cloud Foundation (VCF 9 / 9.1) in Regulated Banking

**Document Classification:** Tier-1 On-Premises Private Cloud Architecture & Security Governance Blueprint  
**Target Platform:** Broadcom VMware Cloud Foundation (VCF 9.0 / 9.1), vSphere Supervisor, vSphere Kubernetes Service (VKS)  
**Core GitOps & Security Stack:** GitHub Enterprise Server (Air-Gapped), ArgoCD (Supervisor Service & Dedicated Hub), OpenBao + External Secrets Operator (ESO), Harbor Enterprise Registry, Kyverno, NSX / VMware vDefend, VMware Avi Load Balancer (AKO)  
**Optional Provisioning Extensions:** Terraform / OpenTofu (`vsphere`, `nsxt`, `vcf` providers) & Crossplane  
**Regulatory Alignment:** RBI Master Direction on IT Governance & Cybersecurity, PCI-DSS v4.0, SOC2 Type II, ISO/IEC 27001  

---

## 1. Executive Summary & Architectural Paradigm Shift in VCF 9 / 9.1

In an on-premises Tier-1 banking data centre running **VMware Cloud Foundation (VCF 9 / 9.1)**, you do not need third-party wrappers to turn bare-metal hypervisors into a declarative cloud API. **vSphere Supervisor** natively transforms ESXi clusters, vSAN Enterprise Storage (CNS), and NSX / Avi Networking into a **Kubernetes-native declarative control plane**.

This creates a fundamental architectural advantage for regulated banks:
1. **Declarative Kubernetes-as-a-Service (VKS):** In VCF 9 / 9.1, **vSphere Kubernetes Service (VKS)** clusters (running FIPS-validated **VMware Kubernetes Releases [VKr]**, such as K8s `v1.31+` to `v1.34+` on Ubuntu 24.04 / Photon OS OVAs) are provisioned declaratively by submitting upstream **Cluster API (`cluster.x-k8s.io/v1beta1`)** manifests directly to a vSphere Supervisor Namespace.
2. **Native ArgoCD Supervisor Service:** VCF 9+ provides an officially supported **ArgoCD Supervisor Service (`argocd-service.vsphere.vmware.com/v1alpha1`)** and standard VKS Carvel packages, enabling **100% pure GitOps** for both **Cluster Lifecycle Management** (provisioning VKS clusters, VM Service VMs, and NSX/Avi networking via Supervisor) and **Workload Lifecycle Management** (deploying banking microservices inside VKS clusters).
3. **Open-Source Sovereign Secret Management (OpenBao):** To eliminate proprietary licensing lock-in and cloud dependencies while maintaining FIPS/HSM compliance, secrets are centralized in an on-premises **OpenBao** HA cluster (the Linux Foundation open-source fork of Vault) backed by on-premises **PKCS#11 Hardware Security Modules (Thales Luna / Entrust nShield)** and reconciled into VKS via the **External Secrets Operator (ESO)**.

---

## 2. The Three-Layer VCF 9 GitOps Architecture (With & Without Terraform/Crossplane)

In VCF 9 / 9.1, the traditional boundary between Infrastructure-as-Code (IaC) and Kubernetes GitOps is streamlined because the **vSphere Supervisor** speaks native Kubernetes YAML. Below is the prescriptive Three-Layer Operating Model, clearly demarcating **Mandatory Core VCF GitOps** from **Optional Terraform / Crossplane** patterns:

| Layer | Scope & VCF 9 / 9.1 Assets | Primary Execution Pattern (Recommended Native VCF 9) | Optional Alternative (Terraform / Crossplane) | Reconciliation Engine |
| :--- | :--- | :--- | :--- | :--- |
| **Layer 0: Physical SDDC & Supervisor Foundation ("Day-0")** | SDDC Manager, Workload Domains, ESXi Hosts, vSAN ESA Datastores (RAID-6 + Encryption), NSX Edge Clusters, Avi Controllers, Content Libraries (VKr OVAs), and **vSphere Supervisor Activation** + **vSphere Namespaces** (CPU/Memory/Storage Quotas, VM Classes). | **VCF Automation (VCFA) Namespace Classes** + SDDC Manager APIs (or Automated Day-0 Ansible/PowerCLI). | **Optional Terraform (`vmware/vcf`, `vmware/vsphere`, `vmware/nsxt`)**: Automates Workload Domain creation, NSX Tier-0/Tier-1 gateways, and initial vSphere Supervisor Namespace boundaries via PR-driven GitHub Actions. | **VCF Automation / PR-Driven Terraform Plan & Apply** |
| **Layer 1: VKS Cluster Lifecycle & Supervisor Infrastructure ("Day-1")** | Provisioning multi-zone **VKS Clusters** (`Cluster` CAPI `v1beta1` using `tanzukubernetescluster` ClusterClass), VM Service Virtual Machines (`vmoperator.vmware.com/v1alpha5`), CNS StorageClass bindings, and Avi LoadBalancer IP Pools. | **Pure ArgoCD against vSphere Supervisor:** ArgoCD syncs Cluster API `Cluster` YAML manifests directly into the vSphere Supervisor Namespace to create, scale, and upgrade VKS clusters. | **Optional Terraform or Crossplane:**<br>• *Terraform:* Uses `vcfa` / `kubernetes` provider to apply CAPI YAML.<br>• *Crossplane:* Runs on a Hub cluster using `provider-kubernetes` or `provider-vsphere` to provision VKS clusters or on-prem DB VMs via XRDs. | **ArgoCD (Hub Control Plane targeting vSphere Supervisor API)** |
| **Layer 2: VKS Cluster Baseline & Banking Workloads ("Day-2")** | **Layer 2A (Platform Baseline):** Antrea CNI NetworkPolicies, vDefend micro-segmentation, AKO (`HostRule`/`HttpRule` WAF), Kyverno, OpenBao ESO, VKS Standard Carvel Packages (Cert-Manager, Contour/Istio, FluentBit).<br>**Layer 2B (Workloads):** Core Banking microservices, HPA, `ExternalSecret` objects. | **Pure ArgoCD against Workload VKS Clusters:** ArgoCD auto-discovers newly provisioned VKS clusters from Supervisor secrets and continuously reconciles platform add-ons and banking apps. | **Optional Crossplane (In-Cluster):** Developers commit a `BankPostgresVM` or `BankMinIOBucket` claim; Crossplane provisions an on-prem VM Service PostgreSQL instance or Ceph/vSAN S3 bucket. | **ArgoCD (Continuous 3-Min Pull Loop with `selfHeal: true`)** |

---

## 3. Prescriptive Decision Matrix: Do You Need Terraform or Crossplane on VCF 9?

As a global consultant advising a bank's Architecture Review Board, use this decision matrix to determine whether to include Terraform or Crossplane in your VCF 9 stack:

### Scenario A: Pure Native VCF 9 GitOps (No Terraform / No Crossplane for Day-1/Day-2) — *Recommended Default*
* **How it works:** VI Admins create the **vSphere Supervisor Namespace** (assigning vSAN Storage Policies, `guaranteed-*` VM Classes, and resource quotas) via **VCF Automation (VCFA)** or vCenter. From that point forward, **ArgoCD connects directly to the vSphere Supervisor API** and manages all **VKS Clusters (`cluster.x-k8s.io/v1beta1`)** and **VM Service VMs (`vmoperator.vmware.com/v1alpha5`)** purely via Git commits.
* **Why Banks Prefer This:** Zero Terraform state files (`.tfstate`) to encrypt, lock, or audit for Kubernetes cluster lifecycle. When a VKr Kubernetes patch (`v1.33.x+vmware.1` $\rightarrow$ `v1.34.2+vmware.1`) is synchronized into the air-gapped vCenter Content Library, upgrading a 50-node PCI VKS cluster is a **1-line Git PR** updating `spec.topology.version` in the CAPI manifest synced by ArgoCD.

### Scenario B: When to Include Terraform / OpenTofu (Optional Layer-0 / Day-0 Module)
Include Terraform **only** at Layer 0 if the bank does not use VCF Automation (VCFA) templates and wants Git-driven provisioning of:
1. **vSphere Supervisor Namespaces** (attaching vSAN storage policies, VM classes, and AD/LDAP/OIDC permissions to namespaces).
2. **NSX Tier-1 Gateways & vDefend Distributed Firewall (DFW) Security Policies** that sit outside Kubernetes.
3. **Legacy VMware Virtual Machines** (e.g., clustered Oracle RAC or IBM MQ VMs on vSphere outside the Supervisor VM Service).
* *Guardrail:* If Terraform is used for Layer 0, store the `.tfstate` in an on-premises **S3-compatible Object Store (vSAN Data Persistence platform / MinIO / Ceph RGW)** with WORM Object Locking and encrypt state using **OpenBao's `transit` encryption engine**.

### Scenario C: When to Include Crossplane (Optional Layer-2 Self-Service Module)
Include Crossplane **only** if application teams require self-service provisioning of **external on-premises stateful dependencies** alongside their VKS pods:
* For example, a developer commits a `BankDatabaseClaim` YAML in their `app-config` repo. Crossplane (using `provider-kubernetes` targeting the Supervisor `vmoperator.vmware.com/v1alpha5` API or `provider-ansible`/`provider-sql`) spins up a hardened Ubuntu 24.04 PostgreSQL VM on vSphere or provisions a logical schema on an existing enterprise DB cluster, injecting credentials straight into **OpenBao**.

---

## 4. Enterprise Repository Topology for On-Premises VCF 9

Enforce strict Segregation of Duties across VI Admins (vSphere/NSX), Platform SREs (VKS/ArgoCD/OpenBao), and Application Tribes using a **Four-Tier Repository Structure** hosted on an air-gapped **GitHub Enterprise Server (GHES)** instance:

```text
1. vcf-supervisor-fleet-gitops/          (Owned by Platform SRE & VI Admin Approvers)
   ├── namespaces/                       # Supervisor Namespace Quotas, RBAC, LimitRanges
   └── vks-clusters/                     # Declarative CAPI v1beta1 VKS Cluster definitions
       ├── non-prod/
       │   ├── dev-payments-vks01.yaml
       │   └── uat-payments-vks01.yaml
       └── prod-pci/
           ├── dc1-mum-payments-vks01.yaml
           └── dc2-hyd-payments-vks01.yaml

2. vks-platform-baseline-gitops/         (Owned by Platform SRE & SecOps)
   ├── argocd-projects/                  # Multi-tenant AppProjects & SyncWindows
   ├── cluster-addons/                   # Antrea policies, Kyverno, OpenBao ESO, AKO CRDs, Cert-Manager
   └── kyverno-policies/                 # Enforce internal Harbor registry, Cosign, non-root

3. app-source-<tribe>-<service>/         (Owned by Application Developers)
   ├── src/
   └── Dockerfile                        # Built & pushed to on-prem Harbor with Cosign signature

4. app-config-<tribe>/                   (Owned by App Tech Leads & CAB Release Managers)
   ├── base/                             # Deployment, Service, HPA
   └── environments/
       ├── dev/                          # Kustomize overlay + ExternalSecret + Avi HostRule
       ├── uat/
       ├── prod-dc1-mumbai/
       └── prod-dc2-hyderabad/
```
*(Note: If Optional Layer-0 Terraform is adopted, add a 5th repository `vcf-foundation-iac` strictly restricted to VI Admins and Network Security Engineers).*

---

## 5. Declarative VKS Cluster Provisioning via GitOps (VCF 9 / 9.1 CAPI Spec)

In VCF 9 / 9.1, VKS clusters are defined using upstream **Cluster API (`cluster.x-k8s.io/v1beta1`)** and the Supervisor's built-in `tanzukubernetescluster` (or `builtin-generic-v3.x`) `ClusterClass`. 

Below is the **prescriptive, production-hardened VKS Cluster manifest** (`prod-pci/dc1-mum-payments-vks01.yaml`) managed by ArgoCD against the vSphere Supervisor:

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: mum-prod-payments-vks01
  namespace: ns-payments-prod-pci # vSphere Supervisor Namespace
  labels:
    bank.com/environment: "prod-pci"
    bank.com/datacenter: "dc1-mumbai"
    bank.com/business-unit: "payments"
    bank.com/cni: "antrea"
spec:
  clusterNetwork:
    services:
      cidrBlocks: ["198.51.100.0/12"] # Disjoint CIDR per environment to prevent routing overlap
    pods:
      cidrBlocks: ["192.0.2.0/16"]
    serviceDomain: "cluster.local"
  topology:
    class: tanzukubernetescluster
    version: v1.33.1+vmware.1-fips.1 # Must exist in local air-gapped vCenter Content Library
    controlPlane:
      replicas: 3 # Mandatory 3-node HA etcd quorum across vSphere Fault Domains
      metadata:
        annotations:
          run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
    workers:
      machineDeployments:
        - class: node-pool
          name: pci-worker-pool-az1
          replicas: 4
          failureDomain: fd-rack-01
          metadata:
            annotations:
              run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
        - class: node-pool
          name: pci-worker-pool-az2
          replicas: 4
          failureDomain: fd-rack-02
          metadata:
            annotations:
              run.tanzu.vmware.com/resolve-os-image: os-name=ubuntu,os-version=24.04
    variables:
      # 1. Enforce Guaranteed VM Classes in Production (100% CPU/Memory vSphere Reservation)
      - name: vmClass
        value: guaranteed-large # Never use best-effort-* classes in Production banking
      # 2. Bind to vSAN ESA Encrypted RAID-6 Storage Policy via CNS
      - name: storageClass
        value: vsan-esa-raid6-encrypted-policy
      # 3. Customise Node Volumes & FIPS Kernel Hardening
      - name: volumes
        value:
          - name: containerd
            mountPath: /var/lib/containerd
            capacity:
              storage: 100Gi
          - name: kubelet
            mountPath: /var/lib/kubelet
            capacity:
              storage: 50Gi
      # 4. Inject Bank Internal Root CA & Air-Gapped Harbor Registry Trust into Nodes
      - name: trust
        value:
          additionalTrustedCAs:
            - name: bank-internal-root-ca
              data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUU... # Base64 Bank PKI Root CA
```

### Automated VKS Cluster Onboarding into ArgoCD
When the vSphere Supervisor provisions `mum-prod-payments-vks01`, it automatically generates a Kubeconfig secret named `mum-prod-payments-vks01-kubeconfig` inside the Supervisor Namespace (`ns-payments-prod-pci`).
* An automated **Cluster-Registration Controller** (or ArgoCD Supervisor Operator) transforms this Kubeconfig (bound via the **vSphere Pinniped / OIDC Supervisor Broker**) into an ArgoCD Cluster Secret with labels `bank.com/environment: prod-pci`.
* ArgoCD's **Cluster Generator (`ApplicationSet`)** immediately detects the new VKS cluster and deploys Layer 2A (Kyverno, OpenBao ESO, AKO, Antrea NetworkPolicies) before any banking application is scheduled.

---

## 6. Secret Zero Architecture with OpenBao & External Secrets Operator (ESO)

In an on-premises banking environment, storing secrets in Git—even encrypted via static SealedSecrets keys—violates PCI-DSS v4.0 key lifecycle and central revocation mandates. We implement **OpenBao** as the sovereign, open-source central Secret Store.

### A. OpenBao On-Premises High-Availability & HSM Architecture
1. **Deployment Topology:** Run a 5-node **OpenBao** cluster (using integrated **Raft storage** with automated snapshots to encrypted vSAN S3/NFS) across dedicated VM Service VMs or a dedicated Security Management VKS cluster, fronted by **VMware Avi Load Balancer** with strict mTLS passthrough.
2. **HSM Auto-Unseal (PKCS#11):** Configure OpenBao's `seal "pkcs11"` stanza to interface directly with the bank's on-premises **Hardware Security Module (Thales Luna / Entrust nShield)** over the internal HSM VLAN. The OpenBao root encryption key never leaves the FIPS 140-3 Level 3 HSM boundary.
3. **Dynamic Secrets & Rotation:** Configure OpenBao to generate **short-lived, dynamic database credentials** (for Oracle, PostgreSQL, MS SQL) and automated X.509 mTLS certificates via OpenBao's PKI Engine chained to the Bank's Enterprise Offline Root CA.

### B. Zero-Secret GitOps Flow (VKS $\leftrightarrow$ ESO $\leftrightarrow$ OpenBao)
1. **Kubernetes Auth Method:** Enable the `kubernetes` auth mount in OpenBao for each VKS cluster. The **External Secrets Operator (ESO)** running inside the VKS cluster authenticates to OpenBao using a short-lived, bound Kubernetes `ServiceAccount` JWT (`TokenRequest` API)—**zero static tokens sit on the VKS cluster**.
2. **`ClusterSecretStore` Configuration (Managed by Platform SRE in `vks-platform-baseline-gitops`):**
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ClusterSecretStore
   metadata:
     name: openbao-bank-prod
   spec:
     provider:
       vault: # OpenBao is 100% API-compatible with the ESO Vault provider
         server: "https://openbao.sec.internal.bank.com:8200"
         path: "kv-prod-pci"
         version: "v2"
         caProvider:
           type: "ConfigMap"
           name: "bank-root-ca"
           key: "ca.crt"
           namespace: "external-secrets"
         auth:
           kubernetes:
             mountPath: "vks-mum-prod-payments-01"
             role: "payments-prod-eso-role"
             serviceAccountRef:
               name: "eso-openbao-auth-sa"
               namespace: "payments-prod"
   ```
3. **Application `ExternalSecret` Manifest (Committed to Git in `app-config-payments`):**
   ```yaml
   apiVersion: external-secrets.io/v1beta1
   kind: ExternalSecret
   metadata:
     name: upi-switch-hsm-and-db-creds
     namespace: payments-prod
   spec:
     refreshInterval: "15m" # Automatically pulls rotated credentials from OpenBao every 15 mins
     secretStoreRef:
       name: openbao-bank-prod
       kind: ClusterSecretStore
     target:
       name: upi-runtime-secret
       creationPolicy: Owner
     data:
       - secretKey: DB_PASSWORD
         remoteRef:
           key: "payments/upi-core/database"
           property: "password"
       - secretKey: NPCI_SIGNING_KEY_PIN
         remoteRef:
           key: "payments/upi-core/hsm-slot"
           property: "partition_pin"
   ```

---

## 7. VCF-Native Networking, WAF & Micro-Segmentation via GitOps

Unlike public cloud Kubernetes where Ingress and Firewalls require disparate cloud controllers, VCF 9 integrates **Antrea CNI**, **NSX / VMware vDefend**, and **VMware Avi Load Balancer (AKO)** directly into the GitOps loop.

### 1. Ingress, TLS & Web Application Firewall (WAF) as Code via Avi (`AKO`)
Instead of manual ticket requests to the load balancer team, application exposure and **PCI-DSS WAF rules** are declared in Git using Avi Kubernetes Operator (`ako.vmware.com`) CRDs synced by ArgoCD:
```yaml
apiVersion: ako.vmware.com/v1beta1
kind: HostRule
metadata:
  name: upi-payments-waf-hardening
  namespace: payments-prod
spec:
  virtualhost:
    fqdn: upi-api.prod.internal.bank.com
    fqdnType: Exact
    enableVirtualHost: true
    # 1. Attach Enterprise WAF Policy (OWASP Top 10 + PCI-DSS Signature Set)
    wafPolicy: Bank-PCI-Strict-WAF-Policy
    # 2. Enforce TLS 1.3 / Strong Cipher Suites via Avi SSL Profile
    sslProfile: Bank-TLS13-Strict-Profile
    # 3. Attach DDoS & Rate-Limiting HTTP Policy
    httpPolicy:
      policySets:
        - Bank-UPI-RateLimit-PolicySet
```

### 2. Zero-Trust East-West Micro-Segmentation (`Antrea` & `vDefend`)
Every VKS namespace enforces a **Default-Deny All East-West & Egress** posture via Antrea/Kubernetes `NetworkPolicy` (deployed automatically by Layer 2A `vks-platform-baseline-gitops`). Furthermore, **VMware vDefend** Distributed Firewall (DFW) policies inspect traffic between VKS Pods and traditional vSphere VMs (such as legacy Core Banking Mainframe/DB VMs), ensuring compromised pods cannot laterally scan the vSphere VLAN.

---

## 8. Enterprise RBAC & Segregation of Duties (SoD) Across VCF, GitHub & ArgoCD

In an on-premises VCF 9 bank environment, four distinct operational personas exist. Integrate **Active Directory / Entra ID / Okta / LDAP** with **VCF Single Sign-On (vCenter Identity Broker)**, **GitHub Enterprise Server**, **ArgoCD**, and **OpenBao**.

| Banking Persona | vCenter / VCF Supervisor Access | GitHub Enterprise Access | ArgoCD RBAC Permissions | Direct VKS `kubectl` Access |
| :--- | :--- | :--- | :--- | :--- |
| **VI / Cloud Admin (SDDC & vSphere)** | Manages ESXi, vSAN, NSX, Content Libraries, and Supervisor Namespace Quotas. **No access inside VKS clusters.** | `CODEOWNERS` on `vcf-foundation-iac` (if used) & Supervisor Namespace quotas. | `role:readonly` on `vks-cluster-lifecycle` project. | **None** |
| **Platform SRE (VKS & GitOps Admin)** | `Can Edit` on Supervisor Namespace via `vcf` CLI (Break-Glass only; normal ops via ArgoCD). | `Maintain` on `vcf-supervisor-fleet-gitops` & `vks-platform-baseline-gitops`. | `role:platform-admin` (Manages `AppProjects`, VKS cluster provisioning, add-ons). | **JIT Break-Glass Only** via CyberArk/PAM with session recording |
| **Application Developer (Maker)** | **None** | `Write` on `app-source-*` & `app-config-*` (Creates PRs; **blocked from merging to `main`**). | `role:dev-deployer` (`get`, `sync` on `dev`/`uat`; `readonly` on `prod`). | Read-only pod logs in `dev` VKS via Pinniped OIDC; **Zero access in `uat`/`prod`** |
| **CAB Release Manager (Checker)** | **None** | `Maintain` (`CODEOWNERS` for `/environments/prod-*/` in `app-config-*`). | `role:prod-operator` (`get`, `sync` on `prod` during approved `SyncWindows`; **no `delete` or `override`**). | **None** |
| **SecOps / Cryptography Admin** | Read-only vDefend & NSX flow logs. | `CODEOWNERS` on `/kyverno-policies/` & OpenBao policy repos. | `role:auditor` (`get` across all projects). | Manages **OpenBao** policies & HSM unseal keys (M-of-N quorum). |
| **Internal / RBI Auditor** | Read-only vCenter Compliance & Events. | `Read` across all repositories, PR approvals, and CI logs. | `role:auditor` (`get` across all apps, sync history, and logs). | Read-only access to Aria Operations for Logs / SIEM. |

### ArgoCD Architecture on VCF 9: Supervisor Service vs. Dedicated Management VKS
In VCF 9 / 9.1, you can deploy ArgoCD using the native **ArgoCD Supervisor Service (`argocd-service.vsphere.vmware.com/v1alpha1`)**:
* **Prescriptive Banking Topology:** Deploy **two physically isolated ArgoCD instances**:
  1. **Non-Prod ArgoCD Instance:** Deployed in the Non-Prod VCF Workload Domain, managing Dev and UAT VKS clusters.
  2. **Prod / PCI-DSS ArgoCD Instance:** Deployed in the Production Management/Workload Domain, managing only Production VKS clusters.
* **Locking Down the ArgoCD Instance (`AppProject`):**
  * Configure `signatureKeys` on all Production `AppProjects` so ArgoCD **only syncs commits signed with hardware-backed GPG/SSH keys**.
  * Set `clusterResourceWhitelist: []` on all application tribe `AppProjects` so application teams can only deploy `Deployment`, `Service`, `ConfigMap`, `HorizontalPodAutoscaler`, `ExternalSecret`, and `HostRule`—preventing them from modifying `ClusterRole`, `Namespace`, or `ClusterSecretStore` objects.

---

## 9. Air-Gapped Supply Chain Security (Harbor + Cosign + Kyverno + VKr)

Tier-1 banking data centres operate in **strictly air-gapped networks** with zero direct internet egress from ESXi hosts, Supervisor Control Planes, or VKS Worker Nodes.

1. **Air-Gapped VKr Node Image Distribution:**
   * VMware Kubernetes Releases (VKr OVAs) and VKS Standard Carvel Package bundles (`vks-standard-packages`) are downloaded on a DMZ Bastion host, verified via SHA256/GPG signature, and imported into an internal **Subscribed vCenter Content Library** backed by vSAN.
   * When ArgoCD updates `spec.topology.version` on a VKS `Cluster` manifest, the vSphere Supervisor clones the hardened OVA locally over the internal storage network at 25/100 GbE speeds.
2. **Harbor Enterprise Container Registry & Cosign Signing:**
   * All base OS images, VKS system packages, and banking microservice containers live in an on-premises, HA **VMware Harbor Registry** (with Trivy vulnerability scanning enabled).
   * The GitHub Actions self-hosted runner signs every built container image digest (`@sha256:...`) using **Cosign** with a private key stored in **OpenBao's `transit` signing engine** (or PKCS#11 HSM).
3. **Kyverno Runtime Enforcement on VKS:**
   * Kyverno runs in `Enforce` mode on every VKS cluster and blocks any Pod admission request unless:
     * The image originates strictly from `harbor.internal.bank.com/*` and uses an immutable `@sha256:` digest.
     * The image carries a valid **Cosign cryptographic signature** verified against the bank's public key in OpenBao/KMS.
     * The Pod enforces `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, and `capabilities.drop: ["ALL"]`.

---

## 10. Multi-Site Disaster Recovery (Mumbai Primary DC $\leftrightarrow$ Hyderabad DR DC)

To satisfy **RBI Cyber Resilience and Business Continuity mandates** across geographically separated seismic zones:

1. **Active-Active / Warm-Standby VKS Fleet Symmetry:**
   * Maintain two separate VCF 9 instances: **DC1 (Mumbai Primary)** and **DC2 (Hyderabad DR)**.
   * In `vcf-supervisor-fleet-gitops`, define identical VKS cluster manifests (`dc1-mum-payments-vks01.yaml` and `dc2-hyd-payments-vks01.yaml`) pinned to the exact same `v1.33.1+vmware.1-fips.1` VKr version and `guaranteed-large` VM class.
2. **OpenBao Multi-Site Replication / Sync:**
   * Run a local OpenBao cluster in both Mumbai and Hyderabad (each unsealed by local DC HSMs), synchronized via automated GitOps policy/secret replication pipelines or Raft DR replication so `ExternalSecret` objects resolve locally in both data centres without cross-DC latency.
3. **Global Server Load Balancing (VMware Avi GSLB) + ArgoCD `ApplicationSets`:**
   * ArgoCD deploys payment microservices simultaneously to both Mumbai and Hyderabad VKS clusters using an `ApplicationSet` Matrix Generator.
   * **VMware Avi GSLB** monitors the health of the Avi VirtualServices in both DCs. During an RBI-mandated unannounced DR drill, shifting traffic from Mumbai to Hyderabad requires either an automated GSLB health-check failover or a single audited GitOps commit updating the Avi `GSLBService` weight manifest.
4. **Persistent Volume DR (Velero + vSphere CNS / vSAN Replication):**
   * Stateless microservices recover instantaneously via GitOps. For stateful workloads using vSphere Cloud Native Storage (CNS), pair **vSAN Replication / Site Recovery** with **Velero** (using the vSphere CSI plugin) to snapshot and replicate PVCs to the DR site's vSAN datastore.

---

## 11. Regulatory Compliance Mapping for VCF 9 On-Premises GitOps

| Regulatory Mandate | Specific Requirement | VCF 9 + VKS + OpenBao + ArgoCD Implementation | Verifiable Audit Evidence |
| :--- | :--- | :--- | :--- |
| **RBI Master Direction on IT Governance & Cybersecurity** | **Maker-Checker, Change Control & Patch Management:** Strict separation between code authors and production deployers; timely OS/hypervisor patching. | • 4-Tier GHES Repo separation + `CODEOWNERS` + ServiceNow CAB Webhook.<br>• Zero-downtime rolling VKS node upgrades by bumping `topology.version` (VKr OVA) in Git. | Signed Git commit log, ServiceNow CR ticket link, and vSphere Supervisor rolling upgrade task logs. |
| **PCI-DSS v4.0 (Req 3.5, 3.6 & 8.6)** | **Cryptographic Key Management & Elimination of Hardcoded Application Secrets.** | • **OpenBao** unsealed via FIPS 140-3 Level 3 HSM (PKCS#11).<br>• **External Secrets Operator (ESO)** using short-lived K8s SA JWTs (`refreshInterval: 15m`). | OpenBao audit log stream in SIEM (Aria Operations for Logs / Splunk) showing dynamic lease issuance and zero static secrets in Git. |
| **PCI-DSS v4.0 (Req 1.3, 1.4 & 6.4)** | **Network Segmentation of CDE & Web Application Firewall Protection.** | • **Antrea CNI** default-deny `NetworkPolicy` + **VMware vDefend** DFW micro-segmentation.<br>• **Avi Load Balancer (`HostRule`)** enforcing `Bank-PCI-Strict-WAF-Policy` via GitOps. | Declarative `HostRule` and `NetworkPolicy` YAML in Git; vDefend blocked packet logs; Avi WAF security analytics dashboard. |
| **SOC2 Type II (CC6.1, CC7.2 & CC8.1)** | **Prevention of Unauthorized Configuration Drift & Privileged Access Control.** | • Human `kubectl` write access disabled on VKS; VI Admins isolated from VKS runtime.<br>• ArgoCD `selfHeal: true` & Kyverno `Enforce` mode blocking unsigned/root pods. | Kyverno `PolicyReport` CRDs (`PASS/FAIL` scorecard) and ArgoCD drift-remediation alerts in SIEM. |

---

## 12. 90-Day VCF 9 GitOps Rollout Roadmap

### Phase 1: VCF 9 Supervisor, Harbor & OpenBao Foundation (Weeks 1–3)
* Configure vSphere Supervisor on VCF 9 / 9.1 Workload Domains with NSX/Avi Load Balancing and vSAN ESA encrypted storage policies.
* Deploy air-gapped **GitHub Enterprise Server**, **VMware Harbor Registry** (synced with VKr OVAs & VKS standard packages), and **OpenBao HA Cluster** integrated with on-premises PKCS#11 HSMs.
* Establish the 4-Tier Git repository structure, `.github/CODEOWNERS`, and corporate IdP OIDC federation (Pinniped / vCenter Identity Broker).

### Phase 2: Declarative VKS Provisioning & Platform Baseline (Weeks 4–6)
* Deploy isolated Non-Prod and Prod **ArgoCD Instances** (via the VCF 9 ArgoCD Supervisor Service).
* Provision the first Dev, UAT, and Prod-PCI **VKS Clusters** purely via GitOps by syncing Cluster API (`cluster.x-k8s.io/v1beta1`) manifests into vSphere Supervisor Namespaces.
* Configure ArgoCD `ApplicationSets` to automatically bootstrap newly created VKS clusters with **External Secrets Operator (connected to OpenBao)**, **Kyverno**, **Antrea NetworkPolicies**, and **AKO**.

### Phase 3: Pilot Application Migration, Avi WAF & CAB Integration (Weeks 7–9)
* Migrate 2 internal banking services and 1 Tier-1 payment microservice onto VKS using Kustomize overlays, OpenBao `ExternalSecrets`, and Avi `HostRule` WAF manifests.
* Enable the ServiceNow CAB Change Request webhook gate on Production GitHub Environments.
* Conduct a live **VCF 9 Resilience GameDay**: test rolling VKr Kubernetes version upgrades via Git PR, simulate unauthorized `kubectl` drift, test OpenBao secret rotation, and execute an Avi GSLB failover between Primary and DR data centres.

### Phase 4: CISO Audit Sign-Off & Enterprise Factory Scale (Weeks 10–12)
* Present the live Kyverno `PolicyReport` compliance scorecards, OpenBao HSM audit trails, and Git-to-Supervisor traceability to Internal Audit and CISO stakeholders.
* Publish standardized VKS Cluster and Microservice GitOps templates for self-service onboarding across all banking business units.