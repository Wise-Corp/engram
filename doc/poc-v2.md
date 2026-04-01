# Engram — Edge-Augmented Memory Consolidation for AI Coding Agents
## PoC v2 — With Transcript-Derived Dynamic Signals

### How to Use

This version adds **dynamic signals** extracted from the actual session transcript,
capturing two dimensions that static code analysis can never reach:
- **User preferences**: what you accepted, rejected, corrected — how you like to work
- **Domain knowledge**: business terminology, rules, workflows, and entity relationships that Claude keeps misunderstanding

**Workflow:**
1. End of session → paste **Prompt 1** into Claude Code
2. Claude generates two scripts: `extract_static.sh` (project signals) and `extract_dynamic.sh` (transcript signals)
3. Run both locally → they produce `.claude/session_signals.json`
4. Paste **Prompt 2** with the JSON results → Claude produces updated `CLAUDE.md`

---

## PROMPT 1 — Generate Extraction Scripts

```
I want to consolidate what happened in this session into persistent project memory using the Engram approach: you generate extraction scripts, I run them locally, then you synthesize the results.

Here is my current persistent memory:
<memory>
[contents of .claude/CLAUDE.md — if it exists]
</memory>

Generate TWO shell scripts that I will run from my project root.

### Script 1: `extract_static.sh` — Project State Signals

This script extracts deterministic signals from the codebase and git history:

1. **Session diff analysis**: `git diff` / `git log` for today's file changes, commits, additions, deletions
2. **Frequency analysis**: most-touched files and directories over the last 5 days
3. **Error pattern extraction**: removed debug statements, recent log errors
4. **Convention detection**: import patterns, naming conventions, endpoint patterns in changed files
5. **Contradiction detection**: compare CLAUDE.md assertions against actual codebase state (package.json vs stated deps, test framework references vs actual test files, stated patterns vs code reality)
6. **Dependency changes**: diff of requirements.txt / package.json / Cargo.toml etc.
7. **TODO/FIXME scan**: new TODOs added in this session, existing ones in codebase

### Script 2: `extract_dynamic.sh` — Interaction & Preference Signals

This script analyzes the Claude Code session transcript to extract behavioral signals.
The transcript location varies — check these paths in order:
- `~/.claude/projects/*/sessions/*/transcript.jsonl` (Claude Code terminal)
- The most recently modified `.jsonl` file under `~/.claude/`
- If VSCode: check `~/.config/Code/User/globalStorage/` for relevant logs

The script should parse the transcript (JSONL format where each line is a JSON object with `role`, `content`, and optionally `tool_use` fields) and extract signals in TWO dimensions:

### Dimension A: User Preferences (how the developer works)

1. **Acceptance rate**: Count user messages that follow an assistant turn and classify them:
   - ACCEPTED: messages matching patterns like "yes", "do it", "looks good", "go ahead", "perfect", "great", "ok", "👍", "ship it", "lgtm", or messages that simply ask for the next task without contesting the previous output
   - REJECTED: messages matching "no", "don't", "stop", "wrong", "not what I", "actually", "wait", "instead", "revert", "undo"
   - MODIFIED: messages matching "but change", "almost but", "close but", "can you also", "one thing", "tweak", "adjust"
   - Output: counts and percentages for each category

2. **Rejection context**: For each REJECTED or MODIFIED response, extract:
   - The user's correction message (first 200 chars)
   - What type of assistant action preceded it (file_edit, bash_command, explanation, architecture_suggestion) — detectable from tool_use blocks
   - Output: list of {action_type, user_feedback_snippet} pairs

3. **Coding preference keywords**: Extract recurring words/phrases from REJECTED and MODIFIED messages that relate to code style/structure:
   - Run word frequency analysis on correction messages
   - Filter out stop words and domain nouns (see Dimension B)
   - Look for style indicators: "simpler", "verbose", "split", "merge", "inline", "separate", "smaller", "cleaner"
   - Output: top 15 most frequent style-related words in corrections

4. **Iteration depth**: For sequences where the same file is edited multiple times consecutively:
   - Count how many edits per file before the user moves on
   - Flag files with 3+ consecutive edits as "high-friction" areas
   - Output: list of {file, edit_count, final_accepted: bool}

5. **Prompt complexity trend**: Measure average user message length (in words) across the session in quartiles:
   - Q1 (first 25% of messages), Q2, Q3, Q4
   - Rising length suggests Claude is losing context and the user is compensating
   - Output: {q1_avg_words, q2_avg_words, q3_avg_words, q4_avg_words}

6. **Explicit style preferences**: Grep for preference language about code style:
   - "I prefer", "always use", "never use", "don't like", "let's stick with"
   - Output: list of extracted preference statements

### Dimension B: Domain Knowledge (what the project is about)

7. **Terminology corrections**: Detect when the user corrects Claude's understanding of a domain concept. Look for patterns:
   - "X is not Y", "X and Y are different", "that's not a X, it's a Y"
   - "when I say X I mean", "X here means", "in our context X is"
   - "no, X refers to", "X is actually", "don't confuse X with Y"
   - Output: list of {wrong_term, correct_term, user_explanation_snippet} triples

8. **Business rule statements**: Extract when the user explains WHY something must work a certain way. Detect:
   - "because", "the rule is", "we have to", "we can't because", "regulation requires", "the client expects", "the workflow requires"
   - "it must go through", "only after", "never before", "always requires"
   - Capture the full sentence containing the rule (up to 300 chars)
   - Output: list of {rule_statement, context_snippet}

9. **Workflow/state corrections**: Detect when Claude proposes an action that violates a process flow:
   - User responses containing: "first we need to", "it has to go through", "before that", "after X then Y", "the order is", "the flow is", "can't skip"
   - Output: list of {stated_workflow_constraint}

10. **Entity relationship clarifications**: Detect when the user clarifies how domain objects relate:
    - "X belongs to Y", "X has many Y", "X is a type of Y", "X is not the same as Y"
    - "a X can have multiple Y", "every X must have a Y", "X is optional on Y"
    - Output: list of {entity_a, entity_b, relationship_description}

11. **Repeated explanations**: Detect when the user writes long (>50 word) messages that are explanatory rather than directive — these often indicate Claude misunderstood a domain concept:
    - Heuristic: user message >50 words AND follows a REJECTED/MODIFIED classification AND contains few code-related tokens (no backticks, no file paths, low symbol density)
    - These are domain lessons the user is teaching Claude
    - Output: list of {explanation_snippet (first 300 chars), preceding_topic}

12. **Abandoned approaches with domain reason**: Extend the general "abandoned approaches" detection — when the user pivots AND gives a domain reason:
    - "that won't work because [business reason]", "the problem is [domain constraint]"
    - Output: list of {abandoned_approach_snippet, domain_reason_snippet}

13. **Domain vocabulary frequency**: Across ALL user messages (not just corrections), extract nouns and noun phrases that appear 3+ times and are NOT common programming terms. These form the project's domain lexicon:
    - Filter out: file, function, class, error, bug, code, test, commit, branch, deploy, API, endpoint, route, model, database, query, etc.
    - What remains are likely domain terms: "invoice", "quotation", "client", "warehouse", "shipment", "approval", etc.
    - Output: ranked list of {term, frequency}

Requirements for both scripts:
- Output merged into a single `.claude/session_signals.json` (extract_static writes first, extract_dynamic merges in)
- Use only standard Unix tools + python3 for JSON parsing where needed
- Handle missing transcripts gracefully (output empty sections, don't fail)
- Complete in under 15 seconds total
- Portable across macOS and Linux (no GNU-only flags like `find -printf`, no `date -d`; use compatible alternatives)
- **Critical JSON safety**: NEVER interpolate shell variables containing JSON into Python heredocs via `json.loads('''$VAR''')` or similar patterns — this breaks when the content contains quotes. Instead, write intermediate JSON fragments to temp files and have the final Python assembler read those files. Use a quoted heredoc (`<< 'EOF'`) for the final assembly script so bash performs no expansion inside it. Each fragment file should be loaded with a `safe_load()` helper that catches `JSONDecodeError` and returns an empty fallback rather than crashing the whole assembly.
- Use a `trap` to clean up any temp files on exit

Output ONLY the two scripts, clearly separated. No explanations.
```

---

## PROMPT 2 — Synthesize Memory Update

```
Here are the structured signals extracted from my project after our session.
They include both static signals (codebase state) and dynamic signals (interaction patterns from the transcript).

<signals>
[paste contents of .claude/session_signals.json here]
</signals>

Current persistent memory:
<current_memory>
[paste contents of .claude/CLAUDE.md here, or "No existing memory file."]
</current_memory>

Produce an updated CLAUDE.md. Follow these rules:

## Content rules
1. Preserve high-value existing entries that are still true
2. Update contradicted facts with a "(updated YYYY-MM-DD)" note
3. Add new learnings from static signals (conventions, architecture, deps)
4. Prune stale entries not reflected in recent signals
5. Use absolute dates, never "recently" or "yesterday"
6. Keep under 150 lines — every line must earn its place

## Structure
Use these sections:
- **Project Overview**: what this project is and does
- **Domain Model**: business entities, their relationships, and how they differ from each other. Written to prevent an AI from confusing related-but-distinct concepts. E.g.: "A Quotation is NOT an Order. A Quotation becomes an Order only after client approval. They are separate database tables with separate lifecycles."
- **Business Rules**: hard constraints that must never be violated, extracted from domain corrections. E.g.: "Invoices require soft-delete for audit compliance — never use hard delete." "A shipment cannot be created without a validated order."
- **Workflow & State Machines**: process flows with explicit ordering constraints. E.g.: "Order lifecycle: draft → submitted → validated → shipped → invoiced. Transitions cannot skip states."
- **Domain Vocabulary**: key terms and their project-specific meanings, especially where they differ from common usage or where two terms are easily confused
- **Architecture Decisions**: key technical choices and rationale
- **Conventions**: coding patterns, naming, file organization
- **User Preferences**: how the developer likes to work with Claude — code style, response style, commit granularity, friction areas
- **Active Work**: current focus areas based on recent signals
- **Known Issues**: bugs, tech debt, friction points
- **Key Dependencies**: current stack
- **Recent Changes**: significant changes from last session(s)

## Critical: the Domain Knowledge sections (Domain Model, Business Rules, Workflow, Vocabulary)
These sections prevent the most costly type of AI error: misunderstanding the business.
Based on the dynamic signals from Dimension B:
- Every terminology correction becomes a vocabulary entry or a disambiguation note in Domain Model
- Every business rule statement becomes an entry in Business Rules, phrased as a hard constraint with its reason
- Every workflow correction becomes an explicit state/ordering constraint
- Every repeated explanation signals a concept that Claude keeps getting wrong — give it extra emphasis and clarity
- Phrase these as RULES, not descriptions. Future Claude sessions should read these as non-negotiable constraints, e.g.:
  "NEVER confuse Client (company entity) with User (login account). They have a many-to-many relationship."
  "RULE: An invoice can only be generated from a validated order. No exceptions."
  "WORKFLOW: Payment must be recorded BEFORE delivery confirmation is sent."

## Critical: the User Preferences section
Based on the dynamic signals from Dimension A:
- If the acceptance rate is high for certain action types, note those as strengths
- If certain patterns are consistently rejected, note them as anti-patterns FOR THIS USER
- Extract any explicitly stated preferences verbatim
- If prompt complexity rises across the session, note what topics caused confusion
- If certain files had high iteration depth, note what the friction was about
- Phrase these as actionable instructions for a future Claude session, e.g.:
  "User prefers small, focused commits over large batches"
  "User rejects class-based patterns — always use functions"
  "User wants explicit error handling, not bare try/except"
  "When editing templates, propose changes in small increments — high iteration area"

## Output
Output ONLY the updated CLAUDE.md content, ready to write to file.
```

---

## Automation

```bash
# Add to .bashrc / .zshrc
alias engram='bash .claude/extract_static.sh && bash .claude/extract_dynamic.sh && echo "✅ Signals ready in .claude/session_signals.json"'
```

### After running engram:
1. Open Claude Code
2. Paste Prompt 2 with the JSON contents
3. Save the output as `.claude/CLAUDE.md`

### Track quality over time:
```bash
echo "$(date -Iseconds) | signals=$(wc -c < .claude/session_signals.json) | memory=$(wc -l < .claude/CLAUDE.md) | acceptance=$(python3 -c "import json; d=json.load(open('.claude/session_signals.json')); print(d.get('interaction',{}).get('acceptance_rate',{}).get('accepted_pct','N/A'))")" >> .claude/engram_log.txt
```

---

## What v2 Adds Over v1

| | v1 (static only) | v2 (static + dynamic) |
|---|---|---|
| Knows what changed in code | ✅ | ✅ |
| Knows what conventions exist | ✅ | ✅ |
| Knows what you accepted/rejected | ❌ | ✅ |
| Learns your coding preferences | ❌ | ✅ |
| Detects communication friction | ❌ | ✅ |
| Captures business terminology | ❌ | ✅ |
| Records business rules & constraints | ❌ | ✅ |
| Maps workflow/state machines | ❌ | ✅ |
| Prevents entity confusion | ❌ | ✅ |
| Detects repeated explanations | ❌ | ✅ |

The static signals tell Claude what the project **looks like**.
The dynamic user signals tell Claude how the developer **wants to work**.
The dynamic domain signals tell Claude what the project **means**.
All three together produce memory that prevents both coding friction and business logic errors.

---

## Roadmap ideas (beyond PoC)

- **Engram daemon**: auto-trigger extraction when a Claude Code session ends
- **Preference drift detection**: compare User Preferences across versions to spot evolving taste
- **Domain model maturation**: track how the Domain Model section evolves — early sessions will produce many corrections, mature projects should produce few. A spike in domain corrections after a period of stability signals a new feature area or business pivot.
- **Cross-project preferences**: some preferences are user-global, not project-specific (code style, verbosity, commit granularity). Extract these into a `~/.claude/global_preferences.md`. Domain knowledge, by contrast, is always project-scoped.
- **Domain ontology graph**: over many sessions, the entity relationships and business rules form a graph. Visualizing it could reveal gaps — entities Claude has never been corrected on (either it understands them or they've never come up).
- **Feedback loop**: after each session, compare what was in CLAUDE.md against what actually got used/referenced. Prune entries with zero utility over N sessions.
- **Meta-learning**: use the engram_log.txt to detect whether memory quality is improving. If acceptance rates rise over time, the system is working. If domain corrections decrease, the business model is being internalized. If not, the extraction scripts need tuning.
- **Contradiction escalation**: when the static script detects a codebase-vs-memory contradiction AND the dynamic script shows Claude acted on the stale memory during the session, flag this as a critical memory failure to fix immediately.