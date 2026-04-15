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
1. Reconcile `.claude/engram_manifest.json` — discover all sessions, identify un-engrammed ones
2. Generate fresh extract_static.sh and extract_dynamic.sh (from Prompt 1 in the engram repo)
3. Write to .claude/, execute both, read .claude/session_signals.json
4. Synthesize memory updates (from Prompt 2 in the engram repo)
5. Tag the session in the manifest
6. **Clean up** — delete generated scripts and signal JSON files from `.claude/`:
   ```bash
   rm -f .claude/extract_static.sh .claude/extract_dynamic.sh .claude/session_signals*.json
   ```

Trigger modes:
- "engram" → current session only (live mode)
- "engram retroactive" → all un-engrammed sessions in chronological order (batch mode)
- "engram retroactive <uuid>" → one specific old session (single mode)

Scripts accept env var overrides for retroactive use:
- ENGRAM_SINCE / ENGRAM_UNTIL → date range for static signals (default: today)
- ENGRAM_TRANSCRIPT → explicit transcript path for dynamic signals (default: most recent)

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
