# Example: the same verdict, with or without the AI

Companion file for the blog post. The MCP chat surfaces a verdict in plain
English; the same deterministic verdict comes straight from the CLI, no AI in
the loop.

```bash
docker run --rm -v "$PWD:/work" -w /work ghcr.io/valqore/engine:latest \
  valqore evaluate payments-taskdef.json --score
```

Expect BLOCK, with AWS-246 (privileged) and AWS-254 (plaintext secret) among the
findings — identical to what the agent reported over MCP.
