# Engram — Edge-Augmented Memory Consolidation for AI Coding Agents

## v5.3 — Script Reliability & New-Project Bootstrapping

### What Changed in v5.3

Three fixes surfaced during a live retroactive engramming session (2026-05-10):

### Heredoc Quoting Rule (Prompt 1)

Python regex patterns contain backticks, which bash interprets as command substitution inside unquoted heredocs (`<< PYEOF`). This caused `extract_dynamic.sh` to crash with a bash syntax error before any signals could be extracted.

The Shared Requirements section in Prompt 1 now explicitly requires quoted heredoc delimiters (`<< 'EOF'`, never `<< EOF`) when the body contains backticks, `$`, or backslashes — which Python regexes always do. Shell values must be passed via `os.environ` or `sys.argv`, never via heredoc interpolation.

### New-Project Bootstrapping (Prompt 2)

Rule 6 ("thin signals → skip update") left brand-new projects with zero memory files. When MEMORY.md contained only the engram trigger, synthesis would skip entirely even though baseline memory was needed for future sessions to build on.

Added an exception: when fewer than 3 memory files exist, bootstrap baseline project memory (project-overview, architecture, conventions) from available static signals and direct codebase inspection. An empty memory is worse than a thin but accurate one.

### Reconciliation Script Instructions (engram.md)

Step 1 of the trigger ("Reconcile `.claude/engram_manifest.json`") was a one-liner with no implementation detail. LLMs reconciled manually by inspecting files rather than generating a proper reconciliation script.

Expanded with explicit instructions: generate and run a small reconciliation script that scans all `.jsonl` transcript files, counts user+assistant messages, reads first/last timestamps, and adds missing sessions to the manifest with `engrammed: false`.

---

## v5.2 — Auto-Update Check

### What Changed in v5.2

Running "engram" now checks GitHub for a newer version of `engram.md` before starting work. If a difference is detected, the user is asked whether to update. This is a lightweight `curl` + `diff` — if there's no network, the check is silently skipped.

This means users no longer need to remember to say "update engram" — the trigger keeps itself current.

---

## v5.1 — Auto-Cleanup & Signal Quality

### What Changed in v5.1

v5 left generated scripts and signal JSON files in `.claude/` after engramming. Over multiple sessions, this accumulated stale artifacts. v5.1 adds automatic cleanup and bakes the v4.1 signal quality fixes into Prompt 1.

### Auto-Cleanup

After synthesis and manifest tagging, engram now deletes all ephemeral artifacts:

```bash
rm -f .claude/extract_static.sh .claude/extract_dynamic.sh .claude/session_signals*.json
```

Only the manifest persists between runs. Scripts are regenerated each time, and signal JSON files have already been consumed by synthesis.

### v4.1 Quality Fixes Applied to Prompt 1

The v4.1 changelog documented 5 signal quality issues found during 10-session retroactive testing on wisecorp-treso. These were guidance notes — v5.1 bakes them directly into `prompt1-extraction.md` as mandatory rules for generated scripts:

1. **System content stripping** — expanded filter for IDE tags, screenshot metadata, session continuations
2. **Context-aware acceptance/rejection** — no more false positives from "no" in "for now" or "actually" in explanations
3. **Domain vocabulary noise filtering** — expanded noise list excludes IDE/UI/system terms
4. **Business rule precision** — requires compound pattern (constraint keyword + domain entity), not bare "because"
5. **Terminology correction precision** — requires noun/entity terms, user role only, excludes programming term pairs

---

## v5 — One-Line Deployment

### What Changed in v5

v4.2 restructured the repo for user-friendliness but still required manual file copying. v5 makes deployment a single natural-language instruction:

```
deploy engram from https://github.com/Wise-Corp/engram
```

Claude fetches `engram.md` from GitHub, copies it into `.claude/memory/`, sets up the memory index, and confirms. No cloning, no manual steps.

The `engram.md` trigger file now contains two triggers:
- **"engram"** → run the consolidation loop (existing behavior)
- **"deploy/update engram"** → fetch latest from GitHub and install/update

Updating is the same command — Claude fetches the latest version and overwrites.

---

## v4 — Session Tagging & Retroactive Engramming

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

### 5. Terminology Correction False Positives

**Problem**: Signal 7 (terminology corrections — "X is not Y", "when I say X I mean") triggers on system-injected text (config skill prompts, session continuations) and on normal explanatory phrases where the user isn't actually correcting terminology.

**Fix in Prompt 1**: Two-layer fix:
- **Source filtering** (covered by fix 1): stripping system content removes the main source of false positives
- **Pattern precision**: tighten the terminology correction patterns themselves:
  - Require the corrected term to appear as a noun/entity, not a verb or common word
  - Require the message to be a user message (role: "user"), not assistant self-correction
  - Exclude matches where both sides of "is not" are common programming terms (e.g. "this is not a bug" is not a terminology correction)
  - Weight matches higher when the user explicitly names a domain entity on at least one side of the correction

---

## Prompts

The two prompts that drive engram have been extracted into standalone files for easy reading and customization:

- **[Prompt 1 — Extraction](../prompts/prompt1-extraction.md)**: generates the two shell scripts adapted to the current project
- **[Prompt 2 — Synthesis](../prompts/prompt2-synthesis.md)**: takes structured signals and produces memory updates

The memory trigger file is at **[engram.md](../engram.md)** — this is the file users copy into their project's `.claude/memory/` directory. See [install.md](../install.md) for setup instructions.

---

## What Changed Across Versions

| | v1 | v2 | v3 | v4 | v5 | v5.1 | v5.2 | v5.3 |
|---|---|---|---|---|---|---|---|---|
| Static signals (git, deps, conventions) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Dynamic signals (transcript analysis) | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| User preference extraction | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Domain knowledge extraction | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Manual copy-paste workflow | 2 pastes | 2 pastes | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| Agent runs loop autonomously | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Session tagging (no double-processing) | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Retroactive batch mode | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Retroactive single mode | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Trivial session auto-skip | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ |
| One-line deployment via Claude | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Self-updating trigger file | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Auto-cleanup of artifacts | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| v4.1 quality fixes in prompt | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Auto-update check on run | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |
| Heredoc-safe script generation | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| New-project memory bootstrapping | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Reconciliation script generation | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## Roadmap (beyond PoC)

- **Engram daemon / hook**: auto-trigger when a Claude Code session ends (currently blocked by hook limitations — hooks can run shell but can't prompt the LLM)
- **Preference drift detection**: compare User Preferences across versions to spot evolving taste
- **Domain model maturation**: track domain correction frequency — a spike after stability signals a new feature area or business pivot
- **Cross-project preferences**: extract user-global preferences (code style, verbosity) into `~/.claude/global_preferences.md`, keep domain knowledge project-scoped
- **Feedback loop**: compare memory entries against actual usage — prune entries with zero utility over N sessions
- **Meta-learning**: track acceptance rates over time via `engram_log.txt` — rising rates mean the system is working
- **Contradiction escalation**: when static signals detect memory-vs-code contradiction AND dynamic signals show Claude acted on the stale memory, flag as critical
