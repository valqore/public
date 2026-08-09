# Example: EU Cyber Resilience Act (CRA) readiness

Companion files for the blog post. Runs offline against the built-in `cra` pack.

```bash
# 0/10 controls — a product with no CRA declarations:
docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest \
  valqore evaluate product-no-cra.yaml --compliance-pack cra

# 7/10 — the same product with the CRA facts declared as annotations:
docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest \
  valqore evaluate product-cra-ready.yaml --compliance-pack cra
```

The `cra` pack maps 9 controls (CMP-021..026, 032..034) to CRA articles:
security contact, CVD policy, manufacturer ID, SBOM, support window, conformity,
secure-by-default, update mechanism, and vulnerability handling.
