# Example: AI token-waste governance (valqore ai-waste)

Companion files for the blog post. Sample per-agent token telemetry + a model map.

```bash
# Audit:
docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest \
  valqore ai-waste telemetry.json --meta models.json

# CI gate — exits non-zero if >15% of spend is recoverable waste:
docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest \
  valqore ai-waste telemetry.json --meta models.json --max-recoverable-pct 15
```

Telemetry per agent: `{ input_tokens, output_tokens, calls }`.
