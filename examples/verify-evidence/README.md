# Verify the evidence yourself

This folder contains a deliberately insecure pod (`pod.yaml`) and the **evidence bundle**
Valqore produced when it evaluated that pod (`evidence.json`): the frozen input, all 512
rule outcomes, and the control statuses for every one of the 19 built-in compliance
frameworks (167 control statuses), sealed under one SHA-256 content hash.

The claim: **this evidence cannot be fabricated.** A Valqore verdict is a pure function of
the captured state, so the bundle re-runs to byte-identical results on any machine:

```bash
# requires Valqore Engine > 1.13.6 (the release after 2026-08-19), e.g.:
docker run --rm -v "$PWD:/w" ghcr.io/valqore/engine:latest verify /w/evidence.json
# -> REPRODUCED -- 512 outcomes re-ran identically; verdict BLOCK. 167 control statuses reproduced.
```

Now try to cheat, and watch it fail:

```bash
# 1) flip a recorded outcome (claim a failing rule passed)
python -c "import json;b=json.load(open('evidence.json'));b['outcomes'][0]['outcome']='PASS';json.dump(b,open('t1.json','w'))"
docker run --rm -v "$PWD:/w" ghcr.io/valqore/engine:latest verify /w/t1.json   # MISMATCH, exit 2

# 2) edit the captured state to "make" the pod compliant
python -c "import json;b=json.load(open('evidence.json'));b['inputs']['manifests'][0]['spec']['containers'][0]['securityContext']['privileged']=False;json.dump(b,open('t2.json','w'))"
docker run --rm -v "$PWD:/w" ghcr.io/valqore/engine:latest verify /w/t2.json   # hash broken AND replay disagrees

# 3) flip a compliance-control status (claim a failed control passed)
python -c "import json;b=json.load(open('evidence.json'));p=next(iter(b['controls']));c=next(iter(b['controls'][p]));b['controls'][p][c]='PASS';json.dump(b,open('t3.json','w'))"
docker run --rm -v "$PWD:/w" ghcr.io/valqore/engine:latest verify /w/t3.json   # control MISMATCH, exit 2
```

The only way to change the verdict is to change the actual state -- fix the pod, re-evaluate,
and produce a new bundle. That is the difference between evidence that is *computed* and
evidence that is *asserted*.

Make your own bundle from anything Valqore evaluates:

```bash
valqore evaluate your-infra/ --bundle evidence.json     # files: Terraform/Helm/K8s/Dockerfile
valqore scan-cloud -c azure --bundle evidence.json      # live cloud: freezes the exact snapshot
valqore verify evidence.json                            # anyone, anywhere, offline
valqore verify evidence.json --pubkey valqore.pub       # + Ed25519 signature (auditor path)
```

Note: byte-identical replay is guaranteed on the same engine version + rule set as the
bundle records (this one: see `engine_version` inside `evidence.json`); other versions
replay with a warning instead of a false claim of identity.
