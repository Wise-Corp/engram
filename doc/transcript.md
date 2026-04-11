# Conversation Transcript: From Claude Code Leak to Engram

**Date**: April 2, 2026  
**Participants**: User (Nizar, Wisecorp) and Claude (Anthropic)  
**Topic**: Analysis of the Claude Code source leak, critique of the "Dream" memory system, and iterative design of "Engram" — an edge-augmented memory consolidation system for AI coding agents.  
**Scope**: This transcript covers the initial design sessions (v1 → v2). The project has since evolved through v3 (full automation), v4 (session tagging, retroactive engramming), and v5 (one-line deployment). See [changelog.md](changelog.md) for the current state.

---

## 1. Did Anthropic open-source its source codebase recently?

**User**: did anthropic opensource its source codebase recently?

**Claude**: Not exactly — Anthropic didn't intentionally open-source its codebase. What happened was an accidental leak of the Claude Code source code, which occurred on March 31, 2026.

A security researcher ([Chaofan Shou on X](https://dev.to/gabrielanhaia/claude-codes-entire-source-code-was-just-leaked-via-npm-source-maps-heres-whats-inside-cjo)) found that Claude Code version 2.1.88, published to the npm registry, contained a source map file that exposed the full underlying codebase. The leak exposed around 500,000 lines of code across roughly 2,000 files ([Axios report](https://www.axios.com/2026/03/31/anthropic-leaked-source-code-ai)). Within hours, the code was mirrored on GitHub and widely dissected by developers.

Anthropic confirmed the incident, with a spokesperson stating that no sensitive customer data or credentials were involved, and that it was a release packaging issue caused by human error, not a security breach ([CNBC coverage](https://www.cnbc.com/2026/03/31/anthropic-leak-claude-code-internal-source.html)).

Important context: this is not the model weights or training data — it's the frontend/CLI TypeScript client, covering the agent logic, tool-calling system, and permission guardrails ([Tech Startups analysis](https://techstartups.com/2026/03/31/anthropics-claude-source-code-leak-goes-viral-again-after-full-source-hits-npm-registry-revealing-hidden-capybara-models-and-ai-pet/)). However, the leak also revealed unannounced features and internal roadmap details, including references to new model codenames ("Capybara" / "Mythos") and capabilities like persistent background assistance and memory consolidation across sessions.

Anthropic has been issuing DMCA takedown notices against GitHub mirrors of the leaked code ([Fortune report](https://fortune.com/2026/03/31/anthropic-source-code-claude-code-data-leak-second-security-lapse-days-after-accidentally-revealing-mythos/)), making it clear they consider this proprietary — not open source. This was also the company's second security lapse in days, following an earlier incident where close to 3,000 internal files were made publicly available, including a draft blog post about the "Mythos" model.

A detailed technical breakdown was published on GitHub by [Kuberwastaken](https://github.com/Kuberwastaken/claude-code), covering the full architecture including the multi-agent coordination, the "Dream" memory system, feature flags, and internal tooling. Another mirror by [nirholas](https://github.com/nirholas/claude-code) included an MCP server for exploring the source interactively.

---

## 2. Isn't memory consolidation already possible?

**User**: isn't memory consolidation across sessions already possible into their vscode extension? I'm using that by means of persistent md files that claude code stores in a hidden dir on a per-repo basis, maybe I didn't understand well?

**Claude**: You're right that Claude Code already has a memory system — the `.claude/` directory with `CLAUDE.md` files that persist project context across sessions. That's the current, shipped feature you're using.

What the leak revealed seems to be something more sophisticated on top of that. Based on the [breakdown from the leaked source](https://github.com/Kuberwastaken/claude-code), the unshipped system (internally called "Dream") goes beyond static markdown files in a few key ways.

The "Dream" system is a reflective pass where a subagent reviews recent sessions to synthesize learnings into durable, well-organized memories so future sessions can orient quickly. The dream subagent gets read-only bash access — it can look at the project but not modify anything. It's purely a memory consolidation pass.

The difference:

- **What you have now** — manually curated (or Claude-appended) markdown files in `.claude/` that store project conventions, preferences, and context. They're useful but relatively passive — Claude reads them at session start.
- **What was found in the leak** — a multi-phase system that automatically gathers recent signals from daily logs, drifted memories, and transcript searches, then consolidates them by converting relative dates to absolute, deleting contradicted facts, and pruning the memory file to stay under 200 lines and ~25KB. Think of it as an active background process that curates and cleans up memory automatically, rather than just appending notes.

---

## 3. Token cost concerns about Dream

**User**: it's clear now, but wouldn't the "Dream" subagent be going to make users consume more tokens, the way it is designed?

**Claude**: That's a very fair concern, and yes, almost certainly. The Dream subagent is fundamentally another Claude invocation — it reads transcripts, reviews memory files, reasons about what to keep or prune, and writes updated memories. All of that runs through the model and costs tokens.

And it's not trivial work either. Based on what was described, it has multiple phases — gathering signals from logs and transcripts, consolidating, then pruning and indexing. That's a decent amount of input and output tokens per "dream" cycle, and none of it is directly producing work the user asked for.

The question becomes how and when Anthropic triggers it. Running it after every session would be expensive and probably wasteful for short interactions. Running it periodically or only when memory files have drifted significantly would be more reasonable but harder to calibrate.

There's also the question of who pays. Anthropic could absorb the cost as a platform feature, or it could count against the user's token budget — which would be a harder sell, paying for work you didn't explicitly ask for.

If a well-timed dream pass saves even one or two "wait, I already told you this" exchanges per session, it could be token-neutral or even net-positive over time. But in practice, cost-conscious users would probably want the ability to disable it or control its frequency.

---

## 4. The edge computing insight

**User**: wouldn't it be smarter to make some pre-work on the edge computing power (no AI, but deterministic sorting, scanning, filtering, aggregating tasks) to compute a real added-value rich memory structure that can be, once submitted to Claude's API be more useful than submitting too much data most of which is unuseful?

**Claude**: That's a really sharp architectural insight, and it's the kind of optimization that separates cost-effective systems from brute-force ones.

The current design as leaked seems to be "throw everything at Claude and let it figure out what matters." You're proposing a pipeline where deterministic preprocessing does the heavy lifting before the LLM ever gets involved.

What could be done locally with zero AI cost: deduplication of near-identical notes, timestamp-based decay scoring, simple diffing to detect contradictions, frequency analysis of file interactions, categorization by type using basic heuristics, and token counting to pre-trim the payload.

All of that is fast, cheap, and deterministic — milliseconds on any modern machine at zero marginal cost. Then you hand the LLM a compact, pre-structured input and ask it to do only what it's uniquely good at: reasoning about relevance, resolving subtle contradictions, and writing natural-language summaries.

The contrast is stark: in the leaked design, Claude is basically doing janitorial work — scanning logs, finding dates, pruning line counts — that any script could handle. That's expensive janitorial work billed at API rates.

The edge approach would flip the cost profile: maybe 90% of the processing happens locally for free, and the LLM call becomes a short, focused, high-value step.

---

## 5. The meta-learning layer

**User**: wow, 90%? also I was thinking about the fact of rendering those deterministic tasks through an initial API call, so that Claude optimizes and personalizes these tasks based on the memory (not the md memory but the remote ram session memory at Anthropic). what I was asking myself about is what would be the meta-rules and meta-knowledge system that would "extract" such orientation questions that would be answered by the local (edge) deterministic calculus, and how they are handled to generate the final persistent context. it seems that this one would be a nice case of "learning" system, my questioning is purely theoretic.

**Claude**: The 90% was illustrative, not measured — fair pushback. But the core point stands.

Your actual idea is more interesting. You're describing three layers working together:

1. **An initial lightweight API call** where Claude, informed by its session-level memory of the user, generates structured queries — instructions for the local machine. Not answers, just questions.
2. **A local deterministic execution layer** that runs those queries against raw data cheaply and fast.
3. **A final synthesis step** where the structured results come back to the API for consolidation.

The meta-system needs to solve several problems simultaneously: knowing what kinds of local data exist and what's queryable, modeling what's likely to be valuable for this specific user, and improving over time.

One approach: a reinforcement loop. Claude generates query templates, the local layer executes them, the synthesis produces updated memory, and then on subsequent sessions there's an implicit feedback signal — did the memory actually get used? Did the user correct something? Did it prevent a redundant question?

The deeper challenge is bootstrapping. The first session has no memory to inform query generation, so you need a base set of generic extraction heuristics. Then the system specializes from there.

What you're describing is a system where the LLM acts as a compiler — it translates fuzzy, user-specific understanding into precise deterministic operations, then interprets the results. The intelligence is in the translation step, not in the data processing.

---

## 6. First deliverables — article and prompt

**User**: can you write an article for me about this vision, and give me a prompt that I can give to my Claude's vscode extension to implement it for me? I think that the first PoC of this can simply take the form of a "generic" prompt to execute at the end of every session (that would ask Claude to produce a series of scripts to be executed locally and their result submitted to Claude in a second prompt asking to produce the md result)

**Claude**: *[Generated two files: an article covering the full vision and a PoC prompt with two-phase workflow (Prompt 1: generate extraction scripts, Prompt 2: synthesize results into CLAUDE.md)]*

The PoC workflow:
1. End of session → paste Prompt 1 → Claude generates a shell script
2. Run `bash .claude/extract_signals.sh` locally (fast, free, deterministic)
3. Paste Prompt 2 with the JSON results → Claude produces updated `CLAUDE.md`

---

## 7. Choosing a project name

**User**: If it's possibly useful for others, I will opensource this into my company's Github repo. how would I name it? eamcaca? eamca? or something more original?

**Claude**: Honest feedback: eamcaca sounds like something you'd find in a medical dictionary.

Suggestions explored several directions:

- **Playing off "Dream"**: Lucid, Reverie
- **Functional metaphors**: Sieve, Glean, Distill
- **Technical/distinctive**: Prelude, Mnemo

Initial pick was **Lucid** — short, memorable, works as a verb, and the "lucid dreaming" connection to Anthropic's Dream codename is a nice nod.

---

## 8. Exploring the neuroscience of sleep and memory

**User**: any other name relative to the "dream" phenomenon? I guess Anthropic named it "dream" because when sleeping, there are many phases one of which permits to our neural brain to review what happened on that day and hence "learn" from it. explore this direction for me please

**Claude**: The neuroscience maps remarkably well:

- **N2 — Sleep Spindles**: short bursts of oscillatory brain activity that act as a gating mechanism, selecting which memories get transferred from hippocampus (short-term) to cortex (long-term). This is close to what the local extraction scripts do — the selection filter. **Spindle** as a name is technically accurate to the metaphor.
- **N3 — Slow-Wave Sleep**: where actual heavy consolidation happens. The hippocampus "replays" compressed versions of the day's experiences. It's not raw replay — the brain abstracts and compresses before transferring.
- **Sharp-wave ripples**: hippocampal firing patterns during replay. **Ripple** would be nice but has crypto connotations (XRP).
- **Engram**: the physical trace a memory leaves in neural tissue — the actual stored representation. The `CLAUDE.md` file is literally an engram.
- **Synaptic homeostasis** (SHY hypothesis by Tononi and Cirelli): sleep's primary function may be pruning — downscaling weak connections so strong ones stand out.

Top picks:
- **Spindle** — most precise metaphor for the local filtering layer
- **Engram** — the memory trace itself, more poetic

---

## 9. The user's first test and the static signal limitation

**User**: *[Shared the generated `extract_signals.sh` script — a working implementation with 7 extraction categories covering git diff analysis, frequency analysis, error patterns, convention detection, contradiction detection, dependency changes, and TODO scanning]*

I chose engram as a name. I tried prompt1 and it generated this script. I think this would be a good version one, but it only looks for "static" signals, I mean the prompt1 would always the same script that never depends on the context. I could imagine more "dynamic" signals like: reading the transcript of the session, statistically, which proposals did the user directly accept, and which ones did he refuse or ask to modify? any common aspect? this signal would "push" Claude into a sort of "strategical" sense of learning

**Claude**: Crucial observation. The script as generated is essentially a project scanner — it looks at the codebase, not at the interaction. It would produce the same output whether you had a brilliant productive session or spent an hour going back and forth on a bad approach. It learns nothing about *you*.

The transcript-derived signals are much more valuable:

- **Acceptance patterns**: "yes"/"do it" vs "no, actually..." — classifiable with keyword matching, no AI needed
- **Correction vocabulary**: "too verbose", "simpler", "don't use classes" — extractable with grep, pure gold
- **Iteration count per task**: same file edited 3 times before acceptance = friction signal
- **Prompt-to-completion ratio**: messages getting longer = the user is compensating for lost context
- **Abandoned paths**: "forget that, let's try something else" = preference pair

*[Generated Engram PoC v2 with `extract_static.sh` and `extract_dynamic.sh` concept, plus updated Prompt 2 with User Preferences section]*

---

## 10. The domain knowledge dimension

**User**: you caught half of my point, possibly because of my too specific example, I think it's possible for extract_dynamic.sh to extract specificities of the project like the business workflow it is implementing, or where claude tends to mis-interpret some technical or functional questions specific to the project itself, not only the user himself and his tastes in matter of design and coding, what do you propose?

**Claude**: You're absolutely right. I was narrowing the lens too much to the developer-as-coder when the developer is also a domain expert who keeps having to re-explain the *business* to Claude.

Half the friction in real sessions isn't about code style — it's about Claude misunderstanding the domain:

- **Terminology corrections**: "that's not an order, it's a quotation, they're different states in our workflow." Extractable by patterns: "X is not Y", "when I say X I mean", "X and Y are different things here."
- **Business rule violations**: Claude proposes hard-delete but the domain requires soft-delete for audit compliance. The user's correction contains the "because" clause — pure domain knowledge.
- **Workflow misunderstandings**: the project has a state machine (draft → submitted → validated → invoiced) and Claude keeps suggesting transitions that skip states.
- **Repeated explanations**: if the user explains the same business concept twice across sessions, the memory system has failed. Detectable by long user messages that follow a rejection.
- **Entity relationship confusion**: Claude treats two models as interchangeable when they serve different business purposes.

Restructured the dynamic extraction into two explicit dimensions:

**Dimension A: User Preferences** — acceptance rates, iteration depth, style corrections. "How does this person want to work?"

**Dimension B: Domain Knowledge** — terminology corrections, business rules, workflow constraints, entity relationships, repeated explanations. "What does this project mean in business terms?"

Updated the `CLAUDE.md` structure with four new domain-specific sections (Domain Model, Business Rules, Workflow & State Machines, Domain Vocabulary) placed above the technical sections — so Claude internalizes business constraints before touching any code.

The most expensive AI errors aren't wrong indentation — they're "confidently built the feature in a way that violates a business rule nobody coded explicitly yet." Those corrections cost the most tokens to fix in real-time, and they're exactly what a grep for "because" and "can't do that" will catch in the transcript for free.

---

## What Happened Next

After this conversation, the project evolved through two more iterations:

- **v3** (in-session automation): eliminated the copy-paste workflow entirely. The agent now generates scripts, runs them, reads the signals, and synthesizes memory — all in a single turn triggered by saying "engram".
- **v4** (session tagging & retroactive engramming): added a manifest that tracks which sessions have been engrammed, preventing double-processing. Supports retroactive batch mode (process all missed sessions chronologically) and single mode (process one specific old session). Trivial sessions (< 10 messages) are auto-skipped.
- **v5** (one-line deployment): users deploy engram by telling Claude `deploy engram from https://github.com/Wise-Corp/engram`. No cloning, no manual file copying. The trigger file is self-updating.

## Summary of Deliverables (from this session)

| File | Description |
|------|-------------|
| `article.md` | Full article: "Edge-Augmented Memory Consolidation for AI Coding Agents" |
| `changelog.md` | Version history and design log (evolved through v1→v5) |

## Key References

| Source | Link |
|--------|------|
| Axios — Anthropic leaked 500,000 lines of source code | [axios.com](https://www.axios.com/2026/03/31/anthropic-leaked-source-code-ai) |
| Fortune — Second major security breach | [fortune.com](https://fortune.com/2026/03/31/anthropic-source-code-claude-code-data-leak-second-security-lapse-days-after-accidentally-revealing-mythos/) |
| CNBC — Anthropic leaks Claude Code internal source | [cnbc.com](https://www.cnbc.com/2026/03/31/anthropic-leak-claude-code-internal-source.html) |
| DEV Community — Full technical breakdown | [dev.to](https://dev.to/gabrielanhaia/claude-codes-entire-source-code-was-just-leaked-via-npm-source-maps-heres-whats-inside-cjo) |
| GitHub — Kuberwastaken's breakdown & source mirror | [github.com](https://github.com/Kuberwastaken/claude-code) |
| GitHub — nirholas mirror with MCP explorer | [github.com](https://github.com/nirholas/claude-code) |
| The AI Corner — Deep dive on 44 feature flags | [the-ai-corner.com](https://www.the-ai-corner.com/p/claude-code-source-code-leaked-2026) |
| Tech Startups — Capybara models and architecture details | [techstartups.com](https://techstartups.com/2026/03/31/anthropics-claude-source-code-leak-goes-viral-again-after-full-source-hits-npm-registry-revealing-hidden-capybara-models-and-ai-pet/) |
| Business Standard — Timeline and DMCA response | [business-standard.com](https://www.business-standard.com/technology/tech-news/anthropic-leaks-source-code-claude-code-again-what-happened-explained-126040100384_1.html) |