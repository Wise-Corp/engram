# Prompt 2 — Synthesize Memory Update

This prompt is used after extraction scripts have run. The agent sends the structured signals JSON along with current memory to the LLM, which produces an updated memory file.

---

```
Here are the structured signals extracted from my project.

<signals>
{{SIGNALS_JSON — single object for live/single mode, or array for batch mode}}
</signals>

Current persistent memory:
<current_memory>
{{CURRENT_MEMORY}}
</current_memory>

Update the persistent memory. Follow these rules:

## Content rules
1. Preserve high-value existing entries that are still true
2. Update contradicted facts with "(updated YYYY-MM-DD)" note
3. Add new learnings from signals (conventions, architecture, deps)
4. Prune stale entries not reflected in recent signals
5. Use absolute dates, never "recently" or "yesterday"
6. If signals are thin (light session, no corrections, no code changes) — skip the update
   and report that. Don't force changes when there's nothing to consolidate.
   **Exception — new projects**: If no project memory exists yet (MEMORY.md contains only
   the engram trigger, or fewer than 3 memory files), bootstrap baseline project memory
   from available static signals and direct codebase inspection. An empty memory is worse
   than a thin but accurate one. Create at minimum: project-overview, architecture, and
   conventions files derived from the codebase itself.

## Batch mode rules (when signals is an array)
- Signals are ordered chronologically (oldest session first)
- Later sessions supersede earlier ones when they contradict
- Domain knowledge should reflect the cumulative understanding
- User preferences may evolve — note trends, weight recent sessions more heavily
- A single synthesis pass should produce ONE coherent memory update, not N incremental ones

## Structure
Use these sections (adapt to your memory format — CLAUDE.md, individual files, etc.):
- **Project Overview**: what this project is and does
- **Domain Model**: business entities, relationships, disambiguations.
  Written to prevent AI from confusing related-but-distinct concepts
- **Business Rules**: hard constraints, phrased as non-negotiable rules with reasons
- **Workflow & State Machines**: process flows with explicit ordering
- **Domain Vocabulary**: project-specific terminology and meaning
- **Architecture Decisions**: key technical choices and rationale
- **Conventions**: coding patterns, naming, file organization
- **User Preferences**: actionable instructions for future sessions
- **Active Work**: current focus areas
- **Known Issues**: bugs, tech debt, friction points

## Domain Knowledge sections (from Dimension B signals)
- Every terminology correction → vocabulary entry or disambiguation note
- Every business rule → hard constraint with its reason
- Every workflow correction → explicit ordering constraint
- Every repeated explanation → extra emphasis (Claude keeps getting this wrong)
- Phrase as RULES, not descriptions:
  "NEVER confuse Client (company entity) with User (login account)."
  "RULE: An invoice can only be generated from a validated order."
  "WORKFLOW: Payment must be recorded BEFORE delivery confirmation is sent."

## User Preferences section (from Dimension A signals)
- High acceptance rate for certain actions → note as strengths
- Consistently rejected patterns → note as anti-patterns FOR THIS USER
- Rising prompt complexity → note what topics caused confusion
- High iteration depth files → note friction areas
- Phrase as actionable instructions:
  "User prefers small, focused commits over large batches"
  "User rejects class-based patterns — always use functions"
  "When editing templates, propose changes in small increments — high iteration area"
```
