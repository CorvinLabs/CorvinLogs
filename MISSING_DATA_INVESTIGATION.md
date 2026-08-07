# Missing Data Investigation: 2026-07-05

**Generated:** 2026-08-07 (post-dedupliation audit)

## Summary

One date remains with sparse/missing trace data after reorganization and deduplication:
- **2026-07-05**: Should have healing traces from production instances, but archive is empty
- Timeline: 2026-07-04 ✓ (1 record) → 2026-07-05 ✗ (0 records) → 2026-07-06 ✓ (2 records)
- Gap duration: 1 day

## Active Instances on Adjacent Dates

Main instance `a5e1103f74f3` (222 total records, active 2026-07-09 to 2026-08-06):
- 2026-07-04: No records (but instance may not have been running)
- 2026-07-05: **NO RECORDS** ← Gap
- 2026-07-06: 2 records ✓ (active after reorganization)
- 2026-07-07: 3 records ✓
- 2026-07-08 onward: consistent daily activity

Secondary instances (occasional):
- `897273db5f6f`: starts 2026-07-14 (sporadic, last seen 2026-08-07)
- `2c85072ef`: only 2026-07-07 activity
- Others: 1-3 records each across July-August

## Hypothesis: Why 2026-07-05 is Missing

### Scenario A: No Healing Events (Healthy Day)
- Instance `a5e1103f74f3` was running but had zero healing events
- No errors → no traces collected
- **Probability: MEDIUM** — but unlikely with 14 days of daily activity before/after

### Scenario B: Collection But Upload Failure
- Healing traces were generated on 2026-07-05
- Upload to staging area succeeded but never made it to public repo
- **Probability: MEDIUM** — common in incomplete migration/sync passes

### Scenario C: Staging Area Retention Policy
- Traces were in private staging for 90 days (per ADR-0180 retention: 90d)
- Auto-deleted after retention window
- Now shows as gap in public repo
- **Probability: LOW** — gap is only 3 days old as of audit

### Scenario D: Server Timezone Edge Case
- Instance generated traces on 2026-07-04 23:59 UTC
- Marked with `ts_day: 2026-07-04`
- But uploaded on 2026-07-05 (next day) to wrong directory
- Then reorganized to 2026-07-04/ during audit
- **Probability: MEDIUM** — matches pattern of all other 1-day divergences (FIXED)

## Investigation Checklist

To determine root cause, check:

### Server-Side
- [ ] CorvinLogs staging upload log for 2026-07-05 (`~/.corvin/cloud/upload.log` or similar)
  - Query: `grep "2026-07-05" *.log | grep -E "(success|error|retry)"`
  - Expected: Upload record with payload size, instance ID, hash
  
- [ ] Railway backend telemetry (https://railway.app – project CorvinLogs)
  - Check: API request volume on 2026-07-05
  - Expected: POST requests from instances around midnight UTC per instance's timezone

- [ ] Instance heartbeat data for `a5e1103f74f3` on 2026-07-05
  - Check: Is the instance running? (heartbeat.py, ADR-0186)
  - Expected: ≥ 1 heartbeat message per day

### CorvinOS Instances
- [ ] Check instance `a5e1103f74f3` local trace dir on that date
  - Path: `~/.corvin/healing-traces/2026-07-05.jsonl` (if still retained)
  - Command: `ls -la ~/.corvin/healing-traces/2026-07-0[34].jsonl`
  - If present: why wasn't it uploaded?

- [ ] Check CorvinOS version on that instance
  - From latest trace: version 0.10.25 (very old)
  - But instance also sent 0.10.117 recently
  - Suggests: instance was updated, or multiple instances share token?

## Remediation Options

1. **If traces exist in staging**: Re-upload via `corvin-maintainer healing-traces migrate`
2. **If traces were auto-deleted**: Accept gap; note in docs that 90-day retention may cause gaps
3. **If upload failed silently**: Fix upload retry logic in htrace.py
4. **If timezone bug**: Validate that fix in Step 1 (reorganization) covers all cases

## Related ADRs & Docs

- **ADR-0180** — Healing trace telemetry design (retention: 90d)
- **ADR-0186** — Instance heartbeat protocol (daily ping, no data loss)
- **CLAUDE.md** — Default-ON telemetry (opt-out: `spec.telemetry.healing_traces: false`)

---

**Next action:** Server team to check staging upload logs and heartbeat data for 2026-07-05.

