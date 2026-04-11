# Installing Engram

## Prerequisites

- Claude Code (CLI, VS Code extension, or JetBrains extension)
- Auto-memory enabled (default in Claude Code)
- `bash`, `python3`, `git` available on your machine

## Setup (30 seconds)

**1. Copy the trigger file into your project's memory directory:**

```bash
# From your project root:
mkdir -p .claude/memory
cp /path/to/engram/engram.md .claude/memory/engram.md
```

Or if you cloned the repo:
```bash
cp ~/engram/engram.md .claude/memory/engram.md
```

**2. Add it to your memory index:**

If you have a `.claude/memory/MEMORY.md` file, add this line:
```
- [Engram](engram.md) — automated memory consolidation trigger
```

**3. Use it:**

```
You: engram                          # process current session
You: engram retroactive              # batch-process all old un-engrammed sessions
You: engram retroactive 954fa331     # process one specific old session
```

That's it. The agent handles everything else — generating extraction scripts, running them, synthesizing memory, tagging the session.

## What happens under the hood

See [doc/how-it-works.md](doc/how-it-works.md) for the three-layer architecture, or [doc/changelog.md](doc/changelog.md) for the full design history.

## Customizing the prompts

The two prompts that drive engram are in [prompts/](prompts/). You can read and modify them to tune extraction or synthesis for your workflow:

- [prompt1-extraction.md](prompts/prompt1-extraction.md) — controls what signals the extraction scripts look for
- [prompt2-synthesis.md](prompts/prompt2-synthesis.md) — controls how signals become memory updates
