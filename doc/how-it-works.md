# How Engram Works

## The Three-Layer Architecture

AI memory consolidation typically dumps everything into the context window and lets the LLM sort it out. Engram splits the work into what requires reasoning and what requires processing:

| Layer | What happens | Where | Cost |
|-------|-------------|-------|------|
| **1. Query Generation** | LLM produces two extraction scripts (shell + python3) tailored to the project's current state and memory | API | ~few hundred tokens |
| **2. Signal Extraction** | Local machine runs the scripts: git diffs, frequency analysis, transcript parsing, convention detection | Edge | Zero |
| **3. Synthesis** | LLM receives structured signals and produces an updated memory file | API | Focused, minimal noise |

Two small API calls. Arbitrarily complex local processing in between.

## What Gets Extracted

### Static Signals (`extract_static.sh`)

Deterministic data from the codebase and git history:

- **Session diff analysis** — file changes, commits, additions, deletions in the date range
- **Frequency analysis** — most-touched files and directories (last 5 days)
- **Error patterns** — removed debug statements, recent log errors
- **Convention detection** — import patterns, naming conventions, endpoint patterns
- **Contradiction detection** — persistent memory assertions vs. actual codebase state
- **Dependency changes** — diffs of requirements.txt / package.json / Cargo.toml etc.
- **TODO/FIXME scan** — new and existing TODOs

### Dynamic Signals (`extract_dynamic.sh`)

Behavioral data from Claude Code session transcripts, in two dimensions:

**Dimension A — User Preferences** (how the developer works):
- Acceptance/rejection/modification rates for agent proposals
- Rejection context — what the user corrected and why
- Coding preference keywords — style indicators from correction messages
- Iteration depth — files edited 3+ times (friction areas)
- Prompt complexity trend — rising length signals the agent is losing context
- Explicit style statements — "I prefer", "always use", "never use"

**Dimension B — Domain Knowledge** (what the project is about):
- Terminology corrections — "X is not Y", "when I say X I mean"
- Business rule statements — constraints with reasons
- Workflow/state ordering — "can't skip", "must go through"
- Entity relationships — "X belongs to Y", "X has many Y"
- Repeated explanations — concepts the agent keeps misunderstanding
- Domain vocabulary — project-specific nouns appearing 3+ times

## The Manifest

File: `.claude/engram_manifest.json`

Tracks which sessions have been engrammed:

```json
{
  "version": 1,
  "project_path": "/home/user/myproject",
  "sessions": {
    "954fa331-...": {
      "first_timestamp": "2026-03-22T14:30:00Z",
      "last_timestamp": "2026-03-23T12:40:00Z",
      "message_count": 247,
      "engrammed": true,
      "engrammed_at": "2026-04-11T14:30:00Z",
      "mode": "batch"
    }
  }
}
```

Modes: `"live"` (current session), `"batch"` (retroactive batch), `"single"` (retroactive single), `"skipped_trivial"` (< 10 messages).

Before each run, a reconciliation step scans all `.jsonl` transcript files and adds any missing sessions to the manifest. This is a lightweight local operation — read two lines per file plus a line count.

## Script Lifecycle

Scripts are **ephemeral** — generated fresh each session by Prompt 1. They are adapted to the specific project (its language, structure, conventions) and placed in `.claude/`. If a generated script has a bug, the fix goes in the prompt, not the script.

## Environment Variable Overrides

For retroactive processing of old sessions:

| Variable | Purpose | Default |
|----------|---------|---------|
| `ENGRAM_SINCE` | Start date for git log queries | Today |
| `ENGRAM_UNTIL` | End date for git log queries | Open-ended |
| `ENGRAM_TRANSCRIPT` | Explicit transcript file path | Most recent .jsonl |
| `ENGRAM_BATCH_ID` | UUID prefix for output filename (avoids overwrites in batch mode) | None |

## Constraints

- Extraction scripts **never** call the LLM API — Layer 2 is deterministic, local, zero-cost
- Scripts must be portable across macOS and Linux (no GNU-only flags)
- Scripts must complete in under 15 seconds total
- JSON assembly uses temp files with `safe_load()`, never shell variable interpolation into Python heredocs
- Temp files are cleaned up via `trap 'rm -rf "$TMPDIR"' EXIT`

## Further Reading

- [Design rationale](article.md) — the full argument for edge-augmented consolidation
- [PoC history](poc.md) — version changelog (v1 through v4.1), design decisions
- [Original design conversation](transcript.md) — the session that started engram
