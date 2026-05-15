---
name: Engram — automated memory consolidation
description: When user says "engram", run memory consolidation. When user says to deploy/update engram, fetch latest from GitHub.
type: reference
---

## Trigger: "engram"

When the user says "engram", execute the full Engram loop:

0. **Auto-update check** — before doing any work, check for a newer version:
   ```bash
   curl -sL https://raw.githubusercontent.com/Wise-Corp/engram/main/engram.md -o /tmp/engram_latest.md
   diff -q .claude/memory/engram.md /tmp/engram_latest.md >/dev/null 2>&1
   ```
   - If `diff` exits 0 (identical): proceed silently, clean up `/tmp/engram_latest.md`
   - If `diff` exits 1 (different): tell the user "A newer version of engram is available." and ask if they want to update before proceeding. If yes, copy `/tmp/engram_latest.md` to `.claude/memory/engram.md` and confirm. If no, proceed with the current version. Clean up `/tmp/engram_latest.md` either way.
   - If `curl` fails (no network): proceed silently, skip the check
1. Reconcile `.claude/engram_manifest.json`:
   a. Generate and run a small reconciliation script that scans all `.jsonl` transcript
      files in `~/.claude/projects/<project-path>/`, counts user+assistant messages,
      reads first/last timestamps, and adds any missing sessions to the manifest
      with `engrammed: false`
   b. Identify un-engrammed sessions. Sessions with < 10 messages are auto-skipped
      as trivial (tag them `skipped` in the manifest)
2. **For each un-engrammed session, in chronological order:**
   a. **Apply Prompt 1** to generate fresh `extract_static.sh` and
      `extract_dynamic.py` adapted to the project. Do NOT reuse scripts
      across sessions — `{{CURRENT_MEMORY}}` may differ between iterations.
      Feed Prompt 1 with:
      - **Current session (live mode):** the full current persistent memory
        PLUS the volatile memory of the ongoing conversation.
      - **Old session (retroactive mode):** ONLY the persistent memory
        files (`.claude/memory/*.md`). Do NOT include the current
        conversation context — that would leak future knowledge into the
        extraction scripts for a past session.
   b. **Run both scripts** for this session only:
      - Static: `ENGRAM_SINCE` / `ENGRAM_UNTIL` set to the session's
        `first_ts` and `last_ts` from the manifest (date + time, not just
        date — this prevents overlap between same-day sessions).
        Output → `.claude/session_signals_<prefix>_static.json`
      - Dynamic: `ENGRAM_TRANSCRIPT` set to the session `.jsonl`.
        Output → `.claude/session_signals_<prefix>_dynamic.json`
   c. **Apply Prompt 2** — read BOTH `_static` and `_dynamic` signal files
      for this session and synthesize memory updates. Do NOT read or inspect
      the signal JSONs outside Prompt 2 — reading them between extraction
      and synthesis wastes context.
   d. Tag the session as `engrammed` in the manifest.
3. Delete generated scripts and signal JSONs:
   ```bash
   rm -f .claude/extract_static.sh .claude/extract_dynamic.py .claude/session_signals*.json
   ```

Trigger modes:
- "engram" → current session only (live mode)
- "engram retroactive" → all un-engrammed sessions in chronological order (batch mode)
- "engram retroactive <uuid>" → one specific old session (single mode)

Scripts accept env var overrides:
- `ENGRAM_SINCE` / `ENGRAM_UNTIL` → date+time range for static extraction
  (default: today at 00:00, open-ended). Git supports timestamps, so use the
  manifest's `first_ts` / `last_ts` with second precision — this avoids giving
  two sessions on the same day identical static signals.
- `ENGRAM_TRANSCRIPT` → explicit transcript `.jsonl` path (default: most recent).
- `ENGRAM_BATCH_ID` → session UUID prefix. Static writes to
  `.claude/session_signals_<id>_static.json`; dynamic to
  `.claude/session_signals_<id>_dynamic.json`.

Sessions with < 10 messages are auto-skipped as trivial.

## Trigger: deploy/update engram

When the user asks to deploy, install, or update engram (referencing https://github.com/Wise-Corp/engram), do this:

1. Fetch the latest engram.md from the repo:
   ```bash
   curl -sL https://raw.githubusercontent.com/Wise-Corp/engram/main/engram.md -o /tmp/engram_latest.md
   ```
2. Copy it into the current project's memory:
   ```bash
   mkdir -p .claude/memory
   cp /tmp/engram_latest.md .claude/memory/engram.md
   ```
3. Add to `.claude/memory/MEMORY.md` (create if missing) — add this line if not already present:
   ```
   - [Engram](engram.md) — automated memory consolidation trigger
   ```
4. Clean up:
   ```bash
   rm -f /tmp/engram_latest.md
   ```
5. Confirm: "Engram deployed. Say 'engram' to consolidate memory."

For the prompts (needed during engram runs), fetch them on demand:
- `https://raw.githubusercontent.com/Wise-Corp/engram/main/prompts/prompt1-extraction.md`
- `https://raw.githubusercontent.com/Wise-Corp/engram/main/prompts/prompt2-synthesis.md`

Source: https://github.com/Wise-Corp/engram
