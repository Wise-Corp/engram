# Engram — Edge-Augmented Memory Consolidation for AI Coding Agents
## PoC v3 — In-Session Automation

### What Changed in v3

v2 required three manual copy-paste steps: paste Prompt 1 → run scripts manually → paste Prompt 2 with JSON.

v3 eliminates all copy-paste. The AI agent executes the entire loop in a single conversation turn:

1. **Agent generates** the extraction scripts (tailored to the current session and project)
2. **Agent runs** the scripts via its shell tool
3. **Agent reads** the resulting signals JSON
4. **Agent synthesizes** the memory update and writes it directly

The user just says **"engram"** and the loop runs end-to-end.

### How It Works

Store the Engram instructions in your project's persistent memory (CLAUDE.md, memory files, or equivalent). When triggered, the agent:

1. Reads its current persistent memory
2. Processes Prompt 1 internally → generates `extract_static.sh` and `extract_dynamic.sh`
3. Writes both scripts to `.claude/` and executes them → `.claude/session_signals.json`
4. Reads the signals JSON
5. Processes Prompt 2 internally → updates the persistent memory files

The scripts are **ephemeral** — regenerated fresh each time, adapted to the current project state and session context. This is by design: the LLM decides what to extract based on what actually happened in the session.

### Setup

Add the following to your project's persistent memory system (e.g. `CLAUDE.md`, or a dedicated memory file if using Claude Code's auto-memory):

```markdown
## Engram — Memory Consolidation

When the user says "engram" or "run engram", execute the full memory consolidation loop:

1. **Generate extraction scripts** — Process Prompt 1 (below) using the current session
   context and existing memory. Write two shell scripts to `.claude/`.
2. **Run extraction** — Execute both scripts. Read `.claude/session_signals.json`.
3. **Synthesize** — Process Prompt 2 (below) with the signals and current memory.
   Update memory files where warranted. Skip updates if signals are too thin.
```

Then include the two prompts (below) in the same file, so the agent has them available.

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

Extracts deterministic signals from the codebase and git history:

1. **Session diff analysis**: `git diff` / `git log` for today's file changes, commits, additions, deletions
2. **Frequency analysis**: most-touched files and directories over the last 5 days
3. **Error pattern extraction**: removed debug statements, recent log errors
4. **Convention detection**: import patterns, naming conventions, endpoint patterns in changed files
5. **Contradiction detection**: compare persistent memory assertions against actual codebase state (package.json vs stated deps, test framework references vs actual test files, stated patterns vs code reality)
6. **Dependency changes**: diff of requirements.txt / package.json / Cargo.toml etc.
7. **TODO/FIXME scan**: new TODOs added in this session, existing ones in codebase

### Script 2: `extract_dynamic.sh` — Interaction & Preference Signals

Analyzes the Claude Code session transcript to extract behavioral signals.

**Finding the transcript:**
- Claude Code stores transcripts at `~/.claude/projects/<project-path>/<uuid>.jsonl`
  where `<project-path>` is the absolute path with slashes replaced by dashes
  (e.g. `-home-user-myproject`)
- Pick the most recently modified `.jsonl` file (by `ls -t`, NOT `find -printf` which is GNU-only)
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
- Output merged into `.claude/session_signals.json`
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
Here are the structured signals extracted from my project after our session.

<signals>
{{CONTENTS_OF_.claude/session_signals.json}}
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

## v3 Automation — Memory File Template

For Claude Code projects using auto-memory, save this as a reference memory file.
The agent will find it automatically when the user says "engram":

```markdown
---
name: Engram — automated memory consolidation
description: When user says "engram", run the full 3-step memory consolidation loop
type: reference
---

When the user says "engram", execute the full Engram loop:

1. Generate fresh extract_static.sh and extract_dynamic.sh (Prompt 1 from Engram PoC)
2. Write to .claude/, execute both, read .claude/session_signals.json
3. Synthesize memory updates (Prompt 2 from Engram PoC)
4. Update memory files where warranted; skip if signals are too thin

Source: https://github.com/Wise-Corp/engram
```

For projects using a single `CLAUDE.md`, append the trigger instruction directly:

```markdown
## Memory Consolidation
When I say "engram", run the Engram loop: generate extraction scripts → run them →
synthesize updates. See https://github.com/Wise-Corp/engram for full prompts.
```

---

## What Changed Across Versions

| | v1 | v2 | v3 |
|---|---|---|---|
| Static signals (git, deps, conventions) | ✅ | ✅ | ✅ |
| Dynamic signals (transcript analysis) | ❌ | ✅ | ✅ |
| User preference extraction | ❌ | ✅ | ✅ |
| Domain knowledge extraction | ❌ | ✅ | ✅ |
| Manual copy-paste workflow | 2 pastes | 2 pastes | ❌ None |
| Scripts generated fresh per session | ✅ | ✅ | ✅ |
| Agent runs scripts autonomously | ❌ | ❌ | ✅ |
| Agent synthesizes autonomously | ❌ | ❌ | ✅ |
| Single-command trigger ("engram") | ❌ | ❌ | ✅ |

The static signals tell Claude what the project **looks like**.
The dynamic user signals tell Claude how the developer **wants to work**.
The dynamic domain signals tell Claude what the project **means**.
v3 closes the loop: the agent does all three without human intermediation.

---

## Roadmap (beyond PoC)

- **Engram daemon / hook**: auto-trigger when a Claude Code session ends (currently blocked by hook limitations — hooks can run shell but can't prompt the LLM)
- **Preference drift detection**: compare User Preferences across versions to spot evolving taste
- **Domain model maturation**: track domain correction frequency — a spike after stability signals a new feature area or business pivot
- **Cross-project preferences**: extract user-global preferences (code style, verbosity) into `~/.claude/global_preferences.md`, keep domain knowledge project-scoped
- **Feedback loop**: compare memory entries against actual usage — prune entries with zero utility over N sessions
- **Meta-learning**: track acceptance rates over time via `engram_log.txt` — rising rates mean the system is working
- **Contradiction escalation**: when static signals detect memory-vs-code contradiction AND dynamic signals show Claude acted on the stale memory, flag as critical
