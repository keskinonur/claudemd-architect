# claudemd-architect

Generate production-grade CLAUDE.md files that steer AI coding agents. Incorporates LLM reasoning failure mitigations to prevent architectural drift, scope creep, and silent bugs in Claude Code sessions.

Based on [Large Language Model Reasoning Failures](https://arxiv.org/abs/2602.06176) (Song et al., TMLR 2026).

---

## The Problem

A CLAUDE.md is not just documentation — it's a **steering mechanism** for an AI coding agent. A poorly written one leads to:

- 🔀 **Architectural drift** — the AI gradually deviates from your intended design
- 🐛 **Silent bugs** — errors that compile and run but produce incorrect results
- 📦 **Scope creep** — unauthorized frameworks, abstractions, and "helpful" additions
- 🔄 **Wasted iterations** — fixing AI mistakes that a better prompt would have prevented

## The Solution

This skill produces CLAUDE.md files designed around **five LLM failure prevention principles:**

| Principle | What It Does | Which Failure It Prevents |
|---|---|---|
| **Constraints over suggestions** | `MUST` / `MUST NOT` rules, not recommendations | Confirmation bias, proactive initiative |
| **Rationale with every decision** | ADRs with "why" and rejected alternatives | Reasoning drift, inverse inference |
| **Concrete code examples** | Exact patterns, not abstract descriptions | Compositional reasoning failures |
| **Explicit anti-patterns** | What NOT to do, mapped to failure categories | All failure modes |
| **Verification checkpoints** | Stop-and-verify gates at phase boundaries | Working memory limitations |

---

## Installation

### Claude Code (global — recommended)

```bash
git clone https://github.com/keskinonur/claudemd-architect.git
cp -r claudemd-architect ~/.claude/skills/claudemd-architect
```

Now available in all your Claude Code projects as `/claudemd-architect` or triggered automatically when you mention creating a CLAUDE.md.

### Claude Code (project-specific)

```bash
cp -r claudemd-architect /path/to/your/project/.claude/skills/claudemd-architect
```

### Claude.ai (Web/App)

1. Download the [latest release](https://github.com/keskinonur/claudemd-architect/releases) as ZIP
2. Go to **Settings → Capabilities → Skills → Upload skill**
3. Upload the ZIP file

---

## Usage

### Starting a new project

```
I'm starting a new project. Help me create a CLAUDE.md.
```

The skill runs a structured interview (8 dimensions, 28 questions) to extract your architectural decisions, then generates a complete CLAUDE.md.

### Converting an existing conversation

```
Turn our conversation into a CLAUDE.md for Claude Code.
```

The skill extracts decisions and rationale from your conversation history and fills gaps with targeted questions.

### Specifying project size

```
Create a CLAUDE.md for a small CLI tool — keep it concise.
```

The skill scales the document: ~100 lines for small projects, ~300-500 for medium, ~500+ for large platforms.

---

## What It Generates

A CLAUDE.md with these sections:

```
1.  Header & AI Reading Guide     ← Sets behavioral expectations
2.  Project Vision                 ← Core purpose in 3-5 sentences
3.  Critical Constraints           ← MUST / MUST NOT rules
4.  Architecture Decisions (ADRs)  ← Every choice with rationale
5.  Key Interfaces                 ← Actual code contracts
6.  Project Structure              ← Directory tree with annotations
7.  Configuration                  ← All settings as code
8.  Infrastructure                 ← Docker Compose, deployment
9.  Integration Notes              ← External API patterns with code
10. Development Phases & TODO      ← Phase-gated tasks with exit criteria
11. Common Pitfalls                ← Anti-patterns by failure category
12. Quick Reference                ← One-glance summary table
```

---

## Skill Structure

```
claudemd-architect/
├── SKILL.md                           # Entry point — 3-stage workflow
├── references/
│   ├── interview.md                   # 28-question discovery questionnaire
│   ├── template.md                    # Annotated CLAUDE.md template
│   └── llm-mitigations.md            # Failure mode → mitigation map
└── examples/
    └── bellek.md                      # Real-world example (M365 backup + RAG platform)
```

### Reference Files

| File | Purpose |
|---|---|
| `interview.md` | Structured interview in 7 rounds: project identity, tech stack, constraints, architecture, deployment, phasing, context |
| `template.md` | Complete CLAUDE.md template with HTML annotations explaining WHY each section exists and which LLM failure it prevents |
| `llm-mitigations.md` | Maps 6 LLM reasoning failure categories to specific CLAUDE.md design patterns. The theoretical foundation of this skill. |
| `examples/bellek.md` | Production CLAUDE.md for [Bellek](https://bellek.ai) — a Go-based M365 backup & RAG platform |

---

## The Research Behind It

This skill applies findings from **"Large Language Model Reasoning Failures"** (Song, Han & Goodman, Transactions on Machine Learning Research, January 2026) — the first comprehensive survey of LLM reasoning failures.

The paper identifies failures across two axes:

**Axis 1 — Reasoning Type:** Informal (intuitive), Formal (logical), Embodied (physical)

**Axis 2 — Failure Type:** Fundamental limitations, Application-specific failures, Robustness issues

We mapped six failure categories most relevant to AI-assisted coding:

| Failure Category | How It Manifests in Coding | CLAUDE.md Mitigation |
|---|---|---|
| **Working memory** | Forgets constraints after 500+ lines | Verification checkpoints, Quick Reference |
| **Confirmation bias** | Sticks with first approach even when wrong | ADRs with rejected alternatives, anti-patterns |
| **Proactive initiative** | Adds unrequested features/frameworks | MUST NOT rules, phase boundaries |
| **Compositional reasoning** | Drops steps in multi-concern logic | Concrete code examples for complex patterns |
| **Inverse inference** | "It compiles" ≠ "it's correct" | Exit criteria with negative tests |
| **Robustness** | Fails on empty/edge/malformed inputs | Explicit edge case lists in pitfalls |

📄 Paper: [arxiv.org/abs/2602.06176](https://arxiv.org/abs/2602.06176)
📚 Resource list: [github.com/Peiyang-Song/Awesome-LLM-Reasoning-Failures](https://github.com/Peiyang-Song/Awesome-LLM-Reasoning-Failures)

---

## Contributing

Contributions welcome! Especially:

- **New examples** — CLAUDE.md files generated by this skill for different project types
- **Template improvements** — Additional sections or better annotation
- **Language-specific adaptations** — Better patterns for Python, Rust, Swift, etc.
- **Interview improvements** — Additional discovery questions or refined flow

---

## License

MIT

---

## Author

[Onur Keskin](https://github.com/keskinonur) — R&D Manager specializing in industrial automation, AI integration, and digital transformation.

Built while developing [Bellek](https://bellek.ai), a self-hosted M365 backup & RAG platform.
