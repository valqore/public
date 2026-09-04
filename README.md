# Valqore

> **One scan. One score. One verdict.**

[![Website](https://img.shields.io/badge/website-valqore.io-blue)](https://www.valqore.io)
[![Image](https://img.shields.io/badge/ghcr.io-valqore%2Fengine-2496ED?logo=docker)](https://github.com/orgs/valqore/packages/container/package/engine)
[![Rules](https://img.shields.io/badge/rules-1,428-brightgreen)]()
[![Compliance Packs](https://img.shields.io/badge/compliance%20packs-19-blueviolet)]()
[![Blog](https://img.shields.io/badge/blog-blog.valqore.io-black)](https://blog.valqore.io)

---

Valqore is an infrastructure governance engine that scans Kubernetes manifests, Terraform configurations, and cloud resources — then returns a **score (0-100)** and a **verdict** (PASS, PASS_WITH_MONITORING, or BLOCK).

**1,428 built-in rules** across security, cost, **carbon/sustainability (GreenOps)**, compliance, and AI governance, organised into **19 compliance packs** (including OWASP Top 10 for Agentic Applications 2026, EU AI Act Annex III, CRA, DORA, SOC2, HIPAA, FedRAMP, SR 11-7, and PQC Migration / CNSA 2.0). No configuration needed. Runs anywhere Docker runs.

---

## Quickstart (60 seconds)

**1. Pull the image** (free, public, no signup):

```bash
docker pull ghcr.io/valqore/engine:latest
```

**2. Scan a file:**

```bash
docker run --rm -v "$PWD:/work" -w /work \
  ghcr.io/valqore/engine:latest valqore evaluate deploy.yaml --score
```

**3. Make it short.** That `docker run …` prefix repeats a lot, so alias it once and every command below becomes just `valqore <cmd>`:

```bash
# bash / zsh (macOS, Linux)
alias valqore='docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest valqore'
```

```powershell
# PowerShell (Windows)
function valqore { docker run --rm -v "${PWD}:/work" -w /work ghcr.io/valqore/engine:latest valqore @args }
```

Now you can run everything against files in your current directory with clean, relative paths:

```bash
valqore evaluate deploy.yaml --score      # one file
valqore evaluate ./k8s/ --score           # a whole folder
valqore agent-audit ./k8s/                 # who governs your AI agents?
```

> Every example in this README uses the `valqore` alias. Without it, just put the
> `docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest` prefix back in front.

## Five ways to run Valqore

| Surface | Install | Best for |
|---|---|---|
| **CLI / Docker** | `docker run ghcr.io/valqore/engine:latest valqore evaluate manifest.yaml --score` | Local checks, CI pipelines |
| **K8s admission control** | [`helm install` the `valqore-stack` chart](#30-second-cluster-install-kubernetes-native) | Cluster-wide enforcement via native `ValidatingAdmissionPolicy` |
| **VS Code extension** | [`valqore-vscode`](https://docs.valqore.io) `.vsix` | Real-time CodeLens + hover + quick-fix in YAML / Terraform / Helm |
| **Freelens K8s IDE** | [`freelens-valqore`](https://docs.valqore.io) extension | Resource-detail panels + cluster overview + right-click policy checks |
| **MCP for Claude / Cursor** | `valqore mcp` | 137 governance tools your AI assistant can call |

## 30-second cluster install (Kubernetes-native)

```bash
# Install the operator stack from the public OCI chart (Go controller-runtime,
# ~30 MB image). The CRDs ship with the chart and install automatically.
helm install valqore oci://ghcr.io/valqore/charts/valqore-stack \
  --namespace valqore-system --create-namespace \
  --set 'policies[0].name=enforce-owasp-agentic' \
  --set 'policies[0].pack=owasp_agentic' \
  --set 'policies[0].action=Warn'
```

Within 30 seconds, **8 native `ValidatingAdmissionPolicy` objects** materialise on the cluster — the K8s API server enforces them. **Valqore is not in the data path.**

```bash
kubectl get valqorepolicy enforce-owasp-agentic
# NAME                    PACK            ACTION   READYVAPS
# enforce-owasp-agentic   owasp_agentic   Warn     8
```

Flip `action: Warn` → `action: Deny` when you're confident, then watch the API server reject unannotated agent workloads at admission time.

**Key differentiators:**
- **GreenOps built-in** — CO2e emissions per workload, greener region suggestions, carbon budgets, GPU emissions tracking. 77 cloud regions with grid carbon intensity data.
- **AI Scan** — one command runs evaluate + drift detection + AI-powered explanation. The AI image includes a fine-tuned model that runs fully offline — your code never leaves the container.
- **Interactive chat** — ask Valqore questions about your scan results in natural language. Get remediation advice, compliance mapping, and cost optimization tips through a conversational interface.
- **AI Governance** — detect ungoverned AI/ML workloads, enforce EU AI Act compliance, and gate model promotions to production.
- **AI agent fleet governance** — `agent-audit` discovers the AI agents already running in your manifests, cluster, or cloud and scores each one's governance posture across five dimensions, then rolls up to a GOVERNED / PARTIAL / UNGOVERNED fleet verdict and exports it as auditor-ready OSCAL. The answer to "who governs the agents now governing your infra?"

---

## Images & licensing

The only requirement is Docker (Linux, macOS, or Windows). **No key, no signup** for the
deterministic core — it's free and runs tokenless. The public image is **compiled** (native
code, no readable source), **multi-arch** (linux/amd64 + linux/arm64), and **cosign-signed
with an SBOM**.

| Image | Distribution | What's included |
|-------|------|-------------|
| `ghcr.io/valqore/engine:latest` | **Free, public, tokenless** | All 1,428 rules, scoring, drift, billing, compliance, MCP, agent-gate |
| `ghcr.io/valqore/engine-ai:1.13.6` | **Licensed** ([request access](mailto:tunc@valqore.io)) | Everything above + embedded offline AI model (AI scan, chat) |

Only the AI features need a license. To activate the AI image, create a persistent volume once,
then activate:

```bash
docker volume create valqore-data
docker run --rm -v valqore-data:/app/data ghcr.io/valqore/engine-ai:1.13.6 valqore activate YOUR_LICENSE_KEY
```

---

## Verify image authenticity (supply chain)

Every published image is cryptographically signed and carries an SPDX SBOM, so you can prove exactly what you're running. Install [cosign](https://docs.sigstore.dev/cosign/system_config/installation/), then:

**Public image** (`ghcr.io/valqore/engine:1.18.0`) — keyless-signed in CI via Sigstore (GitHub OIDC + Rekor):

```bash
cosign verify ghcr.io/valqore/engine:1.18.0 \
  --certificate-identity-regexp 'https://github.com/valqore/valqore-engine/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
cosign verify-attestation ghcr.io/valqore/engine:1.18.0 --type spdxjson \
  --certificate-identity-regexp 'https://github.com/valqore/valqore-engine/.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

**AI image** (`ghcr.io/valqore/engine-ai:1.13.6`) — signed with Valqore's release key ([`cosign.pub`](cosign.pub)):

```bash
cosign verify --key cosign.pub --insecure-ignore-tlog ghcr.io/valqore/engine-ai:1.13.6
```

Both checks confirm the image hasn't been tampered with since publish. The SBOM (SPDX) enumerates every component in the image for vulnerability scanning and audit.

---

## Security & trust

- **[TRUST.md](TRUST.md)** — how the deterministic engine decides, what Valqore does (and doesn't) do with your data, self-hosted/no-egress posture, supply-chain integrity, and an honest in-place-vs-planned status.
- **[SECURITY.md](SECURITY.md)** — coordinated vulnerability disclosure policy, private reporting channels, response targets, and safe harbor.
- **[SECURITY-FAQ.md](SECURITY-FAQ.md)** — vendor security & due-diligence FAQ: SIG / CAIQ-style answers on data handling, subprocessors, encryption, supply chain, AI, and compliance posture, for security/procurement reviews.

---

## What You Can Do

> These use the `valqore` alias from [Quickstart](#quickstart-60-seconds). Commands that read
> live cloud accounts also need credentials passed to the container — those are shown in full
> `docker run` form with the `-e` flags, since an alias can't carry your environment.

### Scan manifests, Terraform, or a whole directory

```bash
valqore evaluate deploy.yaml --score        # a single Kubernetes manifest
valqore evaluate main.tf --score            # a Terraform file
valqore evaluate ./ --score                 # everything in the current folder
```

```
Valqore Score: 84/100 (Grade: B)
  Security: 78 | Reliability: 77 | Cost: 90 | Carbon: 97 | Compliance: 100
Cost estimate: $19.53/mo (aws, us-east-1)
Verdict: BLOCK   ·   Total: 495 | Pass: 383 | Warn: 84 | Fail: 28
```

### Simulate a change before you make it

```bash
valqore what-if deploy.yaml --graviton       # migrate to ARM/Graviton
valqore what-if deploy.yaml --spot-ratio 70  # move 70% to spot instances
```

```
=== What-If: Migrate to Graviton (ARM) ===
  Cost:    $19/mo -> $15/mo (-20%)
  Carbon:  0.29 kg -> 0.09 kg (-69.5%)
```

### Govern your AI agents — the flagship

> **Who governs the agents now governing your infra?** `agent-audit` discovers every AI agent
> in your manifests/cluster/cloud and scores its posture across five dimensions; `agent-gate`
> stops an agent's *proposed* change before it touches anything.

```bash
valqore agent-audit ./                        # score the whole agent fleet
```

```
=== AI Agent Governance Posture ===
  Fleet: 2 agent(s) -- PARTIAL -- 1 governed / 1 with gaps

  Agent             Score  Identity  Guardrails  Boundary  Oversight  Supply Chain
  ops-agent          95     ok        ok          ok        ok         ok
  research-agent     46     gap       gap          --        gap        ok
```

Add `--format oscal -o agentgov.json` for an auditor-ready evidence pack where each dimension maps to a control.

```bash
# Gate a Terraform plan an agent wants to apply — block anything above a blast-radius cap
valqore agent-gate run --tf-plan plan.json --agent sre-bot --max-blast-radius medium
```

```
Agent action: BLOCK (BLOCKED)
sre-bot: infrastructure change
source: tf_plan · change: replace · blast radius: high
Blast radius 'high' exceeds max 'medium'.
```

Exit code `2` on BLOCK. Add `--request-approval` to mint a signed approval request recorded in the agent-gate journal.

### Gate ungoverned AI/ML workloads to production

```bash
valqore ai-gate ./
```

```
=== AI Promotion Gates: ml-inference -> production ===
  AI Registered: FAIL  ·  Human Oversight: FAIL  ·  EU AI Act: FAIL  ·  Kill Switch: FAIL
  Result: BLOCKED -- 4 of 5 gates failing
```

### Shift-left cost gate (cost prevention in the PR)

```bash
valqore finops cost-gate ./proposed/ --baseline ./current/ --max-delta 500
```

Exit code 1 when the change adds more than $500/mo over the baseline — cost *prevention* in the PR, not a dashboard after the bill lands.

### Audit container images

```bash
valqore image-audit ./ --check-updates
```

```
  redis     latest  ->  --       UNPINNED   HIGH
  nginx     1.21    ->  1.27.0   OUTDATED   HIGH
```

### Export compliance evidence

```bash
valqore evidence hipaa -f ./           # add -f oscal for machine-readable NIST OSCAL
```

All 19 packs: `hipaa`, `soc2`, `pci_dss`, `gdpr`, `iso27001`, `iso_42001`, `eu_ai_act`, `nist_csf`, `nist_ai_rmf`, `owasp_llm`, `owasp_agentic`, `owasp_mcp`, `dora`, `fedramp`, `sr_11_7`, `cra`, `pqc_migration`, `fda_samd`, `ai_finops`.

### GreenOps — carbon tracking (built in)

Every `evaluate --score` already includes a carbon estimate:

```
Carbon: 0.182 kg CO2e/mo (aws:us-east-1)
```

Valqore tracks CO2e per workload using grid carbon-intensity data across **77 cloud regions**, suggests greener regions, enforces carbon budgets, and tracks GPU embodied emissions.

---

### Commands that read your cloud account

These need read-only credentials, so they're shown in full `docker run` form (Valqore **never** writes to your cloud):

**Cloud billing & budget gate (AWS):**

```bash
docker run --rm \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  -e AWS_DEFAULT_REGION=us-east-1 \
  ghcr.io/valqore/engine:latest valqore finops billing --cloud aws --daily --days 30
```

```
AWS Daily Cost Trend (last 30 days)  Total: $3,459.14
      $137 |                 #
      $120 |####             ##########
      $103 |#####           #############
       $86 |#############################
           +------------------------------
            03-17                    04-15
```

Swap the final flag for `--fail-if-over 5000` to exit non-zero (CI gate) when monthly spend crosses $5,000. For Azure, pass `-e AZURE_CLIENT_ID / AZURE_TENANT_ID / AZURE_CLIENT_SECRET` and use `--cloud azure --subscription-id <id>`.

**Detect infrastructure drift (Terraform state vs live cloud):**

```bash
docker run --rm -v "$PWD:/work" -w /work \
  -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  -e AWS_DEFAULT_REGION=us-east-1 \
  ghcr.io/valqore/engine:latest valqore drift-state terraform.tfstate --cloud aws --attribution
```

Shows what changed, when, and who changed it (via CloudTrail). Add `--watch --interval 30 --slack <webhook>` for continuous monitoring with Slack alerts.

---

### AI image — offline scan & chat (licensed)

The `ghcr.io/valqore/engine-ai:1.13.6` image bundles a fine-tuned model that runs **fully offline** — your code never leaves the container. [Request a license.](mailto:tunc@valqore.io)

**AI scan — evaluate + drift + plain-English explanation in one shot:**

```bash
docker run --rm -v "$PWD:/work" -w /work -v valqore-data:/app/data \
  ghcr.io/valqore/engine-ai:1.13.6 valqore ai-scan ./ --state terraform.tfstate --cloud aws
```

```
Valqore AI Scan
Step 1/3: Evaluating manifests...   Score: 84/100 (B) | Verdict: BLOCK
Step 2/3: Detecting drift...        3 resources drifted
Step 3/3: Generating AI analysis...

  ## Key findings:
  - SP-007: Container running in privileged mode -- full host access
  - NET-012: LoadBalancer without NetworkPolicy on backend
  ## Remediation:
  1. Remove privileged: true and add capabilities.drop: ['ALL']
  2. Create NetworkPolicy restricting ingress to port 8080
```

**Chat — ask questions about your infrastructure:**

```bash
docker run --rm -it -v "$PWD:/work" -w /work -v valqore-data:/app/data \
  ghcr.io/valqore/engine-ai:1.13.6 valqore chat deploy.yaml
```

```
You: What are the most critical issues?
Valqore: You have 2 CRITICAL findings that must be fixed immediately:
  1. SP-007: Container 'api' is running in privileged mode...
  2. NET-012: LoadBalancer without NetworkPolicy...

You: Is this HIPAA compliant?
Valqore: Your current configuration fails 1 of 7 HIPAA controls...
```

---

## Cloud Provider Setup

Valqore is **read-only**. It never creates, modifies, or deletes resources.

### AWS

Pass credentials as environment variables:

```bash
-e AWS_ACCESS_KEY_ID=AKIA...
-e AWS_SECRET_ACCESS_KEY=...
-e AWS_DEFAULT_REGION=us-east-1
```

Minimum permissions: `ReadOnlyAccess` or custom policy with `ec2:Describe*`, `s3:GetBucket*`, `rds:Describe*`, `iam:List*`, `ce:GetCostAndUsage`, `cloudtrail:LookupEvents`.

### Azure

Pass service principal credentials:

```bash
-e AZURE_CLIENT_ID=your-app-id
-e AZURE_TENANT_ID=your-tenant-id
-e AZURE_CLIENT_SECRET=your-secret
```

Minimum roles: `Reader` + `Cost Management Reader`.

### Kubernetes

Mount your kubeconfig:

```bash
-v ~/.kube/config:/app/data/.kube/config:ro
```

---

## CI/CD Integration

### GitHub Actions

```yaml
- name: Valqore Gate
  run: |
    docker run --rm -v ${{ github.workspace }}:/work -w /work \
      ghcr.io/valqore/engine:latest valqore evaluate ./ --score --fail-on block
```

Exit code 1 on BLOCK verdict = PR fails.

### Cost gate in pipeline

```yaml
- name: Cost Gate
  run: |
    docker run --rm \
      -e AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }} \
      -e AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }} \
      ghcr.io/valqore/engine:latest valqore finops billing --cloud aws --fail-if-over 5000
```

---

## Try It — Example Scenarios

Clone this repo and scan any folder. Each scenario is realistic infrastructure with intentional misconfigurations — copy a command, see a verdict.

```bash
git clone https://github.com/valqore/valqore.git
cd valqore

# (set up the `valqore` alias from Quickstart first, then:)
valqore evaluate examples/ecommerce/ --score
```

### AI agents — governed vs ungoverned (the flagship)

The same agent workload, done two ways. Run `agent-audit` on each and watch the fleet verdict flip:

| File | Verdict | Why |
|------|---------|-----|
| [agent-gate/ungoverned-agent.yaml](examples/agent-gate/ungoverned-agent.yaml) | **UNGOVERNED** (45.9) | Shared/default service account, privileged + root, no kill-switch, no rate limit, no blast-radius cap, no audit sink |
| [agent-gate/governed-agent.yaml](examples/agent-gate/governed-agent.yaml) | **GOVERNED** (94.6) | Dedicated identity, OIDC + signed delegation, step/token/timeout caps, telemetry, escalation, default-deny egress, non-root, pinned by digest |

```bash
valqore agent-audit examples/agent-gate/ungoverned-agent.yaml   # -> UNGOVERNED
valqore agent-audit examples/agent-gate/governed-agent.yaml     # -> GOVERNED
valqore agent-audit examples/agent-gate/                        # -> PARTIAL (1 of 2 governed)
```

And stop a risky change an agent proposes, before it lands:

```bash
# A Terraform plan that deletes a prod DB and opens 0.0.0.0/0 -> blast radius HIGH -> BLOCK (exit 2)
valqore agent-gate run --tf-plan examples/agent-gate/terraform-plan.json \
  --agent sre-bot --max-blast-radius medium
```

#### Govern a whole agentic-framework fleet (kagent / BYO)

Two runnable, end-to-end proofs that drive the real CLI:

- **[kagent_governance/](examples/kagent_governance/)** — take a [kagent](https://kagent.dev) fleet from UNGOVERNED to GOVERNED in five steps: native Agent-CRD discovery, A2A delegation-cycle + shadow-agent detection, telemetry-grounded runtime guardrails, per-agent MCP gating, and VAP admission. `python examples/kagent_governance/demo.py`
- **[versus_incident_gate/](examples/versus_incident_gate/)** — gate a runbook-reading incident agent: the runbook says "roll back prod," the gate holds it for a signed human. `python examples/versus_incident_gate/demo.py`

### Supply chain — unpinned & mutable images

```bash
valqore evaluate examples/supply-chain/unsigned-unpinned.yaml --score
```

[supply-chain/unsigned-unpinned.yaml](examples/supply-chain/unsigned-unpinned.yaml) ships three classic risks — `nginx:latest` (mutable tag), `redis` (no tag), `busybox:1.36` (tagged but not digest-pinned) — the way a "known-good" image silently becomes a different, unverified one at deploy time.

### Basics — Secure vs Insecure

| File | Score\* | What Valqore catches |
|------|--------|---------------------|
| [basics/insecure-deploy.yaml](examples/basics/insecure-deploy.yaml) | 28 / F · BLOCK | Privileged container, unpinned image, no NetworkPolicy |
| [basics/secure-deploy.yaml](examples/basics/secure-deploy.yaml) | 76 / C · **PASS**† | Hardened + full governance annotations (CRA/DORA/rollback) — zero criticals |
| [basics/insecure-terraform.tf](examples/basics/insecure-terraform.tf) | 46 / F · BLOCK | Public RDS, open security group, unencrypted S3 |
| [basics/secure-terraform.tf](examples/basics/secure-terraform.tf) | 70 / C · BLOCK | Encrypted, private, Graviton, DynamoDB locking |
| [basics/ai-workload.yaml](examples/basics/ai-workload.yaml) | AI gate: BLOCK | Ungoverned GPU workload |
| [basics/gpu-ml-training.yaml](examples/basics/gpu-ml-training.yaml) | AI gate: PASS | Proper AI governance annotations |
| [basics/microservices-stack.yaml](examples/basics/microservices-stack.yaml) | 10 / F · BLOCK | Hardcoded secrets, no limits, privileged |

\*Scores reflect Valqore's **strict default policy** — most real-world workloads BLOCK until hardened. The point of each pair is the *relative* improvement.

†**PASS under a policy.** The default verdict is deliberately strict (it blocks on any finding). Real teams set a risk-appropriate bar in [`examples/.valqore/policy.yaml`](examples/.valqore/policy.yaml) — here: block on CRITICAL + require a passing score, treat HIGH as must-fix-soon. Under that policy the hardened workload **passes** and the insecure one still **blocks**:

```bash
# secure-deploy → PASS (76/100, zero criticals)
valqore env-evaluate examples/basics/secure-deploy.yaml \
  -e prod --policy examples/.valqore/policy.yaml

# insecure-deploy → BLOCK (28/100, 3 criticals)
valqore env-evaluate examples/basics/insecure-deploy.yaml \
  -e prod --policy examples/.valqore/policy.yaml
```

### Real-World Scenarios

| Scenario | Path | What's Inside |
|----------|------|--------------|
| E-Commerce | [examples/ecommerce/](examples/ecommerce/) | Storefront + payment + DB. Hardcoded Stripe keys, privileged payment service, public RDS. |
| SaaS Platform | [examples/saas-platform/](examples/saas-platform/) | API gateway + auth + workers. JWT secrets in plain text, no tenant isolation. |
| Data Pipeline | [examples/data-pipeline/](examples/data-pipeline/) | Kafka + Spark + Elasticsearch. AWS creds hardcoded, Spark as root, data on emptyDir. |
| Startup MVP | [examples/startup-mvp/](examples/startup-mvp/) | Node.js + MongoDB in one pod. DB password in env, default namespace, NodePort. |
| Fintech Trading | [examples/fintech/](examples/fintech/) | Matching engine + market feed + risk calc. Privileged engine, single-AZ DB, no encryption. |
| Azure Web App | [examples/azure/](examples/azure/) | App Service + SQL + Storage. TLS 1.0, public SQL, secrets in app settings. |
| GCP GKE | [examples/gcp/](examples/gcp/) | GKE + Cloud SQL + Firewall. Legacy ABAC, public DB, allow-all firewall. |

### GreenOps Scenarios

Compare carbon impact — same workload, different configurations:

| File | What It Shows |
|------|--------------|
| [greenops/high-carbon-deployment.yaml](examples/greenops/high-carbon-deployment.yaml) | 10 GPU replicas + 20 batch workers in us-east-1. High carbon footprint, oversized resources. |
| [greenops/low-carbon-deployment.yaml](examples/greenops/low-carbon-deployment.yaml) | Right-sized, eu-north-1 (hydro/nuclear), carbon budget set, fewer replicas. |

```bash
# Compare the two
valqore evaluate examples/greenops/high-carbon-deployment.yaml --score
valqore evaluate examples/greenops/low-carbon-deployment.yaml --score
```

### AI Scan Scenarios

Full-stack app with K8s + Terraform — run `ai-scan` to get evaluate + drift + AI explanation in one shot:

| File | What It Shows |
|------|--------------|
| [ai-scan/multi-tier-app.yaml](examples/ai-scan/multi-tier-app.yaml) | Frontend + API + workers + DB + Redis. Hardcoded secrets, root user, AWS keys in env. |
| [ai-scan/infra.tf](examples/ai-scan/infra.tf) | EKS + RDS + S3 + IAM. Public EKS, unencrypted DB, admin IAM policy, open security group. |

```bash
# AI Scan — evaluates everything and explains findings (AI image)
docker run --rm -v "$PWD:/work" -w /work -v valqore-data:/app/data \
  ghcr.io/valqore/engine-ai:1.13.6 valqore ai-scan examples/ai-scan/
```

### Chat Scenarios

Scan these files, then start a chat to ask questions — great for compliance-heavy environments:

| File | What It Shows |
|------|--------------|
| [chat/healthcare-api.yaml](examples/chat/healthcare-api.yaml) | Patient API + EHR integration + audit logger. HIPAA-relevant: secrets in env, privileged EHR connector, audit logs on emptyDir. |
| [chat/finserv-platform.yaml](examples/chat/finserv-platform.yaml) | Transaction processor + fraud ML + compliance reporter. PCI-DSS relevant: card encryption keys in env, privileged GPU fraud model. |

```bash
# Scan first, then chat about findings (AI image)
docker run --rm -it -v "$PWD:/work" -w /work -v valqore-data:/app/data \
  ghcr.io/valqore/engine-ai:1.13.6 valqore chat examples/chat/healthcare-api.yaml

# Try asking:
#   "Is this HIPAA compliant?"
#   "What are the biggest risks?"
#   "How do I fix the secrets?"
#   "Generate a compliance report"
```

---

## What Valqore Covers

| Category | What It Does |
|----------|-------------|
| **Security** | Container hardening, RBAC, encryption, network policies, supply chain, attack path analysis |
| **Cost & FinOps** | Waste detection, right-sizing, billing analysis (AWS/Azure/GCP), budget gates, cost simulation |
| **GreenOps** | CO2e per workload, greener region suggestions, carbon budgets, GPU emissions, 77 regions with grid intensity data |
| **Compliance** | 19 packs: HIPAA, SOC 2, PCI-DSS, GDPR, EU AI Act, ISO 42001, NIST AI RMF, OWASP LLM/Agentic/MCP, DORA, FedRAMP, SR 11-7, CRA, PQC, FDA SaMD, AI FinOps, and more — each exportable as machine-readable **NIST OSCAL** evidence |
| **AI Governance** | Shadow AI detection, EU AI Act risk classification, model promotion gates, GPU cost/carbon tracking |
| **AI Agent Fleet Governance** | `agent-audit` discovers AI agents across manifests/cluster/cloud and scores each one's posture (identity, guardrails, boundary, oversight, model supply chain) into a GOVERNED / PARTIAL / UNGOVERNED fleet verdict — exportable as OSCAL. Plus a runtime MCP gate for live agent tool calls |
| **Drift Detection** | Terraform state vs live cloud, CloudTrail attribution, continuous monitoring with Slack alerts |
| **AI Scan & Chat** | One-command scan with AI explanation, interactive chat for remediation advice — fully offline |
| **Multi-Cloud** | AWS, Azure, GCP, and any Kubernetes cluster. Read-only — never modifies your infrastructure |
| **Dashboard** | Web-based dashboard for visualizing scores, trends, and team-wide governance — *coming soon* |

---

## Blog

Technical deep-dives and real-world examples:
- [blog/](blog/) — code examples from published posts
- [blog.valqore.io](https://blog.valqore.io) — full articles

---

## Links

- Website: [valqore.io](https://www.valqore.io)
- Container image: [`ghcr.io/valqore/engine`](https://github.com/orgs/valqore/packages/container/package/engine)
- Blog: [blog.valqore.io](https://blog.valqore.io)
- Book a demo: [calendly.com/tuncvalqore](https://calendly.com/tuncvalqore/30min)
- Contact: tunc@valqore.io

---

VALQORE is a pending trademark. All rights reserved. See [LICENSE](LICENSE).
