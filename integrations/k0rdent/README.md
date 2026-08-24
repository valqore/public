# Govern your AI factory with Valqore on k0rdent

[k0rdent](https://k0rdent.io) (Mirantis) builds and runs the AI factory — GPU
clusters, sovereign clouds, inference. **Valqore governs what runs on it**:
security, cost, carbon, compliance, and the AI agents themselves, in one
deterministic verdict with re-runnable OSCAL evidence. It deploys as a k0rdent
service and runs entirely inside your environment — nothing egresses, which is
exactly what a sovereign or regulated k0rdent deployment needs.

## What's here

| File | Purpose |
|---|---|
| `valqore-servicetemplate.yaml` | k0rdent `ServiceTemplate` + OCI `HelmRepository` wrapping the signed `valqore-stack` chart |
| `catalog-entry.yaml` | Catalog metadata for a k0rdent Catalog submission |

## 1. Register the service (management cluster)

```bash
kubectl apply -f valqore-servicetemplate.yaml
```

This registers `valqore-stack` as a deployable k0rdent service, sourced from the
public, cosign-signed OCI chart at `oci://ghcr.io/valqore/charts`.

## 2. Deploy onto a managed cluster

Reference the service from a `ClusterDeployment`:

```yaml
apiVersion: k0rdent.mirantis.com/v1beta1
kind: ClusterDeployment
metadata:
  name: my-ai-cluster
  namespace: kcm-system
spec:
  template: <your-cluster-template>
  serviceSpec:
    services:
      - name: valqore
        namespace: valqore-system
        template: valqore-stack
        values: |
          webhook:
            enabled: true          # native ValidatingAdmissionPolicy enforcement
          policies:
            - name: enforce-owasp-agentic
              pack: owasp_agentic
              action: Warn         # flip to Deny once confident
```

Within ~30 seconds the API server enforces the compiled policies. **Valqore is
not in the data path** — no single point of failure if the engine is down.

## 3. Govern the AI factory

Once deployed, run the governance loop against your k0rdent-provisioned clusters
and clouds:

```bash
# Score the cluster's posture — one verdict across every lane
valqore scan            # live Kubernetes scan
valqore converge .      # security + cost + carbon + compliance + AI-gov, one verdict

# Who governs the agents running on the factory?
valqore fleet ./agents/         # shadow agents, delegation cycles, fleet score
valqore agent-audit ./agents/   # per-agent governance posture -> OSCAL

# Sovereign readiness for a gov / regulated k0rdent deployment
valqore sovereign .     # FedRAMP, EU AI Act, ISO 42001, CRA, PQC — assessed air-gapped

# Freeze re-runnable proof for an in-country auditor
valqore evaluate . --bundle evidence.json
valqore verify evidence.json --pubkey valqore.pub
```

## Submitting to the k0rdent Catalog

`catalog-entry.yaml` is the listing metadata. The catalog submission itself is a
PR to the [k0rdent Catalog repo](https://github.com/k0rdent/catalog) — confirm the
current schema and `ServiceTemplate` `apiVersion` against that repo before opening
it. (This step is an external submission to Mirantis.)
