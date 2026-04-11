---
name: Engram — automated memory consolidation
description: When user says "engram", run memory consolidation. Supports retroactive mode for old sessions.
type: reference
---

When the user says "engram", execute the full Engram loop:

1. Reconcile `.claude/engram_manifest.json` — discover all sessions, identify un-engrammed ones
2. Generate fresh extract_static.sh and extract_dynamic.sh (see [prompts/prompt1-extraction.md](prompts/prompt1-extraction.md))
3. Write to .claude/, execute both, read .claude/session_signals.json
4. Synthesize memory updates (see [prompts/prompt2-synthesis.md](prompts/prompt2-synthesis.md))
5. Tag the session in the manifest

Trigger modes:
- "engram" → current session only (live mode)
- "engram retroactive" → all un-engrammed sessions in chronological order (batch mode)
- "engram retroactive <uuid>" → one specific old session (single mode)

Scripts accept env var overrides for retroactive use:
- ENGRAM_SINCE / ENGRAM_UNTIL → date range for static signals (default: today)
- ENGRAM_TRANSCRIPT → explicit transcript path for dynamic signals (default: most recent)

Sessions with < 10 messages are auto-skipped as trivial.

Source: https://github.com/Wise-Corp/engram
