# Cron Prompts

All three nightly crons. Install in order. Adjust `--tz`, `--model`, and timing to your setup. The only hard requirement is the sequence: audit first, curation second, pruning last.

---

## 1. memory-audit

```bash
openclaw cron add \
  --name "memory-audit" \
  --cron "0 22 * * *" \
  --tz "America/Your_Timezone" \
  --session isolated \
  --model "anthropic/claude-sonnet-4-6" \
  --no-deliver \
  --message 'Run the nightly memory health check.

Get today'"'"'s date first: run `date +%Y-%m-%d`.

Check each item and report PASS or FLAG with a one-line explanation.

1. Read SESSION-STATE.md. Is "Last updated" within 24 hours? Flag if stale.
2. Count lines in memory/observations.md. Flag if over 100. (If the file does not exist, PASS.)
3. Does today'"'"'s memory/YYYY-MM-DD.md exist? Flag if missing.
4. Run `openclaw cron list`. Flag any jobs in error status.
5. Check memory/ for files not modified in 7+ days (excluding daily logs older than 7 days). Flag as stale.
6. Read MEMORY.md. Is the most recent dated entry within 3 days? Flag if stale.
7. Is memory/working-buffer.md non-empty? Flag if so (should have been processed by quiet-time replay or nightly curation).
8. Run `wc -c ~/.openclaw/workspace/*.md`. Flag if any single file exceeds 18000 characters or total exceeds 135000 characters. Silent bootstrap truncation causes invisible memory loss.

End with summary: X pass, Y flags. If zero flags, just say "All clear."'
```

---

## 2. nightly-memory-curation

```bash
openclaw cron add \
  --name "nightly-memory-curation" \
  --cron "0 23 * * *" \
  --tz "America/Your_Timezone" \
  --session isolated \
  --model "anthropic/claude-sonnet-4-6" \
  --no-deliver \
  --message 'Run nightly memory curation.

1. Get today'"'"'s date: run `date +%Y-%m-%d`.
2. Read today'"'"'s daily log: memory/YYYY-MM-DD.md
3. Read memory/working-buffer.md if it exists and is non-empty. Do NOT truncate or modify it.
4. Read MEMORY.md.
5. Extract 3-8 entries from the daily log (and buffer) that are NOT already in MEMORY.md. Only extract entries that represent decisions, lessons, relationship changes, or reusable knowledge. Skip routine activity.
6. For each entry: strip timestamps and implementation details. Frame as a current-state fact. Add a tag: [decision], [lesson], [human], [surprise], or [identity].
7. Append a new dated section to MEMORY.md:

## YYYY-MM-DD
- [tag] Entry text

If nothing worth promoting, do nothing. Do not append empty sections.'
```

---

## 3. memory-pruning

```bash
openclaw cron add \
  --name "memory-pruning" \
  --cron "45 23 * * *" \
  --tz "America/Your_Timezone" \
  --session isolated \
  --model "anthropic/claude-sonnet-4-6" \
  --no-deliver \
  --message 'Run nightly memory pruning and buffer cleanup.

1. BACKUP: Copy MEMORY.md to memory/MEMORY-backup-YYYY-MM-DD.md.
2. COUNT: Count total lines in MEMORY.md.
3. EVALUATE: If under 50 lines, skip to step 6. No pruning needed.
4. PRUNE: If over 50 lines, remove entries that are: completed tasks older than 7 days, duplicates, or entries made obsolete by newer information. When in doubt, keep the entry.
5. IMMUNITY: Never prune entries dated today. Only evaluate entries older than 24 hours.
6. BUFFER: If memory/working-buffer.md exists and is non-empty, truncate it (empty the file). The curation cron has already read and promoted its contents.

Report:
- Lines before/after (or "skipped, under 50")
- Entries removed and why
- Buffer status (truncated / already empty / not present)'
```

---

## Design Notes

**Why this sequence matters:** Audit runs first to flag problems before anything is modified. Curation runs second to promote entries. Pruning runs last to trim and clean up. Reversing curation and pruning would delete entries before they're promoted.

**Why curation does NOT truncate the buffer:** If curation and truncation are in the same cron and it crashes mid-run, buffer contents are lost before promotion. Pruning runs last, so the buffer survives curation failures intact.

**Why pruning uses "skip to step 6" instead of "exit":** The buffer needs cleanup every night, not just on nights when MEMORY.md exceeds 50 lines. Without "skip to step 6," the buffer only gets cleaned when pruning actually runs.

**Why the 24-hour immunity rule:** Curation promotes entries at 23:00. Pruning evaluates at 23:45. Without immunity, a freshly promoted entry could be flagged as a "duplicate" of its daily log source and immediately deleted. The 24-hour window gives new entries time to settle.

**Delivery:** All three use `--no-deliver` so output stays in the session log. If you want alerts on failures, have your agent check cron status during its morning routine or add a separate triage cron that reads the results.

**Model choice:** All three crons work fine on Sonnet-tier models. Only the weekly brain-lint (not included here) benefits from a stronger model for cross-file reasoning.

**Observer cron not included:** The consolidation rules reference `observations.md`, which is written by a separate observer cron (e.g. every 15 minutes). That cron is deployment-specific and not included here. If you don't run an observer cron, the observations.md thresholds in AGENTS.md are harmless (nothing to prune) but won't do anything.
