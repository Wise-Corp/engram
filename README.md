# engram

Edge-Augmented Memory Consolidation for AI Coding Agents -- an ideation by [Wisecorp](https://wisecorp.tn), a technology startup building FPGA-accelerated solutions for modern hybrid applications.

## Why

AI coding agents are starting to remember things across sessions. But current approaches dump everything into the LLM context window -- paying inference rates for string matching, diffing, and counting that a shell script handles in milliseconds at zero cost.

engram splits memory consolidation into three layers: lightweight LLM query generation, deterministic local execution (zero API cost), and focused LLM synthesis. Two small API calls. Arbitrarily complex local processing in between.

## Quick Start

**1. Copy the trigger file into your project:**

```bash
mkdir -p .claude/memory
cp engram.md .claude/memory/engram.md
```

**2. Add to your memory index** (`.claude/memory/MEMORY.md`):

```
- [Engram](engram.md) — automated memory consolidation trigger
```

**3. Use it:**

```
You: engram                          # process current session
You: engram retroactive              # batch-process all old un-engrammed sessions
You: engram retroactive 954fa331     # process one specific old session
```

The agent generates extraction scripts, runs them, reads the signals, updates memory, and tags the session -- all in one turn, no copy-paste.

## Status

**PoC v4** -- fully automated with session tagging and retroactive engramming. See [v4.1 improvements](doc/poc.md#v41--dynamic-signal-quality-improvements) for signal quality tuning from field testing.

## Project Structure

```
engram.md              <- The one file you need. Copy it, done.
install.md             <- Step-by-step setup instructions
prompts/
  prompt1-extraction.md  <- Controls what signals get extracted
  prompt2-synthesis.md   <- Controls how signals become memory
doc/
  how-it-works.md      <- Three-layer architecture explained
  poc.md               <- Full PoC history (v1 through v4.1)
  article.md           <- Design rationale essay
  transcript.md        <- Original design conversation
```

## Read More

- [Install guide](install.md) -- 30-second setup
- [How it works](doc/how-it-works.md) -- the three-layer architecture, signal taxonomy, manifest system
- [Design rationale](doc/article.md) -- full breakdown of edge-augmented consolidation
- [PoC changelog](doc/poc.md) -- version history, v4.1 improvements, design decisions

## License

See [LICENSE](LICENSE).
