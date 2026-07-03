# CorvinLogs — Transparent Healing Trace Telemetry

This repository is the public transparency mirror for CorvinOS healing trace
telemetry. When CorvinOS fixes a runtime error automatically (self-healing),
it can optionally record a **HealingTrace** — a scrubbed, PII-free description
of what broke and what was done to fix it. These records are published here so
the community can verify exactly what data is collected.

## What is a HealingTrace?

A HealingTrace is a tiny structured record that describes one self-healing
event. Every field is validated by an allowlist before anything is written to
disk. Unknown or unexpected fields cause the entire record to be dropped —
never silently stored.

**Opt-in only.** No data is sent unless you explicitly run:

```bash
corvin-maintainer healing-traces opt-in
```

You will be shown the full consent text before being asked to confirm.

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
| Instance token | `HMAC-SHA256(server_secret, instance_id)` | Not reversible — pseudonym only |

## What is NOT in these traces?

- Any message content, prompts, or conversation text
- Your name, e-mail address, IP address, or any personal identifier
- Configuration values of any kind
- API keys, secrets, tokens, or credentials
- Third-party plugin names or file system paths
- Stack frames from code outside the CorvinOS core

## How to verify this

The schema is in [`schema/htrace-1.0.json`](schema/htrace-1.0.json) in this
repository — a JSON Schema with `additionalProperties: false`. Any field not
in that schema causes the record to be rejected.

The enforcement code is in CorvinOS:
[`core/console/corvin_console/aco/htrace.py`](https://github.com/CorvinLabs/CorvinOS/blob/main/core/console/corvin_console/aco/htrace.py)
— the `_assert_safe_htrace()` function validates every record before it is
written to disk. The PII scanner runs before allowlist check; any trace
containing personal-data patterns is silently dropped.

To inspect a bundle yourself:

```bash
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
      <token_prefix>_YYYY-MM-DD.jsonl.gz   ← daily bundle per instance
schema/
  htrace-1.0.json     ← JSON Schema (single source of truth)
PRIVACY.md            ← detailed privacy policy
```

**Every bundle in this repository has been reviewed by the maintainer before
publication.** Raw uploads go to a private staging area first; only
pseudonymised, verified bundles are committed here.

## Managing your data

```bash
# Check current status
corvin-maintainer healing-traces status

# Stop sending and delete all local traces
corvin-maintainer healing-traces opt-out

# Request deletion of all uploaded traces (GDPR Art. 17)
corvin-maintainer healing-traces erase --confirm
```

## Privacy & GDPR

- **Legal basis:** Art. 6(1)(a) GDPR — explicit consent
- **Data controller:** Corvin Labs UG (haftungsbeschränkt), Berlin, Germany
- **Retention:** 90 days from upload, then auto-deleted
- **Privacy policy:** https://corvin-labs.com/privacy

## Related

- [CorvinOS](https://github.com/CorvinLabs/CorvinOS) — main repository
- [CorvinOS source: htrace.py](https://github.com/CorvinLabs/CorvinOS/blob/main/core/console/corvin_console/aco/htrace.py) — enforcement code
