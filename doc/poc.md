# Edge-Augmented Memory Consolidation — PoC Prompt for Claude Code

## How to Use

Copy the prompt below and paste it into your Claude Code session (VS Code extension or terminal) at the end of a working session. It will:

1. Ask Claude to analyze the current session and generate local extraction scripts
2. You run the scripts locally (they produce a structured JSON file)
3. You paste the second prompt with the JSON results, and Claude produces an updated CLAUDE.md

Over time, you can add this as a slash command or alias for convenience.

---

## PROMPT 1 — Generate Extraction Scripts

```
I want you to help me consolidate what happened in this session into my persistent project memory. But instead of doing all the analysis yourself, I want you to generate a set of lightweight shell scripts that I will run locally. These scripts will extract structured signals from my project, and then I'll give you the results so you can produce a clean memory update.

Here is my current persistent memory (if it exists):
<memory>
[contents of .claude/CLAUDE.md — if the file exists, paste it here, or ask Claude to read it]
</memory>

Based on what happened in this session and my current memory state, generate a single shell script called `extract_signals.sh` that I can run from my project root. The script should:

1. **Session diff analysis**: Use `git diff` and `git log` to identify what files changed, what was added/removed, and commit messages from today's work.

2. **Frequency analysis**: Count which files and directories were touched most across recent git history (last 5 days). Output a ranked list.

3. **Error pattern extraction**: Grep recent terminal/shell history or log files for common error patterns, failed commands, or repeated retries. If no shell history is accessible, scan git diffs for removed debug/print statements as a proxy.

4. **Convention detection**: Scan the codebase for patterns that suggest conventions — consistent naming patterns in recent files, recurring import structures, repeated config patterns. Use simple grep/awk, not AI.

5. **Contradiction detection**: Compare the current CLAUDE.md (if it exists) against the current state of the codebase. For example, if CLAUDE.md says "we use Jest for testing" but the recent diffs show vitest imports, flag that as a potential contradiction. Use simple string matching.

6. **Dependency changes**: Check if package.json, requirements.txt, Cargo.toml, go.mod (or equivalent) changed. If so, summarize what was added or removed.

7. **TODO/FIXME scan**: Grep for TODO, FIXME, HACK, XXX comments added in recent diffs.

Requirements for the script:
- Output everything as a single JSON object written to `.claude/session_signals.json`
- Each section should be a key in the JSON with structured data
- Use only standard Unix tools (git, grep, awk, sed, find, wc, jq if available, otherwise raw echo to build JSON)
- Handle missing tools or empty results gracefully (output empty arrays, don't crash)
- Keep it portable across macOS and Linux
- Add a timestamp and the current git branch to the output
- The script should complete in under 10 seconds for a typical project

Output ONLY the script, no explanation. I'll run it and come back with the results.
```

---

## PROMPT 2 — Synthesize Memory Update

After running `extract_signals.sh`, use this prompt:

```
Here are the structured signals extracted from my local project by running deterministic scripts after our session:

<signals>
[paste contents of .claude/session_signals.json here]
</signals>

And here is my current persistent memory:
<current_memory>
[paste contents of .claude/CLAUDE.md here, or "No existing memory file."]
</current_memory>

Based on these signals, produce an updated version of my CLAUDE.md file. Follow these rules:

1. **Preserve high-value existing entries** — don't drop things that are still true and useful.
2. **Update contradicted facts** — if the signals show that a convention or decision has changed, update the memory to reflect reality. Add a note like "(updated YYYY-MM-DD)" for significant changes.
3. **Add new learnings** — incorporate new patterns, conventions, decisions, or architectural facts discovered in this session.
4. **Prune stale entries** — if something hasn't been relevant in recent signals and seems outdated, remove it.
5. **Use absolute dates** — never write "yesterday" or "recently." Use actual dates.
6. **Keep it under 150 lines** — this file needs to fit comfortably in a context window. Be concise. Every line should earn its place.
7. **Structure clearly** — use sections like: Project Overview, Architecture Decisions, Conventions, Active Work, Known Issues, Key Dependencies, Recent Changes.
8. **Optimize for future Claude sessions** — write entries the way that would be most useful for an AI assistant picking up the project cold. Prioritize actionable context over historical narrative.

Output ONLY the updated CLAUDE.md content, ready to be written to the file.
```

---

## Automation Tips

### Make it a one-liner alias
```bash
# Add to .bashrc / .zshrc
alias consolidate='bash .claude/extract_signals.sh && echo "Signals extracted to .claude/session_signals.json — paste Prompt 2 into Claude"'
```

### Evolve the extraction over time
After a few cycles, you'll notice which signal categories are most useful for YOUR workflow. Edit `extract_signals.sh` to add project-specific extractions:
- If you work with APIs: scan for new route definitions
- If you do data work: track schema changes
- If you manage infra: diff terraform/docker files

### Track extraction quality
Keep a simple log:
```bash
# Append to .claude/consolidation_log.txt after each cycle
echo "$(date -Iseconds) | signals_size=$(wc -c < .claude/session_signals.json) | memory_lines=$(wc -l < .claude/CLAUDE.md)" >> .claude/consolidation_log.txt
```

If your signals file keeps growing but your memory quality doesn't improve, your extractions are producing noise. Prune the script.

---

## What This PoC Validates

- **Cost reduction**: Two focused API calls instead of one large "review everything" call
- **Better signal quality**: The LLM reasons over structured data, not raw transcripts
- **User control**: You see exactly what data is extracted and can tune the scripts
- **No infrastructure required**: Just shell scripts and two copy-paste prompts

The next evolution would be having Claude generate increasingly personalized extraction scripts based on what proved useful in past consolidation cycles — but that's beyond PoC scope.