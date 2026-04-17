# Prompt 1 — Generate Extraction Scripts

This prompt is used by the engram trigger to generate fresh extraction scripts each session. The agent sends this to the LLM, which produces two shell scripts adapted to the current project.

---

```
I want to consolidate what happened in this session into persistent project memory.
Generate TWO shell scripts adapted to this project.

Here is my current persistent memory:
<memory>
{{CURRENT_MEMORY}}
</memory>

### Shared requirements (apply to BOTH scripts)
- Output to `.claude/session_signals.json` (or `.claude/session_signals_<uuid_prefix>.json`
  when `ENGRAM_BATCH_ID` env var is set, to avoid overwriting between batch iterations)
- Use standard Unix tools + python3 for JSON
- Handle missing data gracefully
- Portable macOS/Linux (no GNU-only flags)
- JSON safety: never interpolate shell vars into Python heredocs — use temp files
  or pass via sys.argv / environment. Shell variables containing newlines or quotes
  will break Python string literals if interpolated directly.
- `trap` to clean up temp files

### Script 1: `extract_static.sh` — Project State Signals

Extracts deterministic signals from the codebase and git history.

**Date range**: accept `ENGRAM_SINCE` and `ENGRAM_UNTIL` environment variables.
Default `ENGRAM_SINCE` to today, `ENGRAM_UNTIL` to empty (open-ended).
Replace all `git log --since` calls with `--since="$SINCE" ${UNTIL:+--until="$UNTIL"}`.

**Implementation pattern**: Write each git/shell output to a temp file, then read
all temp files from a Python heredoc to build the JSON. Do NOT use `'''$VAR'''`
or f-string interpolation with shell variables.

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

**Additional Script 2 requirements:**
- Handle missing transcripts gracefully

**v4.1 quality rules — apply these to the generated scripts:**

1. **System content stripping**: Before analyzing transcript messages, strip:
   - Lines matching `\[Image:.*\]` (screenshot metadata)
   - Lines matching `Multiply coordinates` or `displayed at .* resolution`
   - Content inside XML-style tags with system prefixes (`<system-`, `<ide_`, `<command-`)
   - Session continuation / context-restore summaries

2. **Context-aware acceptance/rejection**: Do NOT use bare keyword matching.
   - Require rejection keywords at sentence start or after punctuation
   - Exclude "no" when followed by: "need", "worries", "problem", "idea", "way"
   - Exclude "actually" when followed by: "the reason", "it's because", "I think"
   - For simple accept/reject classification, prefer short messages (< 50 words)

3. **Domain vocabulary noise filtering**: Expand the stop/noise word list to exclude:
   - IDE/UI terms: `plugin`, `marketplace`, `settings`, `properties`, `extension`,
     `workspace`, `panel`, `sidebar`, `toolbar`, `palette`, `readonly`, `proceed`,
     `rejected`, `written`, `completed`, `matches`
   - Image artifacts: `image`, `original`, `coordinates`, `multiply`, `displayed`,
     `resolution`, `screenshot`, `pixel`
   - System terms: `successfully`, `output`, `current`, `context`, `updated`, `related`

4. **Business rule precision**: A bare "because" is NOT enough for signal 8.
   Require at least one of:
   - Constraint keyword: "must", "can't", "never", "always", "only after"
   - Rule framing: "the rule is", "we have to", "it must go through"
   - Domain entity + constraint in the same sentence
   - Exclude debugging context: "because it was failing", "because the error"

5. **Terminology correction precision**: For signal 7:
   - Require the corrected term to be a noun/entity, not a common verb
   - Only match user messages (role: "user"), not assistant self-corrections
   - Exclude matches where both sides of "is not" are common programming terms
```
