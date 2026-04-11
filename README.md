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

**PoC v4** -- fully automated with session tagging and retroactive engramming. Never lose a session, never double-process one.

## Quick Start

Add the Engram trigger to your project memory (see [doc/poc.md](doc/poc.md) for the template), then:

```
You: engram                          # process current session
You: engram retroactive              # batch-process all old un-engrammed sessions
You: engram retroactive 954fa331     # process one specific old session
```

The agent generates extraction scripts, runs them, reads the signals, updates memory, and tags the session in a manifest -- all in one turn.

## Features

- **Zero copy-paste** -- agent orchestrates the full loop autonomously (since v3)
- **Session tagging** -- manifest tracks what's been engrammed, prevents double-processing (v4)
- **Retroactive batch mode** -- process all missed sessions chronologically, synthesize once (v4)
- **Retroactive single mode** -- process one specific old session on demand (v4)
- **Trivial session auto-skip** -- sessions with < 10 messages are tagged and skipped (v4)
- **Static + dynamic signals** -- git/codebase analysis + transcript behavioral analysis (since v2)

## Read More

- [Design rationale](doc/article.md) -- full breakdown of the three-layer architecture and the meta-learning feedback loop
- [PoC prompts + automation setup](doc/poc.md) -- v4 workflow, manifest format, trigger modes, version comparison

## License

See [LICENSE](LICENSE).
