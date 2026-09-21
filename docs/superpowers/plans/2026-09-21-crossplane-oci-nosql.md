# Crossplane + OCI NoSQL Always Free on k3s — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** GitOps repo where Flux installs Crossplane on k3s, and a namespaced XR provisions an Always Free OCI NoSQL table through an XRD + Composition.

**Architecture:** Flux bootstrapped against a public GitHub repo reconciles everything: Crossplane (HelmRelease), oracle/provider-family-oci + provider-oci-nosql v1.4.0, namespaced ProviderConfig, XRD + Composition (function-patch-and-transform), and a demo XR. Crossplane v2 namespaced model — **no Claim resource**: v2 removed claims except via `LegacyCluster` compat; a namespaced XR is the v2 idiom and serves the claim role from the spec.

**Tech Stack:** Flux v2, Crossplane v2 (Helm chart 2.x), ghcr.io/oracle/provider-family-oci:v1.4.0, ghcr.io/oracle/provider-oci-nosql:v1.4.0, xpkg.crossplane.io/crossplane-contrib/function-patch-and-transform:v0.9.0, k3s v1.36.4 (arm64).

**Spec:** `docs/superpowers/specs/2026-09-21-crossplane-oci-nosql-design.md`

**Spec deviations (confirmed during planning):**
- "Claim" → namespaced XR (Crossplane v2 removed claims outside LegacyCluster scope).
- Connection secret dropped: NoSQL endpoint is deterministic (`https://nosql.<region>.oci.oraclecloud.com`), table OCID visible in XR status. YAGNI.

## Global Constraints

- Repo: `oracle-crossplane`, public GitHub, branch `main`, Flux path `clusters/k3s`.
- No credentials in git. `.gitignore` covers `*.pem`, `*.ini`, `.env.local`.
- Table limits pinned: 50 read units, 50 write units, 25 GB storage, `isAutoReclaimable: true`.
- All kubectl commands run against context `default` (k3s at 141.144.234.213:6443).
- Commit attribution trailer on every commit: `Co-Authored-By: Claude Code <noreply@anthropic.com>`
- `<COMPARTMENT_OCID>` below is replaced with the real OCID collected in Task 0 before first commit of the file containing it.

---

### Task 0: Prereqs — tooling and OCI/GitHub inputs

**Files:**
- Create: `.env.local` (gitignored, local only)

**Interfaces:**
- Produces: env vars used by later tasks — `GITHUB_USER`, `GITHUB_TOKEN` (PAT, `repo` scope), `COMPARTMENT_OCID`, `OCI_TENANCY_OCID`, `OCI_USER_OCID`, `OCI_FINGERPRINT`, `OCI_PRIVATE_KEY_PATH`, `OCI_REGION`.

- [ ] **Step 1: Install Flux CLI**

```powershell
winget install fluxcd.flux
```

Verify: `flux version --client` → prints v2.x.

- [ ] **Step 2: OCI console (manual)** — create compartment `crossplane-demo`; create API key for your user (Identity → Domains → Users → API keys → Add); download private key PEM. Record tenancy OCID, user OCID, fingerprint, region (home region).

- [ ] **Step 3: OCI IAM policy (manual)** — Identity → Policies → create in root compartment:

```
Allow any-user to manage nosql-family in compartment crossplane-demo where all {request.principal.type='ApiKey', request.principal.id='<OCI_USER_OCID>'}
```

Simpler alternative if groups exist: `Allow group <group> to manage nosql-family in compartment crossplane-demo`.

- [ ] **Step 4: GitHub PAT (manual)** — GitHub → Settings → Developer settings → PAT (classic) with `repo` scope. Record it.

- [ ] **Step 5: Write `.env.local`** (repo root, never committed):

```bash
GITHUB_USER=<github-username>
GITHUB_TOKEN=<pat>
COMPARTMENT_OCID=ocid1.compartment.oc1..xxx
OCI_TENANCY_OCID=ocid1.tenancy.oc1..xxx
OCI_USER_OCID=ocid1.user.oc1..xxx
OCI_FINGERPRINT=aa:bb:...
OCI_PRIVATE_KEY_PATH=F:/crossplane-oracle/oci-api-key.pem
OCI_REGION=<e.g. eu-frankfurt-1>
```

- [ ] **Step 6: Verify** — `kubectl get nodes` shows `k3s-arm-cluster Ready`; `.env.local` exists and `git check-ignore .env.local` prints the path (after Task 1 .gitignore).

---

### Task 1: Repo scaffolding

**Files:**
- Create: `.gitignore`, `README.md`

**Interfaces:**
- Produces: repo ready for bootstrap; README documents the 90-day auto-reclaim warning.

- [ ] **Step 1: Write `.gitignore`**

```gitignore
*.pem
*.ini
.env.local
```

- [ ] **Step 2: Write `README.md`** — project purpose, architecture summary (from spec), **warning**: Always Free NoSQL tables are auto-reclaimed (deleted with data) after 90 days of no reads/writes; max 3 free tables per region, fixed 50 RU / 50 WU / 25 GB.

- [ ] **Step 3: Create GitHub repo and push**

```bash
source .env.local 2>/dev/null || export $(grep -v '^#' .env.local | xargs)
gh auth login  # or gh auth token via PAT
gh repo create oracle-crossplane --public --source=. --remote=origin --push
```

Verify: `git ls-remote origin` lists `main`.

- [ ] **Step 4: Commit**

```bash
git add .gitignore README.md
git commit -m "chore: repo scaffolding

Co-Authored-By: Claude Code <noreply@anthropic.com>"
git push
```

---

### Task 2: Flux bootstrap

**Files:**
- Create (generated): `clusters/k3s/flux-system/*`

**Interfaces:**
- Produces: Flux controllers running; `clusters/k3s/` is the GitOps root later Kustomizations hang off.

- [ ] **Step 1: Bootstrap**

```bash
source .env.local 2>/dev/null || export $(grep -v '^#' .env.local | xargs)
flux bootstrap github --owner="$GITHUB_USER" --repository=oracle-crossplane \
  --branch=main --path=clusters/k3s --personal --public
```

- [ ] **Step 2: Verify**

Run: `flux get sources all && flux get kustomizations`
Expected: `gitrepository/flux-system` Ready=True, `kustomization/flux-system` Ready=True.

Run: `kubectl get pods -n flux-system`
Expected: source-controller, kustomize-controller, helm-controller Running.

- [ ] **Step 3: Commit** — bootstrap already committed `flux-system/` to git; pull locally:

```bash
git pull --rebase
```

---

### Task 3: Crossplane install via Flux

**Files:**
- Create: `infrastructure/crossplane/helmrepository.yaml`
- Create: `infrastructure/crossplane/helmrelease.yaml`
- Create: `infrastructure/crossplane/kustomization.yaml`
- Create: `clusters/k3s/infrastructure.yaml`

**Interfaces:**
- Produces: Flux Kustomization named `infrastructure` (path `./infrastructure`, recursive) that later tasks add manifests to; Task 6 `apps` Kustomization `dependsOn: infrastructure`.

- [ ] **Step 1: Write `infrastructure/crossplane/helmrepository.yaml`**

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: crossplane-stable
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.crossplane.io/stable
```

- [ ] **Step 2: Write `infrastructure/crossplane/helmrelease.yaml`**

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: crossplane
  namespace: flux-system
spec:
  interval: 5m
  chart:
    spec:
      chart: crossplane
      version: ">=2.0.0 <3.0.0"
      sourceRef:
        kind: HelmRepository
        name: crossplane-stable
        namespace: flux-system
  targetNamespace: crossplane-system
  install:
    createNamespace: true
```

- [ ] **Step 3: Write `infrastructure/crossplane/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - helmrepository.yaml
  - helmrelease.yaml
```

- [ ] **Step 4: Write `clusters/k3s/infrastructure.yaml`**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m
  path: ./infrastructure
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  validation: client
```

Note: `path: ./infrastructure` requires a root `infrastructure/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - crossplane
  - oci-provider
```

(Write this as `infrastructure/kustomization.yaml`; `oci-provider` dir arrives in Task 4 — commit it there, or Flux reports a build error until then. Order: commit Task 4 before this file, or accept transient error.)

- [ ] **Step 5: Verify**

```bash
git add -A && git commit -m "feat: crossplane install via Flux

Co-Authored-By: Claude Code <noreply@anthropic.com>" && git push
flux reconcile kustomization infrastructure
kubectl get pods -n crossplane-system
```

Expected: crossplane + crossplane-rbac-manager pods Running; `kubectl api-resources | grep crossplane.io` lists `CompositeResourceDefinition`, `Composition`, `Provider`, `Function`.

---

### Task 4: OCI creds secret + providers + ProviderConfig

**Files:**
- Create: `infrastructure/oci-provider/provider-family.yaml`
- Create: `infrastructure/oci-provider/provider-nosql.yaml`
- Create: `infrastructure/oci-provider/function-patch-and-transform.yaml`
- Create: `infrastructure/oci-provider/providerconfig.yaml`
- Create: `infrastructure/oci-provider/kustomization.yaml`
- Local only: namespace `demo` + Secret `oci-creds` (imperative)

**Interfaces:**
- Produces: namespaced `ProviderConfig` named `default` in ns `demo` (group `oci.m.upbound.io/v1beta1`); Table CRD `tables.nosql.oci.m.upbound.io` available; composition function installed.

- [ ] **Step 1: Write provider manifests**

`provider-family.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: oracle-provider-family-oci
spec:
  package: ghcr.io/oracle/provider-family-oci:v1.4.0
```

`provider-nosql.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-oci-nosql
spec:
  package: ghcr.io/oracle/provider-oci-nosql:v1.4.0
```

`function-patch-and-transform.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-patch-and-transform:v0.9.0
```

- [ ] **Step 2: Write `providerconfig.yaml`**

```yaml
apiVersion: oci.m.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
  namespace: demo
spec:
  credentials:
    source: Secret
    secretRef:
      name: oci-creds
      namespace: demo
      key: credentials
```

- [ ] **Step 3: Write `infrastructure/oci-provider/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - provider-family.yaml
  - provider-nosql.yaml
  - function-patch-and-transform.yaml
  - providerconfig.yaml
```

- [ ] **Step 4: Create namespace + creds secret imperatively**

```bash
source .env.local 2>/dev/null || export $(grep -v '^#' .env.local | xargs)
kubectl create namespace demo
# build JSON with PEM newlines escaped
PRIVATE_KEY=$(awk '{printf "%s\\n", $0}' "$OCI_PRIVATE_KEY_PATH")
kubectl -n demo create secret generic oci-creds --from-literal=credentials="{
  \"tenancy_ocid\": \"$OCI_TENANCY_OCID\",
  \"user_ocid\": \"$OCI_USER_OCID\",
  \"fingerprint\": \"$OCI_FINGERPRINT\",
  \"private_key\": \"$PRIVATE_KEY\",
  \"region\": \"$OCI_REGION\",
  \"auth\": \"ApiKey\"
}"
```

- [ ] **Step 5: Commit and reconcile**

```bash
git add -A && git commit -m "feat: OCI provider family, NoSQL provider, ProviderConfig

Co-Authored-By: Claude Code <noreply@anthropic.com>" && git push
flux reconcile kustomization infrastructure
```

- [ ] **Step 6: Verify**

```bash
kubectl get providers.pkg.crossplane.io
kubectl get functions.pkg.crossplane.io
kubectl get crd | grep nosql
```

Expected: both providers INSTALLED=True HEALTHY=True; function INSTALLED/HEALTHY; CRD `tables.nosql.oci.m.upbound.io` exists. If `provider-family-oci:v1.4.0` pull fails, run `kubectl describe provider oracle-provider-family-oci`, check tag against `ghcr.io/oracle/provider-family-oci` package page, pin to latest existing.

Also inspect Table schema for exact field names used in Task 5:

```bash
kubectl explain table.nosql.oci.m.upbound.io.spec.forProvider --recursive | grep -iE "ddl|limit|unit|storage|reclaim|compartment|name"
```

---

### Task 5: XRD + Composition

**Files:**
- Create: `platform/xrd.yaml`
- Create: `platform/composition.yaml`
- Create: `platform/kustomization.yaml`
- Modify: `infrastructure/kustomization.yaml` (add `- ../platform` — no; instead platform gets its own Flux Kustomization in Task 6 to keep platform separate from infrastructure; add `clusters/k3s/platform.yaml`)

**Interfaces:**
- Consumes: `ProviderConfig default` in ns `demo` (Task 4), Table CRD field names (Task 4 Step 6).
- Produces: namespaced XR API `nosqltables.platform.example.com/v1alpha1`, kind `NoSQLTable`, with `spec.tableName` and `spec.ddl`; Composition `nosql-free` hardcoding free-tier limits.

- [ ] **Step 1: Write `platform/xrd.yaml`**

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: nosqltables.platform.example.com
spec:
  scope: Namespaced
  group: platform.example.com
  names:
    kind: NoSQLTable
    plural: nosqltables
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required: ["tableName", "ddl"]
              properties:
                tableName:
                  type: string
                ddl:
                  type: string
            status:
              type: object
              properties:
                tableOCID:
                  type: string
                lifecycleState:
                  type: string
```

- [ ] **Step 2: Write `platform/composition.yaml`**

Replace `<COMPARTMENT_OCID>` with the Task 0 value before committing.

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: nosql-free
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: NoSQLTable
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: table
            base:
              apiVersion: nosql.oci.m.upbound.io/v1alpha1
              kind: Table
              spec:
                providerConfigRef:
                  name: default
                  kind: ProviderConfig
                forProvider:
                  compartmentId: <COMPARTMENT_OCID>
                  isAutoReclaimable: true
                  tableLimits:
                    - maxReadUnits: 50
                      maxWriteUnits: 50
                      maxStorageInGBs: 25
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.tableName
                toFieldPath: spec.forProvider.name
              - type: FromCompositeFieldPath
                fromFieldPath: spec.ddl
                toFieldPath: spec.forProvider.ddlStatement
              - type: ToCompositeFieldPath
                fromFieldPath: status.atProvider.id
                toFieldPath: status.tableOCID
              - type: ToCompositeFieldPath
                fromFieldPath: status.atProvider.lifecycleState
                toFieldPath: status.lifecycleState
```

Note: `tableLimits` shown as list — confirm scalar-vs-list shape from Task 4 Step 6 `kubectl explain` output and adjust (`tableLimits:` object without `-` if not a list).

- [ ] **Step 3: Write `platform/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - xrd.yaml
  - composition.yaml
```

- [ ] **Step 4: Server-side dry-run before commit**

```bash
kubectl apply --dry-run=server -f platform/
```

Expected: `configured (server dry run)` for both, no validation errors.

- [ ] **Step 5: Wire Flux Kustomization `clusters/k3s/platform.yaml`**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: platform
  namespace: flux-system
spec:
  interval: 10m
  path: ./platform
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:
    - name: infrastructure
```

- [ ] **Step 6: Commit, reconcile, verify**

```bash
git add -A && git commit -m "feat: XRD + composition for free-tier NoSQL

Co-Authored-By: Claude Code <noreply@anthropic.com>" && git push
flux reconcile kustomization platform
kubectl get xrd
kubectl get composition
```

Expected: XRD ESTABLISHED=True, OFFERED blank/none for namespaced scope; composition exists. `kubectl api-resources | grep nosqltables` shows NAMESPACED=true.

---

### Task 6: Demo XR → real Always Free table

**Files:**
- Create: `apps/demo/xr.yaml`
- Create: `apps/demo/kustomization.yaml`
- Create: `clusters/k3s/apps.yaml`

**Interfaces:**
- Consumes: `NoSQLTable` XR API (Task 5).

- [ ] **Step 1: Write `apps/demo/xr.yaml`**

```yaml
apiVersion: platform.example.com/v1alpha1
kind: NoSQLTable
metadata:
  name: demo-table
  namespace: demo
spec:
  tableName: crossplane_demo
  ddl: "CREATE TABLE IF NOT EXISTS crossplane_demo (id INTEGER, name STRING, PRIMARY KEY (id))"
```

- [ ] **Step 2: Write `apps/demo/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: demo
resources:
  - xr.yaml
```

- [ ] **Step 3: Write `clusters/k3s/apps.yaml`**

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m
  path: ./apps/demo
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:
    - name: platform
```

- [ ] **Step 4: Commit, reconcile**

```bash
git add -A && git commit -m "feat: demo NoSQL XR

Co-Authored-By: Claude Code <noreply@anthropic.com>" && git push
flux reconcile kustomization apps
```

- [ ] **Step 5: Verify chain**

```bash
kubectl get nosqltable -n demo
kubectl get table.nosql.oci.m.upbound.io -n demo
kubectl describe nosqltable demo-table -n demo
```

Expected: XR READY=True (after OCI provisions, can take a few minutes); Table SYNCED=True READY=True; `status.tableOCID` populated. If Table stuck: `kubectl describe table ... -n demo` for OCI API errors (auth/policy/compartment typos).

- [ ] **Step 6: Verify in OCI console (manual)** — NoSQL → Tables in compartment `crossplane-demo`: table `crossplane_demo` exists, Always Free badge shown. Screenshot/record.

---

### Task 7: Negative test + teardown check + docs

**Files:**
- Modify: `README.md` (add verification results, teardown instructions)

- [ ] **Step 1: Negative test — remove XR from git**

```bash
git rm apps/demo/xr.yaml
git commit -m "test: remove demo XR (negative test)

Co-Authored-By: Claude Code <noreply@anthropic.com>" && git push
flux reconcile kustomization apps
kubectl get table.nosql.oci.m.upbound.io -n demo -w
```

Expected: Flux prunes XR; Crossplane finalizer deletes managed resource; OCI table deprovisioned (console confirms).

- [ ] **Step 2: Restore XR** (git revert the removal commit; table recreated — proves GitOps loop end to end).

- [ ] **Step 3: Update README** — verification evidence, teardown order (delete XR first, then providers, then `flux uninstall`), pointer to spec and this plan.

- [ ] **Step 4: Final commit + verify**

```bash
git add -A && git commit -m "docs: verification results and teardown

Co-Authored-By: Claude Code <noreply@anthropic.com>" && git push
flux get kustomizations
```

Expected: all Kustomizations Ready=True.

---

## Self-Review

- **Spec coverage:** repo layout ✔ (Tasks 1–6), Flux ordering ✔ (dependsOn chain), provider + ProviderConfig ✔ (Task 4), XRD/Composition ✔ (Task 5), free-tier pinning ✔ (composition hardcodes caps; XRD schema has no capacity fields), imperative secret ✔ (Task 4 Step 4), verification plan ✔ (per-task + Task 7), risks (arm64 images ✔ confirmed multi-arch; outbound 443 — surfaced as failure mode in Task 6 Step 5), 90-day reclaim warning ✔ (README Task 1).
- **Spec deviations recorded in header:** namespaced XR instead of Claim; no connection secret.
- **Placeholder scan:** `<COMPARTMENT_OCID>` is a deliberate input slot with an explicit replacement step (Task 5 Step 2) — collected in Task 0. `.env.local` values are user inputs by design.
- **Type consistency:** `ProviderConfig` ns `demo` name `default` used identically in Task 4 and composition `providerConfigRef`; XR group/kind `platform.example.com/v1alpha1 NoSQLTable` consistent in XRD, Composition, and demo XR.
