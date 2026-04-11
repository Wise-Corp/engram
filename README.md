# engram

Edge-Augmented Memory Consolidation for AI Coding Agents -- an ideation by [Wisecorp](https://wisecorp.tn), a technology startup building FPGA-accelerated solutions for modern hybrid applications.

## Why

AI coding agents are starting to remember things across sessions. But current approaches dump everything into the LLM context window -- paying inference rates for string matching, diffing, and counting that a shell script handles in milliseconds at zero cost.

engram splits memory consolidation into three layers: lightweight LLM query generation, deterministic local execution (zero API cost), and focused LLM synthesis. Two small API calls. Arbitrarily complex local processing in between.

## Install

Tell Claude:

```
deploy engram from https://github.com/Wise-Corp/engram
```

That's it. Claude fetches the trigger file and sets up your project. Then:

```
You: engram                          # process current session
You: engram retroactive              # batch-process all old un-engrammed sessions
You: engram retroactive 954fa331     # process one specific old session
```

## Status

**v5** -- one-line deployment, fully automated consolidation with session tagging and retroactive engramming. See [changelog](doc/changelog.md) for version history.

## Project Structure

```
engram.md              <- The trigger file. Claude deploys this for you.
install.md             <- Setup instructions (manual alternative)
prompts/
  prompt1-extraction.md  <- Controls what signals get extracted
  prompt2-synthesis.md   <- Controls how signals become memory
doc/
  how-it-works.md      <- Three-layer architecture explained
  changelog.md         <- Version history (v1 through v5)
  article.md           <- Design rationale essay
  transcript.md        <- Original design conversation
```

## Read More

- [How it works](doc/how-it-works.md) -- the three-layer architecture, signal taxonomy, manifest system
- [Design rationale](doc/article.md) -- full breakdown of edge-augmented consolidation
- [Changelog](doc/changelog.md) -- version history, v4.1 improvements, design decisions

## License

See [LICENSE](LICENSE).
