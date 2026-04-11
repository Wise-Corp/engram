# Installing Engram

## The easy way (recommended)

Tell Claude:

```
deploy engram from https://github.com/Wise-Corp/engram
```

Claude will fetch the latest trigger file, copy it into your project's memory, and set up the index. Done.

## Manual setup

If you prefer to do it yourself:

```bash
# From your project root:
mkdir -p .claude/memory
curl -sL https://raw.githubusercontent.com/Wise-Corp/engram/main/engram.md -o .claude/memory/engram.md
```

Add to `.claude/memory/MEMORY.md`:
```
- [Engram](engram.md) — automated memory consolidation trigger
```

## Updating

Same command — Claude will fetch the latest version and overwrite the old one:

```
update engram from https://github.com/Wise-Corp/engram
```

## Usage

```
You: engram                          # process current session
You: engram retroactive              # batch-process all old un-engrammed sessions
You: engram retroactive 954fa331     # process one specific old session
```

## How it works

See [doc/how-it-works.md](doc/how-it-works.md) for the three-layer architecture, or [doc/changelog.md](doc/changelog.md) for the full design history.
