# Enterprise Banking GitOps Live Demo Runbook (5-Act Presenter Guide)

**Target Audience:** CISO, VP of Engineering, Head of Cloud Platform, Internal/External Auditors (RBI / PCI-DSS)  
**Demo Stack:** Kubernetes (`kind`), ArgoCD, Kustomize, Kyverno, Kubernetes Native RBAC  
**Duration:** 25–30 Minutes (Live Interactive Demonstration)  

---

## 0. Pre-Demo Checklist (Run 30 Minutes Before Customer Meeting)

Complete this checklist before entering the boardroom so that your demo runs smoothly even on restricted conference room Wi-Fi:

- [ ] **Tools Installed:** Verify `docker`, `kind`, `kubectl`, `helm`, `argocd` CLI, and `git` are installed on your machine.
- [ ] **GitHub Repository Ready:** Ensure you have created a GitHub repository named `bank-payments-gitops` and have push access configured via SSH or PAT.
- [ ] **Pre-Pull Container Images:** Ensure `hashicorp/http-echo:1.0.0`, `nginx:latest`, ArgoCD images, and Kyverno images are cached in your local Kind cluster.
- [ ] **Port-Forwarding Active:** Verify `kubectl port-forward svc/argocd-server -n argocd 8080:443` is running in a background terminal tab and `https://localhost:8080` loads cleanly.
- [ ] **Browser Windows Arranged:**
  - **Browser Window 1 (Standard):** Logged into ArgoCD UI as **`alice-dev`** (`BankDev#2026`).
  - **Browser Window 2 (Incognito):** Logged into ArgoCD UI as **`bob-release`** (`BankCab#2026`).
- [ ] **Terminal Font Size:** Increase terminal font size to `16pt–18pt` so executives at the back of the room can clearly read the `Forbidden` and `COMPLIANCE VIOLATION` banners.

---

## 1. Demo Architecture & Banking Personas

During this demonstration, you will switch between **three distinct banking personas** to prove **Segregation of Duties (Maker-Checker)**, **Zero-Trust Kubernetes API Access**, and **Runtime Policy Enforcement**:

| Persona | Username | Password | Role in the Bank | Permitted Actions | Explicitly Blocked Actions |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Application Developer (Maker)** | `alice-dev` | `BankDev#2026` | Writes payment microservice code & deploys to `dev`. | • View/Sync `upi-dev` in ArgoCD.<br>• View pod logs in `payments-dev` via `kubectl`. | • **Cannot** Sync or Delete `upi-prod` in ArgoCD.<br>• **Cannot** access `payments-prod` via `kubectl`. |
| **CAB Release Manager (Checker)** | `bob-release` | `BankCab#2026` | Approves & triggers production releases during CAB windows. | • View all apps in ArgoCD.<br>• Trigger **Sync** on `upi-prod` in ArgoCD. | • **Cannot** Delete `upi-prod` in ArgoCD.<br>• **Cannot** bypass `AppProject` or Kyverno policies. |
| **Internal / RBI Auditor** | `charlie-auditor` | `BankAudit#2026` | Inspects release history, sync logs, and compliance reports. | • **Read-Only** (`get`) across all ArgoCD apps, projects, and logs. | • **Zero** mutation (`sync`, `delete`, `override`) across any environment. |

> **Presenter Note on Corporate SSO:** When presenting to the customer, explain that while local ArgoCD accounts (`alice-dev`, `bob-release`, `charlie-auditor`) are used here for an isolated demo environment, in production these exact Casbin RBAC role bindings map 1:1 to **Entra ID / Okta OIDC Security Groups** with hardware MFA.

---

## 2. Phase 1: Bootstrap the Cluster, ArgoCD & Kyverno

### Step 1.1: Create the Cluster and Target Namespaces
```bash
kind create cluster --name bank-gitops-demo

kubectl create namespace argocd
kubectl create namespace payments-dev
kubectl create namespace payments-prod
```

### Step 1.2: Install ArgoCD
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

### Step 1.3: Install Kyverno Admission Controller
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno -n kyverno --create-namespace \
  --set admissionController.replicas=1 \
  --set backgroundController.replicas=1 \
  --set cleanupController.replicas=1 \
  --set reportsController.replicas=1

kubectl wait --for=condition=Ready pods --all -n kyverno --timeout=180s
```

---

## 3. Phase 2: Create the Sample Banking GitOps Repository

Create a GitHub repository named `bank-payments-gitops` and populate it with the following Kustomize structure:

### Step 2.1: Initialize Local Directory Structure
```bash
mkdir -p bank-payments-gitops/{base,environments/{dev,prod}}
cd bank-payments-gitops
```

### Step 2.2: Create Hardened Base Manifests (`base/deployment.yaml` & `base/kustomization.yaml`)
```bash
cat <<'EOF' > base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upi-payment-service
  labels:
    app: upi-payment-service
    tier: core-banking
spec:
  replicas: 1
  selector:
    matchLabels:
      app: upi-payment-service
  template:
    metadata:
      labels:
        app: upi-payment-service
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
      containers:
        - name: payment-api
          image: hashicorp/http-echo:1.0.0
          args:
            - "-listen=:8080"
            - "-text={\"service\":\"UPI-Payment-Engine\",\"version\":\"v1.0.0\",\"status\":\"SECURE\"}"
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
---
apiVersion: v1
kind: Service
metadata:
  name: upi-payment-service
spec:
  selector:
    app: upi-payment-service
  ports:
    - port: 80
      targetPort: 8080
EOF

cat <<'EOF' > base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
EOF
```

### Step 2.3: Create Environment Overlays (`environments/dev` and `environments/prod`)
```bash
cat <<'EOF' > environments/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: payments-dev
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: upi-payment-service
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 1
      - op: replace
        path: /spec/template/spec/containers/0/args/1
        value: "-text={\"service\":\"UPI-Payment-Engine\",\"env\":\"DEV\",\"version\":\"v1.1.0-rc1\"}"
EOF

cat <<'EOF' > environments/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: payments-prod
resources:
  - ../../base
patches:
  - target:
      kind: Deployment
      name: upi-payment-service
    patch: |-
      - op: replace
        path: /spec/replicas
        value: 2
      - op: replace
        path: /spec/template/spec/containers/0/args/1
        value: "-text={\"service\":\"UPI-Payment-Engine\",\"env\":\"PROD-PCI\",\"version\":\"v1.0.0\"}"
EOF
```

### Step 2.4: Commit and Push to GitHub
```bash
git init
git branch -M main
git add .
git commit -m "feat: initial hardened UPI payment service manifests"
git remote add origin https://github.com/<YOUR_GITHUB_USER>/bank-payments-gitops.git
git push -u origin main
```

---

## 4. Phase 3: Configure Authentication, RBAC & Guardrails

### Step 3.1: Provision Banking Personas in ArgoCD (`argocd-cm`)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  accounts.alice-dev: apiKey, login
  accounts.bob-release: apiKey, login
  accounts.charlie-auditor: apiKey, login
  admin.enabled: "true"
EOF
```

### Step 3.2: Enforce Zero-Trust RBAC Matrix in ArgoCD (`argocd-rbac-cm`)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-rbac-cm
    app.kubernetes.io/part-of: argocd
data:
  policy.default: role:none
  policy.csv: |
    # 1. Developer Role (Alice): Full sync in nonprod, Read-Only in prod
    p, role:payments-dev, applications, get, payments-nonprod/*, allow
    p, role:payments-dev, applications, sync, payments-nonprod/*, allow
    p, role:payments-dev, logs, get, payments-nonprod/*, allow
    p, role:payments-dev, applications, get, payments-prod/*, allow
    p, role:payments-dev, applications, sync, payments-prod/*, deny
    p, role:payments-dev, applications, delete, */*, deny
    g, alice-dev, role:payments-dev

    # 2. Release Manager Role (Bob): Can view & sync Prod, CANNOT delete or override
    p, role:prod-operator, applications, get, */*, allow
    p, role:prod-operator, applications, sync, payments-prod/*, allow
    p, role:prod-operator, logs, get, payments-prod/*, allow
    p, role:prod-operator, applications, delete, payments-prod/*, deny
    p, role:prod-operator, applications, override, payments-prod/*, deny
    g, bob-release, role:prod-operator

    # 3. Internal Auditor Role (Charlie): Read-Only across the entire bank
    p, role:auditor, applications, get, */*, allow
    p, role:auditor, projects, get, *, allow
    p, role:auditor, repositories, get, *, allow
    p, role:auditor, clusters, get, *, allow
    p, role:auditor, logs, get, */*, allow
    g, charlie-auditor, role:auditor
EOF
```

### Step 3.3: Set Persona Passwords via ArgoCD CLI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
sleep 3

ADMIN_PASS=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
argocd login localhost:8080 --username admin --password "$ADMIN_PASS" --insecure

argocd account update-password --account alice-dev --current-password "$ADMIN_PASS" --new-password "BankDev#2026"
argocd account update-password --account bob-release --current-password "$ADMIN_PASS" --new-password "BankCab#2026"
argocd account update-password --account charlie-auditor --current-password "$ADMIN_PASS" --new-password "BankAudit#2026"
```

### Step 3.4: Configure Kubernetes Native RBAC (`payments-dev` vs `payments-prod`)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dev-readonly-role
  namespace: payments-dev
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "pods/log", "deployments", "services"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-dev-binding
  namespace: payments-dev
subjects:
  - kind: User
    name: alice-dev
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: dev-readonly-role
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Step 3.5: Create Hardened ArgoCD `AppProjects` and `Applications`
*(Replace `<YOUR_GITHUB_USER>` with your actual GitHub username before running)*
```bash
export GITHUB_USER="<YOUR_GITHUB_USER>"

cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments-nonprod
  namespace: argocd
spec:
  description: "Non-Production Payments Workloads"
  sourceRepos:
    - 'https://github.com/${GITHUB_USER}/bank-payments-gitops.git'
  destinations:
    - namespace: 'payments-dev'
      server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist: []
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments-prod
  namespace: argocd
spec:
  description: "PCI-DSS Production Payments Workloads"
  sourceRepos:
    - 'https://github.com/${GITHUB_USER}/bank-payments-gitops.git'
  destinations:
    - namespace: 'payments-prod'
      server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist: []
  # Strictly whitelist ONLY Deployment and Service kinds in Production
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: Deployment
    - group: ''
      kind: Service
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: upi-dev
  namespace: argocd
spec:
  project: payments-nonprod
  source:
    repoURL: 'https://github.com/${GITHUB_USER}/bank-payments-gitops.git'
    targetRevision: main
    path: environments/dev
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: payments-dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: upi-prod
  namespace: argocd
spec:
  project: payments-prod
  source:
    repoURL: 'https://github.com/${GITHUB_USER}/bank-payments-gitops.git'
    targetRevision: main
    path: environments/prod
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: payments-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### Step 3.6: Apply Kyverno PCI-DSS Admission Guardrail (`ClusterPolicy`)
```bash
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: bank-pci-workload-guardrails
  annotations:
    policies.kyverno.io/title: PCI-DSS Container Hardening & Registry Whitelist
    policies.kyverno.io/category: Banking Security
    policies.kyverno.io/severity: high
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    # RULE 1: Block Untrusted Container Registries
    - name: check-trusted-bank-registry
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - payments-dev
                - payments-prod
      validate:
        message: "COMPLIANCE VIOLATION [RBI-SEC-01]: Untrusted container registry detected. Workloads in payments namespaces may only pull from approved internal registries (hashicorp/* or ghcr.io/bank-internal/*)."
        pattern:
          spec:
            containers:
              - image: "hashicorp/* | ghcr.io/bank-internal/*"

    # RULE 2: Block Root Execution (Must enforce runAsNonRoot: true)
    - name: enforce-non-root-execution
      match:
        any:
          - resources:
              kinds:
                - Pod
              namespaces:
                - payments-dev
                - payments-prod
      validate:
        message: "COMPLIANCE VIOLATION [PCI-DSS-2.2]: Containers must explicitly set securityContext.runAsNonRoot=true and cannot execute as UID 0 (root)."
        anyPattern:
          - spec:
              securityContext:
                runAsNonRoot: true
          - spec:
              containers:
                - securityContext:
                    runAsNonRoot: true
EOF
```

---

## 5. Live Customer Execution Script: The 5-Act Walkthrough

### Act 1: Prove Zero-Trust Kubernetes API ("No Human Can Touch Production")
* **Executive Talk Track:** *"In regulated banking, our first control is eliminating human write access (`kubectl apply/edit/delete`) to UAT and Production clusters. Developers can inspect logs in Dev, but the Production Kubernetes API is completely sealed off to human users—only ArgoCD's internal ServiceAccount can mutate cluster state."*
* **Commands to Run Live:**
  ```bash
  # 1. Developer Alice can view her pods in Dev:
  kubectl get pods -n payments-dev --as=alice-dev

  # 2. Developer Alice cannot even list pods in Production:
  kubectl get pods -n payments-prod --as=alice-dev

  # 3. Developer Alice attempts to delete the Production UPI Payment service:
  kubectl delete deployment upi-payment-service -n payments-prod --as=alice-dev
  ```
* **Expected Output:** Immediate `Error from server (Forbidden)` on both production commands.

---

### Act 2: Prove Segregation of Duties in ArgoCD (Maker vs. Checker)
* **Executive Talk Track:** *"Next, we enforce the RBI Maker-Checker principle inside ArgoCD. Alice (the Developer) can sync changes in Dev, and she has visibility into Prod health, but she is cryptographically and permissionally blocked from deploying to or modifying Production."*
* **Live Action in Browser 1 (Logged in as `alice-dev`):**
  1. Click on `upi-dev` $\rightarrow$ Click **Sync** $\rightarrow$ Show that it succeeds.
  2. Click on `upi-prod` $\rightarrow$ Click **Sync** $\rightarrow$ Click **Synchronize**.
  * **Result on Screen:** Red banner: **`Failed to sync application: permission denied: applications, sync, payments-prod/upi-prod`**.
* **Live Action in Browser 2 (Logged in as `bob-release` - CAB Release Manager):**
  1. Switch to Browser 2 (`bob-release`).
  2. Click on `upi-prod` $\rightarrow$ Click **Sync** $\rightarrow$ Show that the CAB Release Manager's sync succeeds.
  3. Now, still as `bob-release`, click **Delete** at the top of `upi-prod` and type `upi-prod` to simulate an accidental deletion or insider threat.
  * **Result on Screen:** Red banner: **`permission denied: applications, delete, payments-prod/upi-prod`**. Even the Release Manager cannot destroy a production service.

---

### Act 3: Prove Multi-Tenant Blast-Radius Containment (`AppProject` Whitelist)
* **Executive Talk Track:** *"What if a developer and release manager accidentally approve a PR that introduces an unauthorized Kubernetes resource type—like a `ConfigMap`, `DaemonSet`, or `ResourceQuota` override—into the Production folder?"*
* **Commands to Run Live:**
  ```bash
  cat <<'EOF' > environments/prod/rogue-configmap.yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: unauthorized-prod-config
  data:
    debug_mode: "true"
  EOF

  sed -i.bak '/resources:/a\
    - rogue-configmap.yaml' environments/prod/kustomization.yaml

  git add environments/prod/
  git commit -m "chore(prod): attempt to inject unapproved ConfigMap resource"
  git push origin main
  ```
* **Live Action in ArgoCD UI:**
  Click **Refresh** on `upi-prod`.
* **Result on Screen:** ArgoCD flags a Sync Error: **`resource /v1, Kind=ConfigMap is not permitted in project payments-prod`**.
* **Clean Revert Before Act 4:**
  ```bash
  git revert HEAD --no-edit && git push origin main
  ```

---

### Act 4: Prove Real-Time Drift Detection & Self-Healing
* **Executive Talk Track:** *"What happens if an engineer uses an emergency Break-Glass root credential on a cluster node and manually scales down our Core Banking UPI Payment pods in Production?"*
* **Command to Run Live (Keep ArgoCD UI visible beside the terminal):**
  ```bash
  kubectl scale deployment upi-payment-service -n payments-prod --replicas=0
  ```
* **Result on Screen:** Watch the `upi-prod` topology tree in ArgoCD. Within 2 seconds, ArgoCD detects the unauthorized drift, flashes `OutOfSync`, and **automatically terminates the drift by spinning the 2 Production Payment pods right back up**—while logging an audit event for the SIEM.

---

### Act 5: Prove Runtime Admission Guardrails with Kyverno (Blocking Root & Untrusted Images)
* **Executive Talk Track:** *"Finally, let's address the ultimate CISO question: What if a bad change makes it through GitHub PR review AND gets synced by ArgoCD—for example, someone swaps our hardened bank container for a public `nginx:latest` image running as `root`? Kyverno acts as our final Zero-Trust gatekeeper at the Kubernetes API admission boundary."*

#### Part 5A: Instant CLI Proof
```bash
kubectl run rogue-debug-pod --image=nginx:latest -n payments-prod
```
* **Result on Screen:** Rejected immediately with **`COMPLIANCE VIOLATION [RBI-SEC-01]: Untrusted container registry detected`** and **`COMPLIANCE VIOLATION [PCI-DSS-2.2]: Containers must explicitly set securityContext.runAsNonRoot=true`**.

#### Part 5B: GitOps Pipeline Proof (Bad Commit Merged to `main`)
1. **Inject the Policy Violation into `environments/prod/kustomization.yaml`:**
   ```bash
   cat <<'EOF' > environments/prod/kustomization.yaml
   apiVersion: kustomize.config.k8s.io/v1beta1
   kind: Kustomization
   namespace: payments-prod
   resources:
     - ../../base
   patches:
     - target:
         kind: Deployment
         name: upi-payment-service
       patch: |-
         - op: replace
           path: /spec/replicas
           value: 2
         - op: replace
           path: /spec/template/spec/securityContext/runAsNonRoot
           value: false
         - op: replace
           path: /spec/template/spec/containers/0/image
           value: "nginx:latest"
   EOF

   git add environments/prod/kustomization.yaml
   git commit -m "chore(prod): simulate untrusted image and root container merge"
   git push origin main
   ```
2. **Click Refresh / Sync in ArgoCD UI:**
   * Show the customer that ArgoCD's sync **fails immediately (`SyncFailed`)** and surfaces the exact `[RBI-SEC-01]` and `[PCI-DSS-2.2]` violation messages directly inside the ArgoCD UI.
   * Run `kubectl get pods -n payments-prod` to prove that the **2 existing healthy `v1.0.0` Payment Pods remain up and serving traffic without a millisecond of downtime** while the rogue deployment is blocked.
3. **Revert Commit & Show the Auditor's Live Compliance Scorecard:**
   ```bash
   git revert HEAD --no-edit
   git push origin main

   # Show the live Kubernetes Compliance PolicyReport for the Auditor:
   kubectl get policyreport -n payments-prod -o wide
   ```

---

## 6. Post-Demo Teardown
When your customer session concludes, clean up the local environment with:
```bash
pkill -f "port-forward"
kind delete cluster --name bank-gitops-demo
```