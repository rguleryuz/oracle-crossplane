# oracle-crossplane

GitOps repository for learning Crossplane hands-on: Flux on a self-hosted
k3s cluster manages Crossplane, which provisions an **Always Free OCI NoSQL
table** via a composition + claim.

## Architecture

```
GitHub repo (public)
      │  flux bootstrap github
      ▼
k3s ── Flux (source / kustomize / helm controllers)
      ├── Crossplane (HelmRelease, ns crossplane-system)
      ├── provider-oci-nosql (Provider package → CRDs + controller)
      ├── ProviderConfig (creds via SecretRef)
      ├── XRD + Composition (platform definition)
      └── Claim (namespaced) → XR → NoSQL Table managed resource
                                        │
                                        ▼
                            OCI NoSQL Table (Always Free)
```

Two layered control loops: **Flux** reconciles git → cluster; **Crossplane**
reconciles XR → OCI API. Connection details (table OCID, endpoint) flow back
into a Kubernetes Secret for consuming apps.

- **Cluster:** self-hosted k3s `k3s-arm-cluster` (ARM64, Oracle Ampere, single node)
- **GitOps engine:** Flux (`flux bootstrap github`)
- **Crossplane:** v2 (namespaced managed resources)
- **OCI provider:** `ghcr.io/oracle/provider-oci-nosql:v1.4.0`
- **Composition function:** `function-patch-and-transform`

## Repository layout

```
clusters/k3s/        # Flux bootstrap + Kustomizations (infrastructure, apps)
infrastructure/      # Crossplane HelmRelease, OCI provider, ProviderConfig, function
platform/            # XRD (XNoSQLTable) + Composition
apps/demo/           # Demo claim (namespaced)
```

## WARNING: Always Free NoSQL limits

- **Always Free NoSQL tables are auto-reclaimed (deleted with data) after 90
  days of no reads/writes.**
- Maximum **3 free tables per region**.
- Fixed capacity: **50 read units / 50 write units / 25 GB storage** — these
  cannot be overridden (the Composition hardcodes them).

## Secrets

OCI credentials (API key PEM, tenancy/user OCIDs, fingerprint) are created
imperatively as a Kubernetes Secret (`oci-creds` in `crossplane-system`) and
are **never committed** to this repository. `.gitignore` excludes `*.pem`,
`*.ini`, and `.env.local`.
