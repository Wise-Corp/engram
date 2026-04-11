# Engram — Edge-Augmented Memory Consolidation for AI Coding Agents
## PoC v4 — Session Tagging & Retroactive Engramming

### What Changed in v4

v3 automated the full loop but only processed the current session, with no memory of what had already been engrammed. Run it twice on the same session? Double-processed. Forget to run it? Lost forever.

v4 adds:
- **Session tagging**: a manifest tracks which sessions have been engrammed, preventing double-processing
- **Retroactive engramming**: process old un-engrammed sessions — one by one, or all at once
- **Trivial session filtering**: sessions with fewer than 10 messages are auto-skipped

### Trigger Modes

| Command | Behavior |
|---|---|
| `engram` | Process the **current session** only (v3 behavior + tagging) |
| `engram retroactive` | **Batch mode** (default): process ALL un-engrammed sessions chronologically, synthesize once at the end |
| `engram retroactive <uuid>` | **Single mode**: process one specific old session, synthesize immediately |

Chronological order (oldest first) is enforced in all modes — memory should build up progressively.

---

### Manifest

File: `.claude/engram_manifest.json`

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

Fields:
- `first_timestamp` / `last_timestamp`: derived from the JSONL file (first and last message timestamps)
- `message_count`: total user + assistant messages (not counting system/tool metadata lines)
- `engrammed`: whether this session has been processed
- `mode`: how it was processed — `"live"` (current session), `"batch"` (retroactive batch), `"single"` (retroactive single), `"skipped_trivial"` (auto-skipped, < 10 messages)

### Manifest Reconciliation

Before any engram run, the agent generates and runs a small reconciliation script that:

1. Scans all `.jsonl` files in `~/.claude/projects/<project-path>/`
2. Reads `.claude/engram_manifest.json` (creates if missing)
3. For each JSONL not in the manifest: reads first/last lines for timestamps, counts messages, adds with `engrammed: false`
4. Reports: N total sessions, M un-engrammed, K trivial (< 10 messages)

This is a lightweight local operation — read two lines per file plus a line count.

---

### Script Adaptations

The extraction scripts from v3 gain environment variable overrides for retroactive use:

#### `extract_static.sh` — Date Range Override

```bash
# Defaults to today (v3 behavior) when env vars are not set
SINCE="${ENGRAM_SINCE:-$(date +%Y-%m-%d)}"
UNTIL="${ENGRAM_UNTIL:-}"

# All git log calls use the range
git log --since="$SINCE" ${UNTIL:+--until="$UNTIL"} ...
```

For retroactive sessions, the agent invokes:
```bash
ENGRAM_SINCE="2026-03-22" ENGRAM_UNTIL="2026-03-24" bash .claude/extract_static.sh
```

The `--until` is critical for retroactive mode — without it, a session from March 22 would pick up all commits from then to present.

#### `extract_dynamic.sh` — Transcript Override

```bash
# Defaults to most recent transcript (v3 behavior) when env var is not set
if [ -n "${ENGRAM_TRANSCRIPT:-}" ]; then
    TRANSCRIPT="$ENGRAM_TRANSCRIPT"
else
    TRANSCRIPT=$(ls -t "$TRANSCRIPT_DIR"/*.jsonl | head -1)
fi
```

For retroactive sessions:
```bash
ENGRAM_TRANSCRIPT="$HOME/.claude/projects/-home-user-myproject/954fa331-....jsonl" bash .claude/extract_dynamic.sh
```

---

### Workflow: Live Mode (`engram`)

Same as v3, plus:

1. **Reconcile manifest** before starting
2. After synthesis, **tag the current session** in the manifest: `engrammed: true`, `mode: "live"`

### Workflow: Batch Retroactive (`engram retroactive`)

1. **Reconcile manifest** — discover all sessions, identify un-engrammed ones
2. **Sort** un-engrammed sessions by `first_timestamp` ascending (oldest first)
3. **Filter out** the current session (still active, not ready)
4. **Auto-skip trivial** sessions (< 10 messages) — tag as `skipped_trivial`
5. **For each session** in chronological order:
   - Derive date range from `first_timestamp` / `last_timestamp`
   - Run `extract_static.sh` with `ENGRAM_SINCE` / `ENGRAM_UNTIL`
   - Run `extract_dynamic.sh` with `ENGRAM_TRANSCRIPT`
   - Accumulate the signals JSON into an ordered list
   - Tag session in manifest: `engrammed: true`, `mode: "batch"`
6. **Synthesize once** at the end — Prompt 2 receives the full accumulated signals array
7. **Write** updated manifest

The accumulated signals structure:
```json
{
  "batch_mode": true,
  "sessions_processed": 5,
  "signals": [
    {
      "session_uuid": "954fa331-...",
      "session_date": "2026-03-22 to 2026-03-23",
      "static": { ... },
      "dynamic": { ... }
    },
    ...
  ]
}
```

**Why single synthesis?** N synthesis calls would be N expensive LLM calls. A single call sees the full arc of how the project evolved, can spot preference drift, and produces a coherent memory update.

### Workflow: Single Retroactive (`engram retroactive <uuid>`)

1. Validate UUID exists in the transcript directory
2. Extract date range from that session
3. Run `extract_static.sh` with date range
4. Run `extract_dynamic.sh` with transcript path
5. **Synthesize immediately** (Prompt 2, single signals object)
6. Tag session in manifest: `engrammed: true`, `mode: "single"`

---

## v4.1 — Dynamic Signal Quality Improvements

Field-tested with a 10-session retroactive batch on wisecorp-treso (2026-04-11). The extraction pipeline runs correctly, but the dynamic signal extraction in `extract_dynamic.sh` produces noisy output. These are Prompt 1 refinements — fix the prompt, not the generated scripts.

### 1. Incomplete System Content Stripping

**Problem**: Only `<ide_` and `<system-reminder` prefixes are filtered. Session continuation summaries, screenshot metadata (`[Image: original...Multiply coordinates...]`), and other injected system content leak into the analysis, polluting domain vocabulary and business rule signals.

**Fix in Prompt 1**: Expand the content filter to strip:
- Lines matching `\[Image:.*\]` (screenshot metadata)
- Lines matching `Multiply coordinates` or `displayed at .* resolution`
- Session continuation / context-restore summaries (system-injected recaps at session start)
- Any content inside XML-style tags with known system prefixes (`<system-`, `<ide_`, `<command-`)

### 2. Acceptance/Rejection Regex Too Broad

**Problem**: Keyword-based classification produces false positives. "no" inside "for now" triggers a false rejection. "actually" in explanatory context ("actually, the reason is...") triggers a false rejection.

**Fix in Prompt 1**: Switch from bare keyword matching to context-aware patterns:
- Require rejection keywords at sentence start or after punctuation, not mid-phrase
- Exclude "no" when followed by continuation words ("no need", "for now", "no worries")
- Exclude "actually" when followed by explanation patterns ("actually, the reason", "actually it's because")
- Consider requiring the user message to be short (< 50 words) for simple accept/reject classification — longer messages are more likely MODIFIED or explanatory

### 3. Domain Vocabulary Noise

**Problem**: UI/system terms leak into the domain vocabulary frequency list — words like "doublon", "image", "original", "coordinates", "multiply", "displayed", "properties", "settings", "plugin", "marketplace" appear as domain terms when they're IDE or system artifacts.

**Fix in Prompt 1**: Expand the noise filter wordlist for domain vocabulary extraction (signal 13) to exclude:
- IDE/UI terms: `plugin`, `marketplace`, `settings`, `properties`, `extension`, `workspace`, `panel`, `sidebar`, `toolbar`, `palette`
- Image/screenshot artifacts: `image`, `original`, `coordinates`, `multiply`, `displayed`, `resolution`, `screenshot`, `pixel`
- Common non-domain terms: `doublon`, `duplicate`, `default`, `configuration`, `parameter`, `option`

### 4. Business Rule Detection Over-Triggers

**Problem**: Signal 8 (business rule statements) triggers on "because" in normal conversational sentences ("I did X because Y was failing"), not actual business rule declarations.

**Fix in Prompt 1**: Require compound pattern matching — a "because" alone is not enough. Require at least one of:
- Imperative verb + constraint: "must", "can't", "never", "always", "only after", "never before"
- Rule framing: "the rule is", "we have to", "it must go through"
- The sentence contains a domain entity (from the vocabulary list) + a constraint keyword
- Exclude sentences that are clearly debugging context ("because it was failing", "because the error", "because the test")

---

## PROMPT 1 — Generate Extraction Scripts

```
I want to consolidate what happened in this session into persistent project memory.
Generate TWO shell scripts adapted to this project.

Here is my current persistent memory:
<memory>
{{CURRENT_MEMORY}}
</memory>

### Script 1: `extract_static.sh` — Project State Signals

Extracts deterministic signals from the codebase and git history.

**Date range**: accept `ENGRAM_SINCE` and `ENGRAM_UNTIL` environment variables.
Default `ENGRAM_SINCE` to today, `ENGRAM_UNTIL` to empty (open-ended).
Replace all `git log --since` calls with `--since="$SINCE" ${UNTIL:+--until="$UNTIL"}`.

Signals to extract:
1. **Session diff analysis**: `git diff` / `git log` for file changes, commits, additions, deletions in the date range
2. **Frequency analysis**: most-touched files and directories over the last 5 days from `SINCE`
3. **Error pattern extraction**: removed debug statements, recent log errors
4. **Convention detection**: import patterns, naming conventions, endpoint patterns in changed files
5. **Contradiction detection**: compare persistent memory assertions against actual codebase state
6. **Dependency changes**: diff of requirements.txt / package.json / Cargo.toml etc.
7. **TODO/FIXME scan**: new TODOs added in the date range, existing ones in codebase

### Script 2: `extract_dynamic.sh` — Interaction & Preference Signals

Analyzes a Claude Code session transcript to extract behavioral signals.

**Transcript selection**: accept `ENGRAM_TRANSCRIPT` environment variable.
When set, use it directly. When unset, find the most recently modified `.jsonl`
in `~/.claude/projects/<project-path>/`.

**Finding the transcript directory:**
- Claude Code stores transcripts at `~/.claude/projects/<project-path>/<uuid>.jsonl`
  where `<project-path>` is the absolute path with slashes replaced by dashes
- Pick the most recently modified `.jsonl` file (by `ls -t`, NOT `find -printf`)
- Transcript JSONL format: each line is a JSON object. Messages have a `message` field
  containing `role` ("user"/"assistant") and `content` (string or list of
  `{"type": "text", "text": "..."}` blocks)

Extract signals in TWO dimensions:

#### Dimension A: User Preferences (how the developer works)

1. **Acceptance rate**: Classify user messages following assistant turns:
   - ACCEPTED: "yes", "do it", "looks good", "go ahead", "perfect", "great", "ok", "👍", "lgtm"
   - REJECTED: "no", "don't", "stop", "wrong", "not what I", "actually", "wait", "instead", "revert"
   - MODIFIED: "but change", "almost but", "close but", "can you also", "one thing", "tweak", "adjust"
   - Output: counts and percentages

2. **Rejection context**: For REJECTED/MODIFIED responses, extract the user's correction
   (first 200 chars) and the preceding assistant action type (file_edit, bash_command, etc.)

3. **Coding preference keywords**: Word frequency in correction messages,
   filtered for style indicators: "simpler", "verbose", "split", "inline", "smaller", "cleaner"

4. **Iteration depth**: Files edited 3+ times consecutively → "high-friction" areas

5. **Prompt complexity trend**: Average message length in quartiles
   (rising = Claude losing context, user compensating)

6. **Explicit style preferences**: Grep for "I prefer", "always use", "never use", "don't like"

#### Dimension B: Domain Knowledge (what the project is about)

7. **Terminology corrections**: "X is not Y", "when I say X I mean", "don't confuse X with Y"

8. **Business rule statements**: "because", "the rule is", "we have to", "we can't because",
   "it must go through", "only after", "never before"

9. **Workflow/state corrections**: "first we need to", "has to go through", "the order is",
   "can't skip"

10. **Entity relationship clarifications**: "X belongs to Y", "X has many Y",
    "X is not the same as Y"

11. **Repeated explanations**: User messages >50 words that are explanatory (few backticks,
    few file paths) — these signal concepts Claude keeps misunderstanding

12. **Abandoned approaches with domain reason**: "that won't work because [business reason]"

13. **Domain vocabulary frequency**: Nouns appearing 3+ times that aren't common programming
    terms → the project's domain lexicon

**Script requirements:**
- Output to `.claude/session_signals.json` (or `.claude/session_signals_<uuid_prefix>.json`
  when `ENGRAM_BATCH_ID` env var is set, to avoid overwriting between batch iterations)
- Use standard Unix tools + python3 for JSON
- Handle missing transcripts gracefully
- Portable macOS/Linux (no GNU-only flags)
- JSON safety: never interpolate shell vars into Python heredocs — use temp files
  or pass via sys.argv / environment
- `trap` to clean up temp files
```

---

## PROMPT 2 — Synthesize Memory Update

```
Here are the structured signals extracted from my project.

<signals>
{{SIGNALS_JSON — single object for live/single mode, or array for batch mode}}
</signals>

Current persistent memory:
<current_memory>
{{CURRENT_MEMORY}}
</current_memory>

Update the persistent memory. Follow these rules:

## Content rules
1. Preserve high-value existing entries that are still true
2. Update contradicted facts with "(updated YYYY-MM-DD)" note
3. Add new learnings from signals (conventions, architecture, deps)
4. Prune stale entries not reflected in recent signals
5. Use absolute dates, never "recently" or "yesterday"
6. If signals are thin (light session, no corrections, no code changes) — skip the update
   and report that. Don't force changes when there's nothing to consolidate.

## Batch mode rules (when signals is an array)
- Signals are ordered chronologically (oldest session first)
- Later sessions supersede earlier ones when they contradict
- Domain knowledge should reflect the cumulative understanding
- User preferences may evolve — note trends, weight recent sessions more heavily
- A single synthesis pass should produce ONE coherent memory update, not N incremental ones

## Structure
Use these sections (adapt to your memory format — CLAUDE.md, individual files, etc.):
- **Project Overview**: what this project is and does
- **Domain Model**: business entities, relationships, disambiguations.
  Written to prevent AI from confusing related-but-distinct concepts
- **Business Rules**: hard constraints, phrased as non-negotiable rules with reasons
- **Workflow & State Machines**: process flows with explicit ordering
- **Domain Vocabulary**: project-specific terminology and meaning
- **Architecture Decisions**: key technical choices and rationale
- **Conventions**: coding patterns, naming, file organization
- **User Preferences**: actionable instructions for future sessions
- **Active Work**: current focus areas
- **Known Issues**: bugs, tech debt, friction points

## Domain Knowledge sections (from Dimension B signals)
- Every terminology correction → vocabulary entry or disambiguation note
- Every business rule → hard constraint with its reason
- Every workflow correction → explicit ordering constraint
- Every repeated explanation → extra emphasis (Claude keeps getting this wrong)
- Phrase as RULES, not descriptions:
  "NEVER confuse Client (company entity) with User (login account)."
  "RULE: An invoice can only be generated from a validated order."
  "WORKFLOW: Payment must be recorded BEFORE delivery confirmation is sent."

## User Preferences section (from Dimension A signals)
- High acceptance rate for certain actions → note as strengths
- Consistently rejected patterns → note as anti-patterns FOR THIS USER
- Rising prompt complexity → note what topics caused confusion
- High iteration depth files → note friction areas
- Phrase as actionable instructions:
  "User prefers small, focused commits over large batches"
  "User rejects class-based patterns — always use functions"
  "When editing templates, propose changes in small increments — high iteration area"
```

---

## v4 Automation — Memory File Template

For Claude Code projects using auto-memory, save this as a reference memory file:

```markdown
---
name: Engram — automated memory consolidation
description: When user says "engram", run memory consolidation. Supports retroactive mode for old sessions.
type: reference
---

When the user says "engram", execute the full Engram loop:

1. Reconcile `.claude/engram_manifest.json` — discover all sessions, identify un-engrammed ones
2. Generate fresh extract_static.sh and extract_dynamic.sh (Prompt 1)
3. Write to .claude/, execute both, read .claude/session_signals.json
4. Synthesize memory updates (Prompt 2)
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
```

---

## What Changed Across Versions

| | v1 | v2 | v3 | v4 |
|---|---|---|---|---|
| Static signals (git, deps, conventions) | ✅ | ✅ | ✅ | ✅ |
| Dynamic signals (transcript analysis) | ❌ | ✅ | ✅ | ✅ |
| User preference extraction | ❌ | ✅ | ✅ | ✅ |
| Domain knowledge extraction | ❌ | ✅ | ✅ | ✅ |
| Manual copy-paste workflow | 2 pastes | 2 pastes | ❌ None | ❌ None |
| Agent runs loop autonomously | ❌ | ❌ | ✅ | ✅ |
| Session tagging (no double-processing) | ❌ | ❌ | ❌ | ✅ |
| Retroactive batch mode | ❌ | ❌ | ❌ | ✅ |
| Retroactive single mode | ❌ | ❌ | ❌ | ✅ |
| Trivial session auto-skip | ❌ | ❌ | ❌ | ✅ |

---

## Roadmap (beyond PoC)

- **Engram daemon / hook**: auto-trigger when a Claude Code session ends (currently blocked by hook limitations — hooks can run shell but can't prompt the LLM)
- **Preference drift detection**: compare User Preferences across versions to spot evolving taste
- **Domain model maturation**: track domain correction frequency — a spike after stability signals a new feature area or business pivot
- **Cross-project preferences**: extract user-global preferences (code style, verbosity) into `~/.claude/global_preferences.md`, keep domain knowledge project-scoped
- **Feedback loop**: compare memory entries against actual usage — prune entries with zero utility over N sessions
- **Meta-learning**: track acceptance rates over time via `engram_log.txt` — rising rates mean the system is working
- **Contradiction escalation**: when static signals detect memory-vs-code contradiction AND dynamic signals show Claude acted on the stale memory, flag as critical
