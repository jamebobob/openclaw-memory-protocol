> **Archived April 2026.** Built on [OpenClaw](https://github.com/openclaw/openclaw) and moved off of it when the active work migrated to [Hermes](https://github.com/hermes-agent/hermes-agent). The design ideas in this repo might still be useful. The platform wiring is obsolete.

# openclaw-memory-protocol

A layered memory persistence protocol for OpenClaw agents. Prevents context loss during compaction, automates nightly memory maintenance, and catches silent failures before they eat your data.

Built after losing context to compaction one too many times. The core architecture (significance tagging, working buffer, consolidation rules) came from researching how human brains handle memory consolidation and forgetting, then adapting those patterns to an agent that wakes up fresh every session. Audited by multiple AI systems. Stress-tested across 6+ compactions in a single session with zero data loss.

Complements [openclaw-agent-privacy](https://github.com/jamebobob/openclaw-agent-privacy) by ensuring the agent's file-based memory survives compaction. The privacy framework handles which memories each agent can access; this protocol handles making sure the memories exist in the first place.

## Why This Exists

OpenClaw's built-in `memoryFlush` is your agent's only safety net before compaction. It has three failure modes:

1. **One shot per compaction cycle.** The [official docs](https://docs.openclaw.ai/concepts/memory) confirm: "One flush per compaction cycle (tracked in sessions.json)." If context jumps past the threshold after the flush already fired, there is no second chance.

2. **Stale token counts ([#5457](https://github.com/openclaw/openclaw/issues/5457)).** The flush decision uses token counts from the *previous* turn. A large incoming message can push past the threshold before the flush even checks.

3. **Model compliance is optional.** The flush sends a prompt. The model can respond with NO_REPLY and save nothing.

One mechanism with three failure modes is not a safety net. This protocol adds layered defenses so no single failure causes data loss.

If you've hit [#5429](https://github.com/openclaw/openclaw/issues/5429) (lost days of work to silent compaction), this is the structural fix.

---

## Architecture

Two layers that reinforce each other:

- **Consolidation protocol** (manual rules in AGENTS.md): Governs how the agent writes, tags, and prunes memory during active sessions. Quality control at input time.
- **Automated pipeline** (three nightly crons): Audits health, promotes daily entries to long-term memory, prunes to keep files bounded. Runs whether or not the agent had an active session that day.

### The pipeline

```
REAL-TIME
 Every turn: Agent updates SESSION-STATE.md
 At 60% context: Agent appends to memory/working-buffer.md [safety net]
 At ~75% context: OpenClaw memoryFlush --> memory/YYYY-MM-DD.md [built-in, one-shot]
 At 80% context: Agent dumps to daily log + SESSION-STATE.md [pre-compaction]

NIGHTLY (must run in this order, 30+ min gaps)
 First: memory-audit --> flags problems, fixes nothing
 Second: nightly-memory-curation --> daily log to MEMORY.md
 Last: memory-pruning --> trims MEMORY.md + cleans buffer
```

The three real-time checkpoints (60%, 75%, 80%) ensure data reaches disk before compaction fires. If any one fails, the other two catch it. The nightly pipeline handles everything the agent couldn't get to during the day.

### File roles

| File | Purpose | Written by |
|------|---------|------------|
| SESSION-STATE.md | Active session state | Agent (every turn, overwritten each session) |
| memory/working-buffer.md | Overflow safety net | Agent (at 60% context). See [Working Buffer Lifecycle](#working-buffer-lifecycle). |
| memory/YYYY-MM-DD.md | Daily changelog, decisions, handoffs | Agent + memoryFlush |
| memory/observations.md | Extracted insights | Observer cron (writes), Agent (prunes). Dual ownership by design. |
| MEMORY.md | Curated long-term state | Nightly curation cron + Agent |

**Note:** The observer cron that writes `observations.md` is deployment-specific and not included in this repo. The three nightly crons and the consolidation rules work without it. If you don't run an observer cron, the `observations.md` thresholds in the consolidation rules are harmless but inactive.

---

## Consolidation Protocol (Manual Rules)

These go in your AGENTS.md. See [`examples/agents-md-rules.md`](examples/agents-md-rules.md) for a copy-paste block.

### Significance tagging

This is the part that came from neuroscience. Human brains don't remember everything. They tag experiences during sleep based on emotional salience, novelty, and relevance to existing knowledge, then prune the rest. Agents need the same filter, or every trivial log entry has equal weight and files grow until they get truncated.

Before logging any entry, the agent evaluates five questions:

1. Did it change a decision or plan?
2. Was it surprising or unexpected?
3. Did the human explicitly care about it?
4. Is it reusable knowledge (lesson, pattern, fix)?
5. Did it change the agent's self-understanding?

Each "yes" adds a tag: `[decision]`, `[surprise]`, `[human]`, `[lesson]`, `[identity]`.

**Entries with zero tags are pruning candidates.** This gives automated pruning a principled way to decide what to delete instead of guessing. Without it, pruning crons either delete too aggressively (losing important context) or too conservatively (files grow unbounded until bootstrap truncation silently eats the middle of your MEMORY.md).

### Active forgetting

Forgetting isn't a bug. It's a feature you need to design for. Unbounded memory files hit the bootstrap truncation limit (see [Bootstrap Truncation](#bootstrap-truncation)) and the middle of your file vanishes without warning.

| Target | Trigger | Action |
|--------|---------|--------|
| observations.md | > 100 lines | Trim to 60. Keep tagged entries. Drop zero-tag entries > 7 days old. |
| Daily logs | > 14 days old | Extract unprocessed decisions/lessons into MEMORY.md. |
| MEMORY.md | > 50 lines | Compress older entries. Move stable facts to owning files (USER.md, TOOLS.md, IDENTITY.md). |
| SESSION-STATE.md | Every new session | Overwrite completely. |

The 50-line threshold for MEMORY.md must match whatever your pruning cron uses. Two different numbers is a bug waiting to happen. (We learned this the hard way.)

### Episodic to semantic consolidation

Another pattern borrowed from neuroscience. The hippocampus replays episodic memories during sleep and gradually transfers the *meaning* to cortical long-term storage, discarding the specific details. Daily logs are your hippocampus. MEMORY.md is cortex.

When promoting from daily logs to MEMORY.md:

1. Strip timestamps and implementation details
2. Extract the decision, lesson, or relationship change
3. Frame as a current-state fact ("X is the current approach because Y" not "on March 10 we decided X")
4. If the fact is stable, move it to the owning file (USER.md for people, TOOLS.md for infrastructure, IDENTITY.md for self-knowledge)

### Reconsolidation honesty rules

In neuroscience, memory reconsolidation means that every time you recall a memory, it becomes editable. You can strengthen it, update it, or accidentally corrupt it. The same thing happens with agent memory files. Every time you read and rewrite MEMORY.md, you can silently introduce drift.

These rules prevent that:

- Never delete entries because they make the agent look bad
- If unsure what happened, say "I'm not sure" rather than fabricating a clean narrative
- Daily logs stay honest and unedited after the day passes
- When correcting MEMORY.md, acknowledge the previous understanding was wrong rather than silently overwriting

Without these, agents silently "correct" their own history. The errors compound. After a month, the agent's memory of what happened diverges from what actually happened, and nobody notices because the evidence was cleaned up.

### Pre-compaction protocol

Check context percentage every 3rd response, **or immediately after any large file read or tool output > 5K tokens.**

At **80%+ context:**
1. Immediately dump unsaved context to daily log and SESSION-STATE.md
2. Apply significance tagging to dumped entries
3. Do not ask the human for permission. Just do it.

The 3rd-response cadence avoids burning a `session_status` tool call every turn. The large-operation trigger catches sudden spikes that would slip through the regular cadence. We tested 80% across 6+ compactions in a single session with zero data loss. Lower thresholds waste working space; higher ones cut it too close.

### Quiet-time replay

During idle periods, the agent runs a mini consolidation:

1. Reread today's daily log
2. Check observations.md line count; prune if over threshold
3. Promote any outstanding tagged entries to MEMORY.md
4. Check working-buffer.md; if non-empty, process contents

This is the agent's version of sleep consolidation. Not as dramatic, but the same principle: use downtime to organize what happened while you were busy.

---

## Automated Pipeline (Crons)

Full cron prompts with install commands: [`examples/cron-prompts.md`](examples/cron-prompts.md)

### memory-audit (first in sequence)

Health check. Flags problems. Fixes nothing. This is important: the audit never writes to memory files. It only reports. Keeping read and write operations in separate crons means a bug in the audit can't corrupt your data.

Checks:

1. SESSION-STATE.md freshness (flag if > 24 hours stale)
2. observations.md line count (flag if > 100)
3. Daily log existence
4. OpenClaw cron error status
5. Stale files in memory/ (> 7 days unmodified)
6. MEMORY.md currency (flag if > 3 days since last dated entry)
7. **working-buffer.md non-empty** (contents should have been processed)
8. **Bootstrap file sizes** (flag if any file > 18K chars or total > 135K chars)

Items 7 and 8 are the ones nobody else monitors. Item 7 catches orphaned buffer contents from crashed sessions. Item 8 catches silent bootstrap truncation before it starts eating your memory (see [Bootstrap Truncation](#bootstrap-truncation)).

### nightly-memory-curation (second in sequence)

Automated episodic-to-semantic promotion:

1. Read today's daily log
2. Read working-buffer.md (if non-empty)
3. Read current MEMORY.md
4. Extract 3-8 non-obvious learnings/decisions NOT already in MEMORY.md
5. Append a new dated section to MEMORY.md

The curation cron reads the working buffer but **does not truncate it**. Truncation happens in the pruning cron. This is a deliberate design choice (see below for why).

### memory-pruning (last in sequence)

Automated forgetting + buffer cleanup:

1. Backup MEMORY.md
2. Count lines
3. If under 50 lines, **skip to step 6**
4. If over 50, delete: completed tasks > 7 days old, duplicates, obsolete entries
5. **Never prune entries dated today. Only evaluate entries > 24 hours old.**
6. **If working-buffer.md is non-empty, truncate it.**

Three design decisions that matter:

**"Skip to step 6" (step 3).** Even when MEMORY.md needs no pruning, the buffer still needs cleanup. Without this flow, the buffer only gets cleaned on nights when MEMORY.md exceeds 50 lines. Step 6 runs every night regardless.

**24-hour immunity (step 5).** The curation cron promotes entries 45 minutes earlier. Without immunity, those fresh entries could be immediately evaluated as "duplicates" of daily log content and deleted. Fresh promotions need a full day to settle. We caught this race condition during the audit. It's the kind of thing that works fine for weeks, then silently eats an important entry on a busy night.

**Failure-safe truncation (step 6 in pruning, not curation).** If curation and truncation lived in the same cron, a crash between the read and the truncate would lose buffer contents. Pruning runs last in the nightly sequence, making it the safe place for destructive cleanup. If curation crashes, the buffer survives intact for next time.

### brain-lint (weekly, recommended)

Cross-file consistency check. Reads all workspace memory files, flags contradictions. MEMORY.md says one thing, USER.md says another. A promoted fact drifted from the daily log version. TOOLS.md references infrastructure that no longer exists.

This is high-value because memory files are edited by multiple actors (agent, crons, memoryFlush) and drift is inevitable. The longer you go without checking, the more your memory files disagree with each other, and the agent has no way to know which version is right.

---

## Working Buffer Lifecycle

The working buffer (`memory/working-buffer.md`) exists because memoryFlush fires once and uses stale token counts. It's the defense against both known upstream bugs.

1. **Created:** Agent writes to it at 60% context
2. **Processed:** During quiet-time replay OR by nightly curation (read-only)
3. **Cleaned:** Truncated by pruning cron as the final nightly step
4. **Monitored:** memory-audit flags if non-empty

The buffer should normally be empty. Persistent content means sessions are ending abruptly before the agent can process the overflow.

---

## Bootstrap Truncation

This is the silent killer nobody talks about. OpenClaw truncates workspace files before injecting them into the system prompt. Your agent doesn't know it's happening. You don't know it's happening. Your memory just quietly gets smaller.

**Limits:** 20,000 characters per file (`bootstrapMaxChars`), 150,000 characters total (`bootstrapTotalMaxChars`).

When a file exceeds the limit, the middle gets dropped (community sources report a 70/20/10 head/tail/marker split; unverified in official docs). If MEMORY.md's "Recent History" section sits in the middle, it vanishes. The agent loses context with no indication.

We found a 46K file in our workspace that had been silently truncated to 20K on every single session for over a week. The bootstrap monitoring in this protocol caught it on the first run.

**Defense:** The memory-audit cron checks file sizes nightly, flagging at 90% of both limits. You can also verify manually with `/context list` (compare raw vs injected character counts for each file). Keep bootstrap files lean. Every line costs tokens on every turn.

[CodeAlive-AI/awesome-openclaw](https://github.com/CodeAlive-AI/awesome-openclaw) recommends keeping MEMORY.md under 5KB and AGENTS.md under 2KB. Your mileage will vary depending on how many workspace files you load, but the principle holds: every line in a bootstrap file should earn its place.

---

## Recommended Config

Set `reserveTokensFloor` to at least 40K (default 20K is too tight for serious use) and `softThresholdTokens` to at least 10K. This makes memoryFlush fire at ~75% of context instead of ~88%, giving you actual breathing room.

```json5
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard",
        "reserveTokensFloor": 40000,
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 10000
        }
      }
    }
  }
}
```

The math: `contextWindow - reserveTokensFloor - softThresholdTokens` = trigger point. On a 200K window with these values, flush fires at 150K tokens (75%). With defaults, it fires at 176K (88%), which is dangerously close to overflow, especially given the stale token count bug.

---

## Git-Backed Workspace

The [official docs](https://docs.openclaw.ai/concepts/agent-workspace) recommend this. Adding it here because it complements the protocol: if a cron corrupts MEMORY.md, file backups just preserve the corruption. Git gives you `git diff` and rollback.

```bash
cd ~/.openclaw/workspace && git init
# Add a nightly auto-commit to your backup cron:
git add -A && git commit -m "nightly: $(date '+%Y-%m-%d %H:%M')" 2>/dev/null || true
```

Keep `~/.openclaw/openclaw.json`, credentials, and session transcripts out of git. Only the workspace.

---

## Known Upstream Issues

| Issue | Impact | Workaround in this protocol |
|---|---|---|
| [#5457](https://github.com/openclaw/openclaw/issues/5457) | memoryFlush uses stale token counts | Working buffer at 60% + manual pre-compaction at 80% |
| [#5429](https://github.com/openclaw/openclaw/issues/5429) | Silent data loss when agent ignores flush | Three checkpoints ensure data is on disk before flush fires |
| [#7477](https://github.com/openclaw/openclaw/issues/7477) | Safeguard compaction silently fails on large contexts | reserveTokensFloor 40K+ triggers compaction earlier |
| [#17034](https://github.com/openclaw/openclaw/issues/17034) | softThresholdTokens doesn't scale with context window | Set explicitly, don't rely on defaults |

---

## Credits

Developed by [@jamebobob](https://github.com/jamebobob). Core architecture designed by the live agent (consolidation protocol, significance tagging, working buffer lifecycle, failure-safe cron ordering, pre-compaction protocol). Audited and hardened by Claude (24-hour immunity rule, bootstrap truncation monitoring, memoryFlush math correction).

Significance tagging and the consolidation/forgetting model draw from memory consolidation neuroscience research. The protocol treats agent memory the way brains treat sleep: tag what matters, replay it, promote the meaning, forget the noise.

Inspired by the OpenClaw community, especially [#5429](https://github.com/openclaw/openclaw/issues/5429) and the [VelvetShark Memory Masterclass](https://velvetshark.com/openclaw-memory-masterclass).

## License

MIT. See [LICENSE](LICENSE).
