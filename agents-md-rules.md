# AGENTS.md Memory Rules

Copy this block into your AGENTS.md. Adjust thresholds if your context window differs from 200K.

**Note:** The Quiet-Time Replay and Thresholds sections reference `observations.md`. This file is written by a separate observer cron (not included in this repo). If you don't run an observer cron, those references are harmless but inactive. Remove them from your copy if you prefer a clean block.

---

```markdown
## Memory Rules

### Write Discipline
- Update SESSION-STATE.md every turn with active context
- After any significant work, immediately append to memory/YYYY-MM-DD.md
- Do not batch writes. Compaction can happen at any time and erases what isn't on disk.

### Significance Tagging
Before logging any entry, ask:
1. Did it change a decision or plan? --> [decision]
2. Was it surprising? --> [surprise]
3. Did the human explicitly care? --> [human]
4. Is it reusable knowledge? --> [lesson]
5. Did it change my self-understanding? --> [identity]
Zero-tag entries are pruning candidates.

### Pre-Compaction
Check context % every 3rd response, or immediately after large file reads / tool output > 5K tokens.
At 60%: append unsaved context to memory/working-buffer.md.
At 80%: dump everything unsaved to daily log + SESSION-STATE.md. Apply tags. Do not ask permission.

### Quiet-Time Replay
During idle periods (no active conversation), run a mini consolidation:
- Reread today's daily log
- Check observations.md line count; prune if over 100
- Promote any outstanding tagged entries to MEMORY.md
- Process working-buffer.md if non-empty (promote or discard)

### Search Before Acting
Before answering questions about past decisions, preferences, or context: use memory_search first, then memory_get. Do not guess from conversation alone.

### Honesty
- Never delete entries because they make me look bad
- If unsure what happened, say "I'm not sure"
- Daily logs stay unedited after the day passes
- When correcting MEMORY.md, acknowledge the old understanding was wrong

### Promotion
When promoting from daily logs to MEMORY.md:
- Strip timestamps and implementation details
- Frame as current-state fact, not historical event
- If the fact is stable, move to the owning file (USER.md, TOOLS.md, IDENTITY.md)

### Thresholds
- observations.md > 100 lines: trim to 60, keep tagged, drop zero-tag > 7 days
- Daily logs > 14 days: extract unprocessed decisions into MEMORY.md
- MEMORY.md > 50 lines: compress, move stable facts to owning files
- SESSION-STATE.md: overwrite completely each session
```
