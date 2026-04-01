# engram

Edge-Augmented Memory Consolidation for AI Coding Agents -- an ideation by [Wisecorp](https://wisecorp.tn), a technology startup building FPGA-accelerated solutions for modern hybrid applications.

## Why

AI coding agents like Claude Code are starting to remember things across sessions. Anthropic's leaked internal system ("Dream", exposed via the Claude Code source map incident on March 31, 2026) shows one approach: a subagent that reviews past sessions, deduplicates notes, detects contradictions, and prunes stale entries.

The problem? Every phase runs through the LLM. You're paying inference rates for string matching, diffing, and counting -- work a shell script handles in milliseconds at zero cost.

## What

engram splits memory consolidation into what requires reasoning and what requires processing:

| Layer | Where | Cost |
|-------|-------|------|
| **Query Generation** -- LLM produces extraction instructions (shell scripts) based on session context and current memory | API | ~few hundred tokens |
| **Signal Extraction** -- local machine runs the scripts: git diffs, frequency analysis, error patterns, convention detection, contradiction checks | Edge | Zero |
| **Synthesis** -- LLM receives structured signals and produces an updated memory file | API | Focused, minimal noise |

Two small API calls. Arbitrarily complex local processing in between.

## Status

This is currently a **proof of concept** -- two copy-paste prompts for Claude Code that validate the core idea. No infrastructure required.

## Quick Start

1. At session end, paste **Prompt 1** ([doc/poc.md](doc/poc.md)) into Claude Code -- it generates `extract_signals.sh`
2. Run locally: `bash extract_signals.sh`
3. Paste **Prompt 2** with the JSON output -- Claude produces an updated `CLAUDE.md`

## Read More

- [Design rationale](doc/article.md) -- full breakdown of the three-layer architecture and the meta-learning feedback loop
- [PoC prompts](doc/poc.md) -- ready-to-use prompts and automation tips

## License

See [LICENSE](LICENSE).
