# Sealed-Loop AI Governance

**A Deterministic Control Plane for Agentic Infrastructure**

| | |
|---|---|
| **Author** | Tunc Celik, Founder, Valqore |
| **Version** | 1.2 |
| **Date** | 2026-09-04 (v1.0: 2026-05-07) |
| **Audience** | Enterprise CISOs, Chief AI Officers, Heads of Model Risk |

> **Revision note (v1.1/v1.2):** engine counts refreshed to the current release (1,428 rules / 19 categories, v1.18.0); §5.5/§5.6 updated — the three-guard EWMA anomaly model described as future work in v1.0 has shipped; EU AI Act enforcement date language updated (now in force); §5.6 gains a third independent convergence (sofka).

---

## Abstract

Probabilistic guardrails — LLM-as-judge models that classify other models' outputs as safe or unsafe — cannot satisfy the audit and reproducibility requirements of the EU AI Act, NIST AI RMF, ISO/IEC 42001, or US bank Model Risk Management (SR 11-7 / OCC Bulletin 2011-12). Their verdicts are non-deterministic, irreproducible between runs, and cannot be cryptographically chained to a sealed audit trail.

This paper introduces the Sealed-Loop architecture: a deterministic post-condition engine that produces verdicts on every infrastructure and agent path, with AI used only for human-readable explanation. The architecture composes well-established cryptographic primitives (Sigstore for attestation, in-toto for supply-chain integrity, OPA or Cedar for policy decisions, OSCAL for assessment evidence, OpenTelemetry for governance attributes) into a single control plane that gates AI workloads at deployment, blocks tool-poisoning attacks, and produces auditor-ready evidence at every verdict.

We document the threat model, the architecture, and the compliance-evidence mapping. We use Valqore as the reference implementation but the architecture is generalisable.

## 1. Introduction

The 18 months from late 2024 through mid-2026 produced more AI deployment incidents and more AI regulation than the prior decade combined. Production agents looped for days generating five-figure cloud bills. Tool-poisoning attacks became a documented OWASP category. The EU AI Act began phased enforcement (general enforcement August 2, 2026 — now in force — with fines up to EUR 35 million or 7% of global annual turnover). FedRAMP 20x added an AI Prioritization track in 2025 with the first authorizations expected to complete in early 2026. The Linux Foundation formed the Agentic AI Foundation (AAIF) on December 9, 2025, with eight platinum members including AWS, Anthropic, Google, Microsoft, and OpenAI.

Despite this regulatory momentum, the AI safety tooling market remains dominated by probabilistic guardrails — second-tier LLMs trained to classify the outputs of first-tier LLMs. This paper argues that probabilistic guardrails are architecturally incapable of satisfying the audit, reproducibility, and supply-chain integrity requirements that regulators have already named, and proposes the Sealed-Loop architecture as the alternative.

## 2. The Threat Model

The Sealed-Loop architecture defends against four threats. Each is documented in real-world incidents or in formal threat models published by recognised standards bodies.

### 2.1 Probabilistic LLM-judge inadequacy

LLM-as-judge guardrails (a second LLM evaluating the first LLM's output for safety) are stochastic by construction. For a given input, the judge model produces different verdicts across runs because of temperature sampling, model versioning, and provider-side caching invalidation. Three failure modes follow:

- **(a) Non-reproducibility.** An auditor cannot re-derive yesterday's verdict from today's evidence.
- **(b) Drift.** Provider-side model updates silently change verdict distributions, invalidating prior evidence.
- **(c) Model dependency.** The judge model itself is a regulated AI system in jurisdictions like the EU, requiring its own governance — an infinite-regress problem.

EU AI Act Article 12 requires "automatic recording of events" sufficient to "allow tracing back to the system's input" and to "identify situations that may result in a risk." Article 13 requires "transparent" technical documentation. Annex IV requires "description and characteristics of the system" that is "specific enough" for assessment. None of these requirements can be satisfied when the verdict-producing component is itself a probabilistic black box.

### 2.2 MCP tool poisoning (OWASP MCP Top 10)

The Model Context Protocol (MCP), now a Linux Foundation Agentic AI Foundation project, enables AI agents to discover and call tools exposed by external servers. The OWASP MCP Top 10 — the first OWASP project dedicated to MCP security — identifies tool poisoning as risk MCP03, with rug-pull attacks (silent post-approval tool modification) as a sub-technique.

The threat: an adversary publishes a benign-looking MCP tool that passes initial review, then silently modifies the tool's behavior or schema after the agent has authorized it. The agent then executes malicious operations under its own credentials, often with no visible logging of the schema mutation.

Probabilistic guardrails cannot detect this because the schema mutation is invisible to the LLM judge — the judge only sees the tool's responses, not its declared behavior at approval time vs. call time.

### 2.3 Transport-level disclosure (STDIO MCP)

MCP transports include STDIO (standard-input/standard-output local process pipes). Public security disclosures in 2025 documented that many STDIO MCP implementations leak credentials, file paths, and internal hostnames into process command lines, environment variables, or stderr — visible to any process with /proc access on the host.

Probabilistic guardrails operate on text content; they have no visibility into transport-level disclosure.

### 2.4 Regulatory inadmissibility of non-reproducible evidence

US bank Model Risk Management — SR 11-7 (Federal Reserve) and OCC Bulletin 2011-12 — explicitly requires that model verdicts be reproducible from documented inputs. The 2024 OCC Request for Information on model risk specifically asked whether AI/ML models satisfied this requirement.

EU AI Act Annex IV requires "the methodology used for the design and development of the AI system, the validation methods, and the testing data sets" — all of which presume deterministic, re-runnable artifacts. ISO/IEC 42001 Annex A controls require documented "system development life cycle" evidence that is, by implication, re-derivable.

A non-deterministic guardrail produces evidence that fails this audit — not because the guardrail is wrong, but because its outputs cannot be reproduced.

## 3. The Sealed-Loop Architecture

The Sealed-Loop architecture inverts the dominant pattern. Rather than letting a probabilistic model decide and using deterministic rules as a guard, deterministic rules decide and the AI explains.

### 3.1 Principles

- **P1. Determinism wins.** Every verdict is produced by a versioned, testable rule. AI never overrides a rule.
- **P2. AI explains, never decides.** The fine-tuned local AI model generates plain-language explanations and remediation suggestions. It cannot change a verdict.
- **P3. Local execution.** The engine runs entirely inside the customer's network. No infrastructure data leaves the host.
- **P4. Sealed evidence.** Every verdict is sealed in an append-only log with cryptographic chaining and signed attestations.
- **P5. Composable primitives.** Standard tools (Sigstore, in-toto, OPA, Cedar, OSCAL, OpenTelemetry) wherever possible.

### 3.2 Components

The architecture has five layers.

**Layer 1 — Input.** Sources: Kubernetes manifests, Terraform plans, Helm charts, Dockerfiles, live cloud APIs (AWS, Azure, GCP), live K8s clusters via read-only kubeconfig.

**Layer 2 — Deterministic rule engine.** 1,428 rules across 19 categories (as of engine v1.18.0). Each rule is:

- Versioned (rule_id + rule_version)
- Pure: same input always returns same output
- Testable: shipped with unit tests
- Cryptographically hashable: rule contents → SHA-256

Rules return one of {PASS, WARN, FAIL, INFO, ERROR} and a human-readable message.

**Layer 3 — Deterministic post-condition engine.** Per-agent and per-deployment post-conditions:

- Step limits (max iterations per session)
- Token budgets (max LLM tokens consumed)
- Tool-call allowlists (which MCP tools may fire)
- Timeout policies (wall-clock and idle)
- Egress NetworkPolicies (data exfiltration prevention)
- Promotion gates (AI Registered, Human Oversight, EU AI Act classification, Kill Switch, AIG Score ≥ 70)

Every post-condition is a versioned rule. Every violation blocks execution and writes a sealed entry.

**Layer 4 — Sealed audit log.** Append-only log with cryptographic chaining (each entry's hash feeds into the next). Each entry includes:

- Rule ID and version
- Input fingerprint (SHA-256 of normalised input)
- Verdict (PASS/WARN/FAIL/INFO/ERROR)
- Timestamp (UTC, nanosecond precision)
- Sigstore signature over the entry hash
- In-toto attestation linking the entry to its provenance (which artifact, which build pipeline, which commit)

Storage: SQLite with WAL mode locally, optionally replicated to S3 / Azure Blob / GCS with object-lock.

**Layer 5 — AI explainer (bounded).** Fine-tuned Qwen 2.5 0.5B model embedded in the engine. Receives a redacted version of the verdict and emits plain-language explanation + remediation suggestion. CANNOT modify the verdict. CANNOT access the raw input. CANNOT call external APIs without explicit configuration. Pre-redaction strips secrets, tokens, IPs, and PII before any prompt construction.

### 3.3 Architecture flow

```
    [INPUT: K8s YAML / Terraform / Live Cloud]
                     |
                     v
    [Layer 2: 1,428 Deterministic Rules]
                     |
                     v
    [Layer 3: Post-Condition Engine + Promotion Gates]
                     |
              +------+------+
              |             |
              v             v
    [Verdict: PASS|     [Layer 4: Sealed
      WARN|FAIL|         Audit Log
      INFO|ERROR]        (cryptographic chain
              |           + Sigstore + in-toto)]
              |
              v
    [Layer 5: AI Explainer (read-only)]
              |
              v
    [Output: human-readable explanation + remediation]
```

Compliance evidence pipeline:

```
    [Sealed Audit Log] -> [Control Mapper]
                              |
        +----------+----------+----------+
        |          |          |          |
        v          v          v          v
    [ISO 42001] [NIST AI  [EU AI Act] [OSCAL]
                 RMF]      Annex IV]
        |          |          |          |
        +----------+----------+----------+
                              |
                              v
            [HTML / OSCAL JSON / PDF auditor reports]
```

## 4. The Deterministic Post-Condition Engine

Section 3 introduced the post-condition engine; this section describes its semantics in detail because it is the most novel component of the architecture.

### 4.1 Definition

A post-condition is a predicate evaluated at the boundary between agent intention and effect. For an agent issuing a tool call:

- **pre-condition:** the policy bundle in effect at session start
- **intent:** the tool the agent intends to call, with arguments
- **post-condition:** the predicate that must hold for the call to fire

Examples of post-condition predicates:

```
tool_id IN allowlist[session.role]
session.token_count + estimated_tokens <= session.token_budget
session.step_count + 1 <= session.step_limit
tool.classification != "destructive" OR session.role >= "operator"
data.classification != "PII" OR target.jurisdiction = source.jurisdiction
elapsed_wall_clock_seconds < session.timeout_seconds
```

### 4.2 Enforcement points

The post-condition engine enforces at three points:

1. **Kubernetes admission webhook** — blocks Pod, Deployment, StatefulSet, Job, and CronJob creates that violate post-conditions on AI workloads (no kill switch, no risk class, no resource limits, etc.).
2. **Agent middleware** — Python decorator `@valqore.policy` that wraps tool-call functions in LangChain, LlamaIndex, and raw OpenAI/Anthropic SDK code. Decision latency budget: < 200 ms.
3. **CI/CD evaluator** — `valqore evaluate` runs in GitHub Actions, GitLab CI, Azure DevOps, and Bitbucket pipelines, blocking pull requests that fail post-conditions.

### 4.3 Latency budget

Sub-200ms decision latency is achievable because:

- Policy bundles are pre-compiled OPA WASM or Cedar AST
- Rule engine uses an in-process LRU cache keyed on the input fingerprint
- The sealed audit log writes asynchronously (the verdict is returned synchronously; durability is guaranteed by SQLite WAL)

## 5. Cryptographic Primitives in Use

The architecture composes four well-established primitives.

### 5.1 Sigstore (signed attestation)

Every rule bundle, AI-BOM, and audit log entry is signed via Sigstore (cosign + Rekor transparency log). Verifiers can prove that a specific rule version was in effect at a specific time without trusting the issuing party.

### 5.2 in-toto (supply-chain attestation)

Each artifact (rule bundle, model adapter, AI-BOM, evidence report) ships with an in-toto attestation linking it to its build provenance: source commit, builder, dependencies. This is consumable by federal supply-chain tooling (CISA SLSA L3+).

### 5.3 OPA / Cedar (policy decision)

Policy bundles are expressed in Open Policy Agent's Rego language or AWS Cedar (customer choice). Both languages have a deterministic evaluation model and pre-compilable bundles. The Sealed-Loop engine accepts either; both produce the same PASS/FAIL verdict for equivalent input.

### 5.4 OSCAL (compliance evidence)

Every compliance evaluation produces an OSCAL Assessment Results document conforming to NIST OSCAL 1.1.2. OSCAL is the format demanded by federal cloud authorization (FedRAMP 20x, Phase 2 Moderate Pilot). It is also the format Big-Four audit pipelines have begun to consume directly.

### 5.5 Local-learning stack

The Sealed-Loop architecture is sometimes mis-read as static — as if deterministic rules implied that the engine cannot adapt to a customer's environment. The opposite is true. Valqore ships a four-layer local-learning stack that adapts to each tenant's infrastructure without sending a single byte of data to any vendor cloud. Every layer persists locally to SQLite and operates in air-gapped environments by default.

**Layer 1 — Pattern catalog with operator feedback.** Recurring drift findings and rule failures are mined into stable "signatures." Pod hashes, AWS resource IDs (i-..., sg-..., vpc-...), Docker SHA fragments, and other environment-specific identifiers are normalised before hashing, so different instances of the same workload collapse to a single pattern. The operator labels each pattern as one of three states:

- **expected** — suppress this from alerts. It is normal in our env.
- **suspicious** — show in dashboard but do not page.
- **critical** — page on every occurrence.

The labelled patterns become a per-tenant rule set without any AI model retraining. After 30 days of operation, false-positive rate drops by an order of magnitude in mature deployments. Storage: `~/.valqore/patterns.db` (SQLite).

**Layer 2 — EWMA score baselines per workload.** Every (cluster, namespace, workload) tuple maintains its own score trajectory using Hunter's online EWMA mean and variance:

```
diff       = x - ewma_prev
incr       = alpha * diff
ewma_var   = (1 - alpha) * (ewma_var_prev + diff * incr)
ewma       = ewma_prev + incr
```

Anomaly detection uses three simultaneous guards, all of which must hold for an alert to fire: (1) deviation — `|x - ewma| > n_sigma * sqrt(ewma_var)` (default n_sigma = 2.0); (2) magnitude — `|x - ewma| >= min_delta` score points (default 5.0, rejecting small-magnitude noise that reads as many sigmas on a near-flat baseline); (3) warmup — a minimum number of prior samples (default 5) before detection activates. Default alpha = 0.3. All four parameters are tunable per deployment via environment variables.

This catches team-specific regressions that are invisible at the org-wide level. A single namespace can degrade for weeks while the org average looks fine; per-workload EWMA flags this within the first detectable deviation. Storage: `~/.valqore/score-baselines.db` (SQLite).

**Layer 3 — Local AI provider (Ollama).** When `VALQORE_AI_PROVIDER=ollama` is set, every AI explanation routes through the customer's local Ollama install. The engine talks to Ollama over the OpenAI-compatible API on localhost. No API keys, no external inference, no telemetry. The engine ships with model auto-resolution: if the configured model is missing, it falls back to a documented default and logs a warning, so deployments do not silently fail.

**Layer 4 — Embedded fine-tuned model.** The `-ai` container variant ships a fine-tuned Qwen 2.5 0.5B model embedded directly in the image. It is bundled, not downloaded — the customer never needs internet to run AI explanation, even on the first scan. Internal benchmark scores: 84% Grade A on the 0.5B variant; 86% on the 7B variant. Training completed in 11 minutes (0.5B) and 43 minutes (7B) on a single consumer GPU.

**Why this matters.** Three independent constraints make a local-learning stack a prerequisite for the customers Valqore is built for:

1. US federal procurement (FedRAMP 20x, IL5, classified networks) explicitly forbids vendor SaaS for infrastructure data. Air-gapped tooling is required by procurement language, not just preferred by buyers.
2. EU AI Act Article 12 requires "automatic recording of events" sufficient to "allow tracing back to the system's input." Forensic investigators must reach the same conclusion the engine did, from the same inputs, on the same hardware. That reproducibility cannot be guaranteed when the engine's learned state lives in a third-party cloud subject to provider-side caching invalidation, model versioning, and tier-based eviction.
3. Bank Model Risk Management (SR 11-7) requires that every model verdict be reproducible from documented inputs. A per-tenant locally-learned baseline satisfies this; a cloud ML system shared across tenants does not.

The local-learning stack is the architecture these three constraints demand. Valqore ships it today, in production.

### 5.6 Independent validation of the architecture

The Sealed-Loop pattern is not unique to Valqore. The Versus Incident project (github.com/VersusControl/versus-incident, MIT-licensed, v1.3.9 released May 2026) ships an AI SRE Agent that independently arrived at five of the same architectural primitives:

- **(a) Three rollout modes named identically** — Training, Shadow, Detect — with the same semantics: training is observation only; shadow logs would-have-alerted decisions but takes no action; detect creates real incidents.
- **(b) Pre-LLM secret redaction** as a mandatory first stage in the processing pipeline. Versus's documented pipeline is: "Read → Hide secrets → Filter → Group → Save → Decide."
- **(c) Pattern catalog with operator feedback.** Versus learns "templates" of normal log lines; Valqore learns operator-labelled signature patterns of normal infrastructure state.
- **(d) Multi-guard EWMA baselines for anomaly detection.** Versus uses three guards (spike_multiplier, spike_min_frequency, spike_min_baseline_count); Valqore ships the equivalent three-guard model in score-space (sigma deviation, absolute magnitude floor, warmup sample count) as of engine v1.18.x.
- **(e) Sealed audit log of decisions,** distinct from the AI inference output. In Versus, this is the shadow.json file; in Valqore, it is the cryptographically chained SQLite audit log.

A third independent convergence appeared in 2026: **sofka** (github.com/nklmilojevic/sofka, MIT/Apache-2.0), a Kubernetes TUI written in Rust by a third unrelated author, ships:

- **(f) a deterministic incident view** — its `X` "explain why" feature is documented as "a deterministic incident view: rollout state, degraded conditions, blocking pods, container failure reasons, recent Warning events. **No AI, no external service.**" Its roadmap goes further: health explanations "must be deterministic core features — AI may summarize or help explore collected evidence" but cannot be required.
- **(g) guardrails enforced, not remembered** — read-only mode and "never delete in prod" are "enforced per context, not remembered," alongside **an action journal of everything you did** — the same enforcement-over-convention and journaling primitives as P4 and Layer 4.

Three independent teams — operating in adjacent but non-overlapping domains (incident routing, cluster operations, and infrastructure governance) — converging on the same primitives is strong evidence that the Sealed-Loop architecture is the correct response to the threat model in Section 2, not a Valqore-specific design choice.

## 6. Benchmark Methodology

Sealed-Loop is not a head-to-head replacement for probabilistic guardrails — they target different layers (probabilistic guardrails operate on output content; Sealed-Loop operates on infrastructure and agent paths). However, the architectures can be compared on audit-relevant properties.

### 6.1 Properties under test

For a given test corpus of 100 violation scenarios:

- **Verdict reproducibility:** P(verdict_t1 == verdict_t2 | same input)
- **Audit completeness:** fraction of verdicts with sealed log entries
- **Forensic depth:** median bytes of structured evidence per verdict
- **Adversarial resistance:** fraction of injected violations blocked despite obfuscation (whitespace, base64, foreign-language encoding)

### 6.2 Public benchmark commitment

A public benchmark harness comparing Valqore against representative tools on a fixed, versioned corpus is published at [valqore.io/compare](https://valqore.io/compare); the corpus and runner are open source and results are reproducible by third parties. This paper deliberately avoids citing third-party benchmark numbers. Where third-party numbers are available, we will cite them with verifiable arXiv IDs or DOIs in subsequent versions.

## 7. Compliance-Evidence Mapping

Sealed-Loop produces evidence directly mappable to the four frameworks regulators are most likely to audit.

### 7.1 ISO/IEC 42001 (AI management systems)

- Annex A.5 (AI System Lifecycle): A.5.1, A.5.2, A.5.3 — covered by deterministic rule engine + sealed audit log + AI-BOM signing.
- Annex A.6 (Data Management): A.6.1 — covered by PII redaction + jurisdiction-bound policies.
- Annex A.9 (System Implementation): A.9.1, A.9.2 — covered by Kubernetes admission webhook + CI evaluator.

### 7.2 NIST AI RMF 1.0

- **GOVERN:** covered by signed policy bundle distribution and the sealed audit log.
- **MAP:** covered by AI workload discovery rules (AIG-001..015 in the engine).
- **MEASURE:** covered by EWMA per-workload baselines and threshold-based monitoring.
- **MANAGE:** covered by promotion gates and post-condition engine.

### 7.3 EU AI Act

- Article 9 (Risk Management System): per-workload risk classification and continuous evaluation.
- Article 10 (Data Governance): data-classification rules + jurisdiction-bound policies.
- Article 12 (Record-Keeping): sealed audit log with cryptographic chaining.
- Article 13 (Transparency): deterministic verdict + AI-explainer remediation guidance.
- Article 14 (Human Oversight): promotion gate "Human Oversight" requirement.
- Article 15 (Accuracy and Robustness): versioned rules + reproducible verdicts.
- Annex IV (Technical Documentation): OSCAL Assessment Results export.

### 7.4 SR 11-7 / OCC Bulletin 2011-12 (US bank model risk)

- Pillar 1 (Development): AI-BOM, rule mapping, development controls.
- Pillar 2 (Validation): independent evaluation rules and explainability annotations.
- Pillar 3 (Governance): promotion gates and inference auth requirements.
- Pillar 4 (Inventory): AI workload discovery.
- Pillar 5 (Monitoring): audit logging + kill switch + agent loop limit.
- Pillar 6 (Effective Challenge): human-oversight gate.
- Pillar 7 (Documentation): AI-BOM + explainability + OSCAL export.
- Pillar 8 (Vendor Models): supply-chain rules and Sigstore attestation.
- 2024 RFI Extension (Agentic AI): step / token / tool-call limits.
- 2024 RFI Extension (Reproducibility): the deterministic rule architecture itself — the entire raison d'être.

## 8. Limitations and Open Problems

The Sealed-Loop architecture is not a complete AI safety solution. Three boundaries remain.

### 8.1 Output-content safety

Probabilistic guardrails are still useful — and probably necessary — at the model output layer (jailbreak detection, prompt-injection mitigation, hallucination scoring). Sealed-Loop is complementary, not substitutive.

### 8.2 Model-internal safety

Bias, fairness, and adversarial robustness in the model itself are out of scope. Sealed-Loop enforces governance at deployment and runtime; it does not inspect model weights.

### 8.3 Social-layer failures

A correctly-governed AI system can still produce socially harmful outcomes (e.g., a credit-scoring model that complies with every SR 11-7 control but produces disparate outcomes across protected classes). Governance evidence is necessary but not sufficient for ethical deployment.

## 9. Conclusion

The next 12 months will determine whether AI governance is a checkbox category or a true infrastructure discipline. Probabilistic guardrails will continue to play a role at the output layer, but they cannot satisfy the audit, reproducibility, and supply-chain integrity requirements that EU AI Act, NIST AI RMF, ISO/IEC 42001, and SR 11-7 each require.

The Sealed-Loop architecture — deterministic rules deciding, AI explaining, sealed evidence chaining — is the only architecture we are aware of that simultaneously satisfies these requirements and ships today. Valqore is the reference implementation. The architecture itself is generalisable and can be implemented with any combination of the cryptographic primitives discussed in Section 5.

Enterprise CISOs and Chief AI Officers exposed to the EU AI Act (now in force), federal AI procurement under FedRAMP 20x, or US bank Model Risk Management under SR 11-7 should evaluate Sealed-Loop control planes alongside, not instead of, their existing probabilistic safety stack.

## Appendix A — Definitions

| Term | Definition |
|---|---|
| **Sealed-Loop** | Architecture in which deterministic rules produce verdicts and AI produces explanations. Sealed because every verdict is recorded in a tamper-evident append-only log. |
| **Post-Condition** | A predicate evaluated at the boundary between agent intention and effect. |
| **Promotion Gate** | A boolean predicate that must hold for an AI workload to advance from staging to production. Five gates: AI Registered, Human Oversight, EU AI Act Classification, Kill Switch, AIG Score ≥ 70. |
| **AI-BOM** | AI Bill of Materials — manifest documenting a model's training data sources, base model hashes, adapter lineage, and inference-time configuration. Standards: SPDX-AI, CycloneDX-ML. |
| **OSCAL** | Open Security Controls Assessment Language, NIST standard for compliance evidence. Version 1.1.2 supported by the Sealed-Loop reference implementation. |

## Appendix B — References

1. EU AI Act, Regulation (EU) 2024/1689, in force 2024-08-01, general enforcement 2026-08-02. https://eur-lex.europa.eu/eli/reg/2024/1689/oj
2. NIST AI Risk Management Framework (AI RMF 1.0), January 2023. https://www.nist.gov/itl/ai-risk-management-framework
3. ISO/IEC 42001:2023 — Information technology — Artificial intelligence — Management system. December 2023.
4. Federal Reserve SR 11-7, "Supervisory Guidance on Model Risk Management," April 4, 2011. https://www.federalreserve.gov/supervisionreg/srletters/sr1107.htm
5. OCC Bulletin 2011-12, "Sound Practices for Model Risk Management," April 4, 2011.
6. OCC RFI on Model Risk Management, comments closed September 2024.
7. OWASP MCP Top 10. https://owasp.org/www-project-mcp-top-10/
8. FedRAMP 20x AI Prioritization. https://www.fedramp.gov/ai/
9. FedRAMP 20x Phase 2 Moderate Pilot, January 13, 2026.
10. Linux Foundation Agentic AI Foundation (AAIF), formed December 9, 2025. https://aaif.io/
11. Sigstore project. https://www.sigstore.dev/
12. in-toto framework. https://in-toto.io/
13. Open Policy Agent (OPA). https://www.openpolicyagent.org/
14. AWS Cedar policy language. https://www.cedarpolicy.com/
15. OSCAL 1.1.2 specification. https://pages.nist.gov/OSCAL/concepts/layer/assessment/assessment-results/
16. Versus Incident AI SRE Agent (independent prior art on training/shadow/detect modes, pre-LLM redaction, EWMA spike detection, sealed audit log). MIT-licensed, v1.3.9 May 2026. https://github.com/VersusControl/versus-incident
17. sofka Kubernetes TUI (independent prior art on deterministic no-AI incident views, enforced guardrails, action journal). MIT/Apache-2.0, 2026. https://github.com/nklmilojevic/sofka · https://sofka.rs
