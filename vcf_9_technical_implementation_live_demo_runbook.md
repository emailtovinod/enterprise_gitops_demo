# VCF 9 / 9.1 Technical Implementation Pack & Live Customer Demo Runbook

**Target Environment:** Broadcom VMware Cloud Foundation (VCF 9.0 / 9.1), vSphere Supervisor, vSphere Kubernetes Service (VKS)  
**Core Demo Stack:** ArgoCD, OpenBao (with PKCS#11 HSM Auto-Unseal), External Secrets Operator (ESO), Antrea CNI, VMware Avi Load Balancer (`AKO`), Kyverno  
**Audience:** Bank CISO, Head of Infrastructure / VI Admins, Platform SRE Leads, Internal/External Auditors (RBI / PCI-DSS)  

---

## 0. Pre-Demo Checklist (VCF 9 / VKS Customer Presentation)

Complete these checks 30 minutes prior to the customer session:

- [ ] **Kubeconfig Contexts Ready:**
  - Context 1 (`supervisor-ctx`): Authenticated against the **vSphere Supervisor Control Plane** (or simulated management cluster) to show declarative CAPI `Cluster` manifests.
  - Context 2 (`vks-prod-payments-01`): Authenticated against the **VKS Workload Cluster** where ArgoCD, OpenBao, ESO, and the sample UPI Payment workload run.
- [ ] **OpenBao & ESO Running:** Verify pods in `openbao` and `external-secrets` namespaces are `Running` and `Ready`.
- [ ] **ArgoCD UI Port-Forward Active:** Ensure `https://localhost:8080` is reachable and pre-logged in with banking personas (`alice-dev` in standard window, `bob-release` in Incognito window).
- [ ] **Git Repository Cloned Locally:** Have the `bank-payments-gitops` and `vcf-supervisor-fleet-gitops` repositories open in your IDE/terminal for live commits.

---

## 1. OpenBao Production HSM Config & ESO Kubernetes Auth Setup

In an on-premises Tier-1 banking environment, **OpenBao** eliminates cloud KMS dependencies by running in High Availability (HA) mode backed by an on-premises **FIPS 140-3 Level 3 Hardware Security Module (Thales Luna / Entrust nShield / Utimaco)** via PKCS#11. **External Secrets Operator (ESO)** runs inside each VKS cluster and authenticates to OpenBao using short-lived Kubernetes ServiceAccount JWT tokens.

### A. Production OpenBao Server Configuration (`openbao-server.hcl`)
Deploy this configuration across a 3-node or 5-node OpenBao cluster running on hardened VM Service VMs or a dedicated Security Management VKS cluster:

```hcl
ui = true
cluster_name = "bank-vcf9-openbao-prod"
disable_mlock = true

# 1. Integrated Raft Storage Backend (Encrypted at Rest on vSAN ESA)
storage "raft" {
  path    = "/opt/openbao/data"
  node_id = "openbao-mum-node-1"

  retry_join {
    leader_api_addr         = "https://openbao-mum-node-1.internal.bank.com:8200"
    leader_ca_cert_file     = "/opt/openbao/tls/bank-root-ca.pem"
    leader_client_cert_file = "/opt/openbao/tls/node-cert.pem"
    leader_client_key_file  = "/opt/openbao/tls/node-key.pem"
  }
  retry_join {
    leader_api_addr         = "https://openbao-mum-node-2.internal.bank.com:8200"
    leader_ca_cert_file     = "/opt/openbao/tls/bank-root-ca.pem"
    leader_client_cert_file = "/opt/openbao/tls/node-cert.pem"
    leader_client_key_file  = "/opt/openbao/tls/node-key.pem"
  }
  retry_join {
    leader_api_addr         = "https://openbao-mum-node-3.internal.bank.com:8200"
    leader_ca_cert_file     = "/opt/openbao/tls/bank-root-ca.pem"
    leader_client_cert_file = "/opt/openbao/tls/node-cert.pem"
    leader_client_key_file  = "/opt/openbao/tls/node-key.pem"
  }
}

# 2. Mutual TLS 1.3 Listener
listener "tcp" {
  address                  = "0.0.0.0:8200"
  cluster_address          = "0.0.0.0:8201"
  tls_cert_file            = "/opt/openbao/tls/server.crt"
  tls_key_file             = "/opt/openbao/tls/server.key"
  tls_client_ca_file       = "/opt/openbao/tls/bank-root-ca.pem"
  tls_min_version          = "tls13"
}

# 3. On-Premises FIPS 140-3 Level 3 HSM Auto-Unseal (PKCS#11)
seal "pkcs11" {
  lib            = "/usr/safenet/lunaclient/lib/libCryptoki2_64.so"
  slot           = "0"
  pin            = "env://BAO_HSM_PIN"
  key_label      = "openbao-prod-master-unseal-key"
  hmac_key_label = "openbao-prod-hmac-key"
  generate_key   = "false"
}

api_addr     = "https://openbao.internal.bank.com:8200"
cluster_addr = "https://openbao-mum-node-1.internal.bank.com:8201"
```

### B. Step-by-Step OpenBao Kubernetes Auth Binding for VKS
Run these commands to bind your VKS cluster (`vks-prod-payments-01`) to a least-privilege secret path in OpenBao:

```bash
export BAO_ADDR="https://openbao.internal.bank.com:8200"

# 1. Enable Mandatory Audit Logging to Bank SIEM (Aria Operations for Logs / Splunk)
bao audit enable syslog tag="openbao-prod" facility="AUTH"

# 2. Enable KV-v2 Secret Engine and Database Dynamic Secrets
bao secrets enable -path=kv-prod kv-v2
bao secrets enable database

# 3. Seed a sample Core Banking UPI Payment Switch credential
bao kv put kv-prod/payments/upi-switch \
  merchant_id="BANK-UPI-PROD-991" \
  signing_key_passphrase="9f86d081884c7d659a2feaa0c55ad015"

# 4. Create a strict Read-Only Policy for the Payments Production Namespace
bao policy write vks-payments-prod-read - <<EOF
path "kv-prod/data/payments/*" {
  capabilities = ["read"]
}
EOF

# 5. Enable Kubernetes Auth Path for the target VKS Cluster
bao auth enable -path=vks-prod-payments-01 kubernetes

# 6. Extract the VKS API Server endpoint and CA Certificate from the VKS kubeconfig
VKS_HOST=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.server}')
VKS_CA_CERT=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d)

# 7. Configure OpenBao to validate VKS TokenReview requests
bao write auth/vks-prod-payments-01/config \
  kubernetes_host="$VKS_HOST" \
  kubernetes_ca_cert="$VKS_CA_CERT" \
  disable_local_ca_jwt="true"

# 8. Bind the OpenBao Role strictly to the ESO ServiceAccount in 'payments-prod' namespace
bao write auth/vks-prod-payments-01/role/payments-eso-role \
  bound_service_account_names="eso-payments-sa" \
  bound_service_account_namespaces="payments-prod" \
  policies="vks-payments-prod-read" \
  ttl="15m" \
  max_ttl="1h"
```

### C. VKS `SecretStore` & `ExternalSecret` Manifests (Committed to Git)
Commit these manifests to the `app-config-payments` repository so ArgoCD reconciles them into `payments-prod`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eso-payments-sa
  namespace: payments-prod
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: openbao-payments-store
  namespace: payments-prod
spec:
  provider:
    vault: # OpenBao is 100% API-compatible with the ESO Vault provider
      server: "https://openbao.internal.bank.com:8200"
      path: "kv-prod"
      version: "v2"
      caProvider:
        type: "ConfigMap"
        name: "bank-root-ca-bundle"
        key: "ca.crt"
      auth:
        kubernetes:
          mountPath: "vks-prod-payments-01"
          role: "payments-eso-role"
          serviceAccountRef:
            name: "eso-payments-sa"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: upi-switch-credentials
  namespace: payments-prod
spec:
  refreshInterval: "15m" # Automatically pulls rotated credentials every 15 minutes
  secretStoreRef:
    name: openbao-payments-store
    kind: SecretStore
  target:
    name: upi-switch-k8s-secret
    creationPolicy: Owner
    template:
      type: Opaque
  data:
    - secretKey: UPI_MERCHANT_ID
      remoteRef:
        key: payments/upi-switch
        property: merchant_id
    - secretKey: UPI_SIGNING_PASSPHRASE
      remoteRef:
        key: payments/upi-switch
        property: signing_key_passphrase
```

---

## 2. ArgoCD `ApplicationSet` for Automated VKS Cluster Bootstrapping

When the vSphere Supervisor provisions a new VKS cluster via Cluster API (`cluster.x-k8s.io/v1beta1`), the cluster is registered in the Hub ArgoCD instance with specific metadata labels (`vcf.bank.com/environment: prod-pci`).

This **Matrix `ApplicationSet`** watches all registered VKS clusters and automatically bootstraps the mandatory Day-2 banking security stack (**Kyverno, External Secrets Operator, Antrea NetworkPolicies, and Avi Kubernetes Operator**) in strict Sync-Wave order:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: vks-pci-day2-baseline-bootstrap
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - matrix:
        generators:
          # Generator 1: Match all VCF 9 VKS Clusters labeled for Production PCI
          - clusters:
              selector:
                matchLabels:
                  vcf.bank.com/cluster-type: "vks"
                  vcf.bank.com/environment: "prod-pci"
          # Generator 2: Deploy the 4 Mandatory Banking Platform Add-ons in Sync-Wave Order
          - list:
              elements:
                - addonName: "kyverno-admission-guardrails"
                  targetNamespace: "kyverno"
                  gitPath: "platform-addons/kyverno/overlays/prod-pci"
                  syncWave: "1"
                - addonName: "openbao-external-secrets-operator"
                  targetNamespace: "external-secrets"
                  gitPath: "platform-addons/external-secrets/overlays/prod-pci"
                  syncWave: "2"
                - addonName: "antrea-zero-trust-microsegmentation"
                  targetNamespace: "kube-system"
                  gitPath: "platform-addons/antrea-policies/overlays/prod-pci"
                  syncWave: "3"
                - addonName: "vmware-avi-ako-waf-operator"
                  targetNamespace: "avi-system"
                  gitPath: "platform-addons/avi-ako/overlays/prod-pci"
                  syncWave: "4"
  template:
    metadata:
      name: '{{.name}}-{{.addonName}}'
      annotations:
        argocd.argoproj.io/sync-wave: '{{.syncWave}}'
      finalizers:
        - resources-finalizer.argocd.argoproj.io
    spec:
      project: platform-vks-bootstrap
      source:
        repoURL: 'https://github.internal.bank.com/platform-sre/vcf9-platform-gitops.git'
        targetRevision: main
        path: '{{.gitPath}}'
      destination:
        server: '{{.server}}'
        namespace: '{{.targetNamespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```

---

## 3. Live Demo Environment Bootstrap (Local or VCF 9 Lab)

Run these commands prior to your customer presentation to stage **OpenBao**, **External Secrets Operator**, and the **Active Secret Store** inside your demo cluster:

```bash
# 1. Install OpenBao & External Secrets Operator via Helm
helm repo add openbao https://openbao.github.io/openbao-helm
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install openbao openbao/openbao -n openbao --create-namespace \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=bank-root-demo-token"

helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace --set installCRDs=true

kubectl wait --for=condition=Ready pods --all -n openbao --timeout=120s
kubectl wait --for=condition=Ready pods --all -n external-secrets --timeout=120s

# 2. Seed the Demo UPI Switch Credential inside OpenBao
kubectl exec -n openbao openbao-0 -- bao secrets enable -path=kv-prod kv-v2
kubectl exec -n openbao openbao-0 -- bao kv put kv-prod/payments/upi-switch \
  merchant_id="VCF9-UPI-MUMBAI-001" \
  signing_key_passphrase="Initial-HSM-Backed-Key-v1"

# 3. Create the Demo SecretStore & ExternalSecret in payments-prod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: openbao-demo-token
  namespace: payments-prod
stringData:
  token: "bank-root-demo-token"
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: openbao-payments-store
  namespace: payments-prod
spec:
  provider:
    vault:
      server: "http://openbao.openbao.svc.cluster.local:8200"
      path: "kv-prod"
      version: "v2"
      auth:
        tokenSecretRef:
          name: "openbao-demo-token"
          key: "token"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: upi-switch-credentials
  namespace: payments-prod
spec:
  refreshInterval: "1m"
  secretStoreRef:
    name: openbao-payments-store
    kind: SecretStore
  target:
    name: upi-switch-k8s-secret
    creationPolicy: Owner
  data:
    - secretKey: UPI_MERCHANT_ID
      remoteRef:
        key: payments/upi-switch
        property: merchant_id
    - secretKey: UPI_SIGNING_PASSPHRASE
      remoteRef:
        key: payments/upi-switch
        property: signing_key_passphrase
EOF
```

---

## 4. The 4-Act VCF 9 Executive Demo Script (What to Show & Say)

### Act 1: Declarative VKS Cluster & Node Pool Lifecycle via vSphere Supervisor
* **Executive Talk Track:** *"In VCF 9, our VI Admins allocate compute, encrypted vSAN storage policies, and NSX boundaries inside a Supervisor Namespace. From that moment on, Platform SREs never click around in vCenter to provision or scale Kubernetes clusters—ArgoCD reconciles the upstream Cluster API manifest directly against the vSphere Supervisor."*
* **Live Action:**
  1. Open `prod-pci/dc1-mum-payments-vks01.yaml` in your IDE and highlight `class: tanzukubernetescluster`, `version: v1.33.1+vmware.1-fips.1`, `vmClass: guaranteed-large`, and `storageClass: vsan-esa-raid6-encrypted-policy`.
  2. Edit `workers.machineDeployments[0].replicas` from `4` to `5` (or upgrade the VKr version string) and commit to Git.
  3. Show ArgoCD syncing the `Cluster` resource against the vSphere Supervisor, triggering vCenter to automatically clone a hardened, air-gapped Ubuntu FIPS node OVA from the local Content Library.

---

### Act 2: Zero-Secret GitOps & Live Key Rotation with OpenBao + ESO
* **Executive Talk Track:** *"Banking regulators strictly prohibit storing database passwords or HSM pins in Git—even if base64 encoded. In our `app-config-payments` repository, there is zero `kind: Secret` in Git—only an `ExternalSecret` metadata pointer. Let's watch OpenBao dynamically inject and rotate a live UPI payment switch key without touching Git."*
* **Commands to Run Live:**
  ```bash
  # 1. Verify the ExternalSecret is synced and inspect the initial secret value:
  kubectl get externalsecret -n payments-prod
  kubectl get secret upi-switch-k8s-secret -n payments-prod -o jsonpath='{.data.UPI_SIGNING_PASSPHRASE}' | base64 -d ; echo
  ```
  *Expected Output:* `Initial-HSM-Backed-Key-v1`

  ```bash
  # 2. Simulate an emergency key rotation inside OpenBao by the SecOps team:
  kubectl exec -n openbao openbao-0 -- bao kv put kv-prod/payments/upi-switch \
    merchant_id="VCF9-UPI-MUMBAI-001" \
    signing_key_passphrase="Rotated-Zero-Downtime-Key-v2"

  # 3. Trigger an immediate reconciliation and inspect the updated Kubernetes Secret:
  kubectl annotate externalsecret upi-switch-credentials -n payments-prod force-sync=$(date +%s) --overwrite
  sleep 2
  kubectl get secret upi-switch-k8s-secret -n payments-prod -o jsonpath='{.data.UPI_SIGNING_PASSPHRASE}' | base64 -d ; echo
  ```
  *Expected Output:* `Rotated-Zero-Downtime-Key-v2`
* **Customer Takeaway:** Point out that SecOps rotated a Tier-1 payment credential centrally in OpenBao, and the VKS cluster absorbed the new secret automatically with zero developer intervention, zero Git commits, and zero cleartext exposure.

---

### Act 3: Zero-Trust East-West Micro-Segmentation via Antrea & NSX
* **Executive Talk Track:** *"In VCF 9, VKS clusters run Antrea CNI natively integrated with NSX and VMware vDefend. Every production namespace defaults to Zero-Trust deny-all. Let's apply our GitOps-managed NetworkPolicy so that only the API Gateway can talk to the UPI Payment Engine on port 8080, blocking any lateral movement from other pods."*
* **Manifest to Show & Apply (`antrea-zero-trust-policy.yaml`):**
  ```bash
  cat <<EOF | kubectl apply -f -
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: upi-payments-zero-trust-isolation
    namespace: payments-prod
  spec:
    podSelector:
      matchLabels:
        app: upi-payment-service
    policyTypes:
      - Ingress
      - Egress
    ingress:
      - from:
          - podSelector:
              matchLabels:
                role: approved-api-gateway
        ports:
          - protocol: TCP
            port: 8080
  EOF
  ```
* **Live Verification Test:**
  ```bash
  # Attempt an unauthorized lateral connection from a pod without the 'role=approved-api-gateway' label:
  kubectl run lateral-attacker --rm -it --restart=Never --image=hashicorp/http-echo:1.0.0 \
    -n payments-prod --overrides='{"spec":{"securityContext":{"runAsNonRoot":true,"runAsUser":1000}}}' \
    -- -version
  ```
  *(Show how Antrea + Kyverno work in tandem: Kyverno enforces `runAsNonRoot: true`, and Antrea drops any unauthorized East-West socket connections to `upi-payment-service:80`).*

---

### Act 4: Declarative WAF & TLS 1.3 Enforcement via VMware Avi (`HostRule`)
* **Executive Talk Track:** *"Finally, exposing a banking payment API requires a PCI-DSS Web Application Firewall (WAF) and strict TLS 1.3 cipher enforcement. Instead of raising manual tickets for the network team to configure load balancers by hand, our GitOps repository defines a VMware Avi `HostRule` CRD right next to the application deployment."*
* **Manifest to Show in Git (`environments/prod/avi-waf-hostrule.yaml`):**
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
      wafPolicy: Bank-PCI-Strict-WAF-Policy
      sslProfile: Bank-TLS13-Strict-Profile
      httpPolicy:
        policySets:
          - Bank-UPI-RateLimit-PolicySet
  ```
* **Customer Takeaway:** Explain that because this `HostRule` manifest lives in `/environments/prod/`, GitHub `CODEOWNERS` enforces a mandatory review from `@bank-org/network-security` and `@bank-org/cab-release-managers` before ArgoCD syncs it to the Avi Kubernetes Operator (AKO)—giving the bank 100% auditable "WAF-as-Code."