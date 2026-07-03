# CorvinLogs — Transparent Healing Trace Telemetry

This repository is the public transparency mirror for CorvinOS healing trace telemetry,
as specified in [ADR-0180](https://github.com/CorvinLabs/Corvin-ADR/blob/main/decisions/0180-cross-instance-healing-trace-aggregation.md).

## What is this?

When a CorvinOS instance self-heals (fixes a runtime error autonomously), it can
optionally record a **HealingTrace** — a scrubbed, PII-free description of what broke
and what was done to fix it. These traces are collected here so that:

1. The maintainer can identify which bugs affect multiple users and prioritise fixes.
2. The community can verify exactly what data is collected (transparency).

**Every file in this repository has already been reviewed by the maintainer before
being published here.** Raw uploads go to a private staging area first; only
pseudonymised, verified bundles are committed here.

## What data is in these traces?

Each trace contains **only**:

| Field | Example | Notes |
|---|---|---|
| CorvinOS version | `0.9.60` | |
| Platform | `linux/x86_64` | OS family + CPU arch, nothing else |
| Python version | `3.12` | Major.minor only |
| Error type | `AttributeError` | Exception class name |
| Error location | `chat_runtime / stream_turn` | Module + function, from CorvinOS core only |
| Error template | `object '{}' has no attribute '{}'` | Structure without values |
| Last N event names | `["os_turn.started", "heal.triggered"]` | Audit event *names* only, from a fixed allowlist |
| Heal action | `restart_service` | What was tried |
| Heal outcome | `success` | |
| Config key hash | `sha256(sorted config key names)` | Key NAMES only, never values |
| Day | `2026-07-03` | Day only, no time |
| Instance token | `HMAC-SHA256(server_secret, instance_id)` | Not reversible |

## What is NOT in these traces?

- Any message content, prompts, or conversation text
- Your name, e-mail address, IP address, or any personal identifier
- Configuration values of any kind
- API keys, secrets, tokens, or credentials
- Third-party plugin names or file system paths
- Stack frames from code outside the CorvinOS core

## How to verify this

The schema is fully specified in [ADR-0180 §1](https://github.com/CorvinLabs/Corvin-ADR/blob/main/decisions/0180-cross-instance-healing-trace-aggregation.md#1-healingtrace--schema-allow-list-enforced-pii-free-by-construction).

The `_assert_safe_htrace()` function in
[`core/console/corvin_console/aco/htrace.py`](https://github.com/CorvinLabs/CorvinOS/blob/main/core/console/corvin_console/aco/htrace.py)
enforces this schema before any record is written.

To inspect a bundle locally:

```bash
# Download and decompress
curl -L <bundle_url> | gunzip | python3 -c "
import sys, json
for line in sys.stdin:
    rec = json.loads(line)
    print(json.dumps(rec, indent=2))
    print('Fields:', sorted(rec.keys()))
"
```

## Repository structure

```
traces/
  htrace-v1/
    YYYY-MM-DD/
      <instance_token_prefix>_YYYY-MM-DD.jsonl.gz   ← daily bundle per instance
schema/
  htrace-1.0.json     ← JSON Schema for validation
PRIVACY.md            ← detailed privacy policy
```

## Privacy & GDPR

- **Legal basis:** Art. 6(1)(a) GDPR — explicit consent
- **Data controller:** Corvin Labs UG (haftungsbeschränkt), Berlin, Germany
- **Retention:** 90 days from upload, then auto-deleted
- **Opt-out:** `corvin-maintainer healing-traces opt-out` — immediate effect
- **Erasure (Art. 17):** `corvin-maintainer healing-traces erase --confirm`
- **Privacy policy:** https://corvin-labs.com/privacy

## Related

- [ADR-0180](https://github.com/CorvinLabs/Corvin-ADR/blob/main/decisions/0180-cross-instance-healing-trace-aggregation.md) — design document
- [ADR-0179](https://github.com/CorvinLabs/Corvin-ADR/blob/main/decisions/0179-telemetry-driven-proven-repair.md) — upstream telemetry layer
- [CorvinOS](https://github.com/CorvinLabs/CorvinOS) — main repository
