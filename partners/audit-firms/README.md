# Valqore for audit firms — verify evidence, don't trust a vendor

Most compliance tooling asks an auditor to trust the vendor's dashboard. Valqore
inverts that: every evidence bundle is a **pure function of captured state**, so
an audit firm re-executes it independently and gets **byte-identical results** —
no Valqore account, no secret, no network. You trust the math, not us.

This kit is for audit firms and consultancies who want to verify Valqore evidence
for their clients and, optionally, join the partner track.

## The core: independent verification

A Valqore evidence bundle freezes the exact inputs the rules saw, every rule
outcome, and the compliance-control statuses, under one SHA-256 content hash and
an Ed25519 signature. To verify a client's bundle, you need only the public image
and the client's public key:

```bash
# 1. Pull the signed public engine (or use the client's pinned version)
docker pull ghcr.io/valqore/engine:latest

# 2. Re-execute the bundle. No secret, no account, no network.
docker run --rm -v "$PWD:/w" -w /w ghcr.io/valqore/engine:latest \
  valqore verify evidence.json --pubkey client.pub
# -> REPRODUCED — outcomes re-ran identically; verdict …; control statuses reproduced.
#    Signature verified.
```

Three ways it fails loudly, so a forged bundle can't pass:

| Tampering | Result |
|---|---|
| Flip a recorded outcome | `MISMATCH` — replay recomputes it from state, exit 2 |
| Edit the captured state | content hash breaks, exit 2 |
| Flip a compliance-control status | named precisely (`soc2/CC6.1: recorded=FAIL → replayed=PASS`), exit 2 |

Byte-identical replay is guaranteed on the bundle's engine version; a different
version replays with a warning, never a false claim. See the runnable public demo
at [`examples/verify-evidence/`](../../examples/verify-evidence/).

## What you can attest

Because the verification is reproducible and the signature proves origin, an audit
firm can independently confirm, for the **technical and computed people/process
controls**, that:

- the evidence was produced by the stated party (signature), and
- the control statuses are a faithful function of the captured state (replay).

Valqore separates **VERIFIED** evidence (computed) from **ATTESTED** evidence
(signed, expiring human attestations for the un-computable long tail — board
minutes, training) and never blends them, so you always see which is which.

## The partner track

For firms who verify and distribute Valqore evidence at scale:

- **Independent verification** — your team re-runs client bundles with the
  public-key path; the trust is in the math, not a service relationship.
- **Branded policy packs** — publish your firm's control interpretations as a
  signed, portable Valqore pack your clients install with verified provenance
  (`valqore packs publish` / `install --pubkey`).
- **Handoff, not lock-in** — clients keep self-hosted, air-gapped Valqore; you
  verify the same bundles they run.

> **Getting involved is an external step.** The partner program itself
> (agreements, listing, co-marketing) is arranged directly with Valqore —
> email **tunc@valqore.io**. This kit is the technical foundation the program
> is built on; it works today with the shipped `valqore verify --pubkey`.
