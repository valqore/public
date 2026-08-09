# Example: EU AI Act Article 50 transparency (AIG-179)

Companion files for the blog post. Fully offline, no cluster needed.

```bash
# Fails AIG-179 (no disclosure signal on a chatbot workload):
docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest \
  valqore evaluate chatbot-no-disclosure.yaml

# Passes AIG-179 (one annotation added):
docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest \
  valqore evaluate chatbot-disclosed.yaml
```

The only difference between the two files is the annotation
`valqore.io/ai-interaction-disclosure: "ui-banner"`.
