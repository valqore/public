# Valqore PR Verdict — governance on every pull request

A deterministic governance verdict inside the PR, at the moment of change. The action scans
**only the IaC files the PR touched** (Terraform, Kubernetes YAML, Helm-rendered manifests,
Dockerfiles) against the Valqore engine's 1,400+ rules — security, cost, carbon, compliance,
and AI governance in one pass — and posts a sticky comment with:

- the **verdict** (PASS / PASS_WITH_MONITORING / BLOCK) and **Valqore Score** with the delta vs base,
- the **new findings this PR introduces** (not the repo's pre-existing noise),
- a **reproducible evidence bundle** attached to the run — anyone can re-execute the verdict
  with [`valqore verify`](https://valqore.io/verify), byte-for-byte.

Same input, same verdict, every run. No SaaS, no data leaves the runner: the engine is the
public `ghcr.io/valqore/engine` container running locally in the job.

## Usage

```yaml
# .github/workflows/valqore.yml
name: Valqore
on: pull_request

permissions:
  contents: read
  pull-requests: write   # the sticky comment

jobs:
  verdict:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: valqore/valqore/action@main
```

That's the whole setup. A PR that introduces a `privileged: true` pod or an unpinned MCP
server gets a 🔴 BLOCK comment naming the exact rules — and, by default, a failed check.

## Inputs

| Input | Default | What it does |
|---|---|---|
| `engine-version` | `1.13.7` | `ghcr.io/valqore/engine` tag to scan with |
| `fail-on` | `block` | `block` fails the check on a BLOCK verdict; `never` reports only |
| `comment` | `true` | Post/update the sticky PR comment |
| `evidence-bundle` | `true` | Emit + upload the reproducible evidence bundle artifact |
| `compare-base` | `true` | Also evaluate the base versions of the files and report deltas |
| `paths` | *(all)* | Space-separated path prefixes to limit which changed files count |
| `github-token` | `github.token` | Token used for the comment |

## Outputs

`verdict` (PASS / PASS_WITH_MONITORING / BLOCK), `score` (0–100), `new-findings` (count vs base).

## How "new findings" works

The action evaluates the changed files at the PR head **and** the same files at the base ref,
then diffs the FAIL/WARN sets. Your comment shows what *this PR* introduces (and fixes) — not
the backlog the repo already carries. Turn off with `compare-base: false`.

## The evidence bundle

With `evidence-bundle: true` (default), the scan emits a signed-format, content-hashed bundle
of exactly what the rules saw and every outcome, uploaded as the `valqore-evidence` artifact.
Re-run it anywhere:

```bash
docker run --rm -v "$PWD:/w" ghcr.io/valqore/engine:1.13.7 valqore verify /w/valqore-evidence.json
```

If it reproduces byte-for-byte, the PR verdict was true. That's the difference between a
scanner's assertion and evidence — see [the evidence guarantee](https://valqore.io/verify).
