---
name: claudemd-architect
description: Generate production-grade CLAUDE.md project reference documents for Claude Code. Use when a user wants to create a CLAUDE.md, project spec, or architectural reference for AI-assisted development. Incorporates LLM reasoning failure mitigations (constraint-based guardrails, concrete examples, verification checkpoints) to produce documents that minimize AI coding errors. Trigger on mentions of "CLAUDE.md", "project setup for Claude Code", "AI coding reference", or when starting a new software project that will use Claude Code.
---

# CLAUDE.md Architect

A skill for generating high-quality CLAUDE.md project reference documents that serve as the single source of truth for AI-assisted development with Claude Code.

## Why This Skill Exists

A CLAUDE.md is not just documentation — it's a **steering mechanism** for an AI coding agent. A poorly written CLAUDE.md leads to architectural drift, scope creep, silent bugs, and wasted iterations. A well-written one produces code that aligns with the developer's intent on the first try.

This skill incorporates findings from LLM reasoning failure research (Song et al., "Large Language Model Reasoning Failures", TMLR 2026) to produce documents that actively prevent common AI coding mistakes.

## Core Philosophy

A great CLAUDE.md follows five principles:

1. **Constraints over suggestions.** "MUST use SQLite" is better than "we prefer SQLite." LLMs treat suggestions as optional and will drift. Constraints are respected.

2. **Rationale with every decision.** "We chose Go because..." prevents the LLM from reasoning "wouldn't Python be better here?" mid-task. Understanding WHY blocks reasoning drift.

3. **Concrete examples over abstract descriptions.** A code snippet showing the exact pattern is worth 10 paragraphs of architectural prose. LLMs compose concrete patterns more reliably than abstract instructions.

4. **Explicit anti-patterns.** Telling the AI what NOT to do is as important as telling it what to do. LLMs suffer from "proactive initiative" — they'll add features, frameworks, and abstractions unless explicitly told not to.

5. **Verification checkpoints.** LLMs lose context over long tasks. Periodic "stop and verify" points catch drift before it compounds.

---

## Workflow: Three Stages

### Stage 1: Discovery Interview

**Goal:** Extract all architectural decisions, constraints, and context from the developer's mind (or from an existing conversation).

There are two entry paths:

**Path A — Fresh project:** Run the interview questionnaire (see `references/interview.md`). Ask questions across 8 dimensions: project purpose, tech stack, constraints, deployment, data model, API/integrations, team context, and phasing.

**Path B — Existing conversation:** The user may say "turn our conversation into a CLAUDE.md" or paste a conversation transcript. In this case:
1. Read the entire conversation/transcript carefully.
2. Extract all decisions made and their rationale.
3. Identify any unresolved questions or implicit assumptions.
4. Present a summary of extracted decisions and ask the user to confirm or correct.
5. Fill gaps with targeted questions.

**Key interview principles:**
- Ask for DECISIONS, not preferences. "What database will you use?" not "What databases do you like?"
- Ask for REJECTED alternatives. "What did you consider and reject?" — this becomes ADR rationale.
- Ask about DEPLOYMENT constraints. Where it runs determines what's possible.
- Ask about the TEAM. Solo dev? 10-person team? This affects how prescriptive the doc should be.
- Collect concrete examples: existing code snippets, API responses, data schemas.

### Stage 2: Document Generation

**Goal:** Produce the CLAUDE.md following the proven template structure.

Read `references/template.md` for the full template structure. The document MUST include these sections in order:

1. **Header** — Project name, one-line description, tagline if available.
2. **AI Reading Guide** — Explicit instructions for how Claude Code should interpret this document.
3. **Project Vision** — What the project does, in 3-5 sentences. Layered capabilities if applicable.
4. **Critical Constraints** — MUST and MUST NOT rules. Non-negotiable. Read first.
5. **Architecture Decisions (ADR Log)** — Every major technical decision with: Decision, Context, Rationale, Rejected alternatives.
6. **Key Interfaces** — The contracts between system layers, as actual code.
7. **Project Structure** — Directory tree with comments explaining each file's purpose.
8. **Configuration** — Config struct/schema with all settings, defaults, and descriptions.
9. **Infrastructure** — Docker Compose, deployment configs, environment setup.
10. **Integration Notes** — External API details, auth patterns, critical implementation patterns with code.
11. **Development Phases & TODO** — Phase-gated task list with exit criteria per phase.
12. **Common Pitfalls** — Explicit anti-patterns mapped to known LLM failure modes.
13. **Quick Reference** — Summary table: Component → Technology → Why.

**Generation principles:**
- Write ADRs in the voice of a technical decision log, not a tutorial.
- Include code examples for every pattern that Claude Code will need to implement.
- Use compile-time interface checks (`var _ Interface = (*Impl)(nil)`) for typed languages.
- Phase TODO items should be checkbox-style for trackability.
- Each phase needs explicit exit criteria — "how do we know this phase is done?"

### Stage 3: Review & Refinement

**Goal:** Verify completeness and quality before handoff.

Run through this checklist with the developer:

**Completeness:**
- [ ] Every technology choice has an ADR with rationale?
- [ ] Every ADR mentions what was rejected and why?
- [ ] Critical constraints cover both MUST and MUST NOT?
- [ ] All external integrations have auth + error handling patterns?
- [ ] All interfaces are defined with actual code signatures?
- [ ] Project structure accounts for every major component?
- [ ] Configuration covers all environment-specific settings?
- [ ] TODO phases are ordered by dependency (no forward references)?
- [ ] Each phase has exit criteria?

**LLM Failure Prevention:**
- [ ] Constraints are stated as absolutes, not suggestions?
- [ ] Code examples exist for the 3-5 most critical patterns?
- [ ] Anti-patterns section covers at least: error handling, scope creep, premature optimization, pagination/completeness, testing?
- [ ] Verification checkpoints exist for each phase boundary?
- [ ] The AI reading guide at the top sets clear behavioral expectations?

**Practical:**
- [ ] A developer unfamiliar with the project can understand the architecture from this doc alone?
- [ ] The document can be dropped into a fresh Claude Code session and produce aligned code?

---

## Adapting to Project Size

Not every project needs a 500-line CLAUDE.md. Scale the document to the project:

**Small project (single purpose, 1 developer, < 2 weeks):**
- Header + Vision + Constraints + ADRs (3-5) + Structure + TODO
- Skip: Interfaces section (define inline), Infrastructure, Pitfalls
- ~100-200 lines

**Medium project (multiple components, 1-3 developers, 1-3 months):**
- Full template, all sections
- ADRs for every significant choice
- Code examples for critical patterns
- ~300-500 lines

**Large project (platform/product, team, ongoing):**
- Full template + additional reference documents
- Split into CLAUDE.md (overview + constraints + ADRs) + separate ARCHITECTURE.md, API.md, etc.
- CLAUDE.md stays under 600 lines, references other docs
- ~400-600 lines for CLAUDE.md alone

---

## Reference Files

- `references/template.md` — The full CLAUDE.md template with placeholder content and annotations
- `references/interview.md` — The structured interview questionnaire for Stage 1
- `references/llm-mitigations.md` — LLM reasoning failure types and how each section prevents them
- `examples/bellek.md` — A real-world example: the Bellek M365 backup + RAG platform CLAUDE.md
