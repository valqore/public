# Valqore in your Kubernetes TUI

One keypress from the console you already live in to a Valqore answer:

- **Shift-W** on a workload → `valqore why` — a **deterministic incident
  view**: rollout state, per-container failure reasons (CrashLoopBackOff,
  OOMKilled, image pulls, stuck scheduling), recent Warning events, and the
  **governance findings on the same workload that relate to the symptom** —
  the part only Valqore adds. No AI, no external service; same inputs,
  same answer. Read-only.
- **Shift-V** on any resource → `valqore evaluate` — score, verdict, and
  findings for the live manifest.

Works with:

- **k9s** — merge [`k9s-plugins.yaml`](k9s-plugins.yaml) into
  `$XDG_CONFIG_HOME/k9s/plugins.yaml`.
- **sofka** — merge [`sofka-plugins.toml`](sofka-plugins.toml) into your
  sofka config's `[[plugins]]` list.

Requirements: `valqore` and `kubectl` on PATH, and a kubeconfig for the
cluster the TUI is pointed at. Both plugins are read-only — they never
mutate the cluster.

Also usable outside any TUI:

```bash
valqore why checkout-api -n ecommerce          # why is it broken?
valqore why inference-server -n ai --json      # machine-readable
```

Full docs: https://docs.valqore.io/integrations/k9s-tui
