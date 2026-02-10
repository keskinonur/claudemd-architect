# CLAUDE.md Template

This is the template structure for generating CLAUDE.md files. Sections marked [REQUIRED] must always be included. Sections marked [RECOMMENDED] should be included for medium+ projects. Sections marked [OPTIONAL] are for large projects or specific needs.

Replace all `{{placeholder}}` values with project-specific content. Remove all `<!-- annotation -->` comments in the final output.

---

```markdown
# CLAUDE.md — {{Project Name}}

<!-- 
  PURPOSE: Immediate identity. The AI sees this first.
  If the project has a tagline or one-liner, include it.
-->
> {{One-line project description}}
> *{{Optional tagline}}*

<!-- 
  PURPOSE: Sets behavioral expectations for the AI agent.
  This is the single most important paragraph in the document.
  LLM FAILURE PREVENTED: Proactive initiative — without this, the AI
  will invent its own interpretation of how to use the document.
-->
> **Reading guide for AI assistants:** This document is the single source of truth for all architectural decisions. When in doubt, re-read the relevant ADR before writing code. Do NOT invent alternative approaches — every major choice has been deliberately made and documented with rationale. If you find yourself unsure, ask the developer rather than guessing.

---

## Project Vision [REQUIRED]

<!--
  PURPOSE: 3-5 sentences max. What does this do? Who is it for?
  If the project has layered capabilities, list them numbered.
  LLM FAILURE PREVENTED: Working memory — the AI will reference this
  section when making micro-decisions deep in implementation.
-->

{{Project description in 3-5 sentences}}

**Core Principle:**
> {{The one rule that trumps all others — e.g., "Security over convenience", "Restore fidelity over feature count"}}

---

## Critical Constraints [REQUIRED]

<!--
  PURPOSE: Non-negotiable rules stated as absolutes.
  This section exists because LLMs treat suggestions as optional.
  MUST rules = things the AI must always do.
  MUST NOT rules = things the AI must never do.
  LLM FAILURES PREVENTED:
  - Confirmation bias: AI won't rationalize violating a clear MUST NOT
  - Proactive initiative: AI won't add unauthorized frameworks/services
  - Compositional failure: Explicit rules survive long context better than implicit ones
-->

### MUST Rules
- **MUST** {{constraint 1 — e.g., "use explicit error handling for every function that can fail"}}
- **MUST** {{constraint 2}}
- **MUST** {{constraint 3}}

### MUST NOT Rules
- **MUST NOT** {{prohibition 1 — e.g., "add any JavaScript framework — UI is server-rendered only"}}
- **MUST NOT** {{prohibition 2}}
- **MUST NOT** {{prohibition 3}}

### Verification Checkpoints
<!--
  PURPOSE: Concrete checklist the AI runs at phase boundaries.
  LLM FAILURE PREVENTED: Working memory limitations — after 500 lines
  of code, the AI may have forgotten constraints from this section.
  Checkpoints force re-verification.
-->
Before considering any phase complete, verify:
- [ ] {{Check 1 — e.g., "All error paths are handled"}}
- [ ] {{Check 2 — e.g., "All interfaces have compile-time verification"}}
- [ ] {{Check 3 — e.g., "No new external dependencies were introduced"}}
- [ ] {{Check 4 — e.g., "Tests exist for happy path AND error cases"}}

---

## Architecture Decisions [REQUIRED]

<!--
  PURPOSE: Every major technical decision, documented as an ADR.
  FORMAT: Decision → Context → Rationale → Rejected alternatives
  WHY RATIONALE MATTERS: If the AI understands WHY Go was chosen,
  it won't suggest rewriting a module in Python "for convenience."
  LLM FAILURES PREVENTED:
  - Reasoning drift: Rationale anchors the AI to the original reasoning
  - Inverse inference: "We rejected X because Y" prevents re-proposing X
-->

### ADR-001: {{Decision Title — e.g., "Language — Go"}}

**Decision:** {{What was decided}}

**Context:** {{What problem this solves, what alternatives existed}}

**Rationale:**
- {{Reason 1 — be specific, not generic}}
- {{Reason 2}}
- {{Reason 3}}

**Rejected:** {{What was considered and why it was rejected}}

<!--
  Repeat ADR-NNN for each major decision. Common decisions to document:
  - Programming language
  - Web framework (or lack thereof)
  - Database choice
  - Authentication method
  - Real-time communication (SSE/WS/polling)
  - Deployment model
  - External service choices
  - UI framework
-->

### ADR-002: {{Next Decision}}
...

---

## Key Interfaces [RECOMMENDED]

<!--
  PURPOSE: The contracts between system layers, as ACTUAL CODE.
  Not prose descriptions — real type definitions.
  LLM FAILURE PREVENTED: Compositional reasoning — abstract descriptions
  of interfaces lead to inconsistent implementations. Concrete code
  signatures are composed correctly by LLMs far more reliably.
-->

```{{language}}
// Define every interface that represents a boundary between components.
// Include compile-time checks where the language supports them.

type {{Interface1}} interface {
    {{Method1}}(ctx context.Context, ...) ({{ReturnType}}, error)
    {{Method2}}(ctx context.Context, ...) ({{ReturnType}}, error)
}

type {{Interface2}} interface {
    ...
}

// Compile-time interface checks (for Go — adapt for other languages)
var _ {{Interface1}} = (*{{Implementation1}})(nil)
```

---

## Project Structure [REQUIRED]

<!--
  PURPOSE: Directory tree with EVERY file explained.
  The AI uses this as a map — it decides where to create files based on this.
  If a directory isn't here, the AI will invent its own location.
  LLM FAILURE PREVENTED: Proactive initiative — without structure,
  the AI creates files wherever it thinks best, often inconsistently.
-->

```
{{project-name}}/
├── {{file1}}                    # {{Purpose of this file}}
├── {{file2}}                    # {{Purpose}}
├── {{dir1}}/
│   ├── {{file3}}               # {{Purpose}}
│   └── {{file4}}               # {{Purpose}}
└── ...
```

---

## Configuration [RECOMMENDED]

<!--
  PURPOSE: All configurable values with types, defaults, and descriptions.
  Presented as actual code (struct, schema, or config file format).
  LLM FAILURE PREVENTED: The AI won't invent config keys or hardcode
  values that should be configurable if they're all listed here.
-->

```{{language}}
type Config struct {
    {{Field1}} {{type}} `env:"{{ENV_VAR}}" default:"{{default}}"`  // {{description}}
    {{Field2}} {{type}} `env:"{{ENV_VAR}}" default:"{{default}}"`  // {{description}}
    ...
}
```

---

## Infrastructure [RECOMMENDED]

<!--
  PURPOSE: Docker Compose, deployment manifests, or setup instructions.
  Actual config files, not descriptions of config files.
-->

```yaml
# docker-compose.yml
services:
  {{service1}}:
    build: .
    ...
```

---

## Integration Notes [OPTIONAL — include if external APIs exist]

<!--
  PURPOSE: Auth patterns, rate limiting, critical gotchas for external APIs.
  MUST include actual code showing the correct pattern.
  LLM FAILURE PREVENTED: Compositional reasoning failure in API integration
  is the #1 source of bugs in AI-generated code. Showing the exact
  throttling/retry/pagination pattern prevents the AI from inventing
  a broken version.
-->

### {{API Name}} Integration

**Authentication:** {{How to authenticate}}

**Critical patterns:**
```{{language}}
// Show the EXACT pattern for:
// - Auth token acquisition
// - Rate limit handling (429 responses)
// - Pagination (follow all pages)
// - Error handling
{{code example}}
```

**Common mistakes to avoid:**
- {{Mistake 1 — e.g., "Don't stop at first page of results"}}
- {{Mistake 2}}

---

## Development Phases & TODO [REQUIRED]

<!--
  PURPOSE: Phase-gated task list. Each phase has explicit exit criteria.
  CRITICAL RULE: "Complete each phase before starting the next."
  LLM FAILURES PREVENTED:
  - Long-horizon planning failure: Phases break work into manageable chunks
  - Scope creep: Phase boundaries prevent "while we're at it" additions
  - Working memory: Exit criteria force verification at natural breakpoints
-->

> **Rule: Complete each phase before starting the next.** Phase N+1 assumes Phase N is solid and tested.

### Phase 1: {{Phase Name}} (MVP)

**Goal:** {{One sentence — what does "done" look like?}}

- [ ] {{Task 1}}
- [ ] {{Task 2}}
- [ ] {{Task 3}}

**Phase 1 exit criteria:**
- [ ] {{Criterion 1 — measurable, not vague}}
- [ ] {{Criterion 2}}

### Phase 2: {{Phase Name}}

**Goal:** {{One sentence}}

- [ ] {{Tasks...}}

**Phase 2 exit criteria:**
- [ ] {{Criteria...}}

<!-- Continue for each phase -->

---

## Common Pitfalls [RECOMMENDED]

<!--
  PURPOSE: Explicit anti-patterns. What NOT to do and WHY.
  Mapped to LLM reasoning failure categories from Song et al. (2026).
  LLM FAILURES PREVENTED: ALL OF THEM — this is the direct application
  of the failure taxonomy to this specific project.
  
  Categories to cover:
  1. Compositional reasoning failures (multi-step logic bugs)
  2. Confirmation bias (first-working-approach lock-in)
  3. Working memory limitations (losing constraints over long tasks)
  4. Inverse inference (assuming success means correctness)
  5. Robustness (edge cases, empty inputs, malformed data)
  6. Proactive initiative (unauthorized additions)
-->

### Compositional Reasoning
1. **{{Pitfall}}** — {{Why it's dangerous. Concrete example.}}

### Confirmation Bias
2. **{{Pitfall}}** — {{Description}}

### Working Memory
3. **{{Pitfall}}** — {{Description}}

### Inverse Inference
4. **{{Pitfall}}** — {{Description}}

### Robustness
5. **{{Pitfall}}** — {{Description}}

### Scope Creep
6. **{{Pitfall}}** — {{Description}}

---

## Quick Reference [REQUIRED]

<!--
  PURPOSE: One-glance summary table. The AI's "cheat sheet."
  LLM FAILURE PREVENTED: Working memory — after generating 1000 lines,
  the AI can glance here to remember core decisions.
-->

| Component | Technology | Why This, Not That |
|---|---|---|
| {{Component 1}} | {{Tech}} | {{One-line rationale}} |
| {{Component 2}} | {{Tech}} | {{One-line rationale}} |
| ... | ... | ... |
```

---

## Template Usage Notes

### Scaling Guidance

**For small projects (~100 lines):** Keep only [REQUIRED] sections. Merge Constraints into a shorter list. Skip Interfaces (define inline). Collapse Pitfalls into 3-4 bullet points.

**For medium projects (~300-500 lines):** Include all [REQUIRED] and [RECOMMENDED] sections. Full ADRs, full Pitfalls, code examples for critical patterns.

**For large projects (~500+ lines):** Full template. Consider splitting into CLAUDE.md (main) + ARCHITECTURE.md (deep technical) + API.md (integration details), with CLAUDE.md referencing the others.

### Language-Specific Adaptations

**Go:** Use `type X interface{}` + `var _ X = (*Y)(nil)` compile-time checks. Error handling emphasis.

**TypeScript:** Use `interface X {}` + type narrowing examples. Show strict mode patterns.

**Python:** Use `Protocol` classes or `ABC`. Show type hints. Emphasize explicit over implicit.

**Swift:** Use `protocol X {}`. Show `@MainActor` patterns for concurrency.

**Rust:** Use `trait X {}`. Show `Result<T, E>` patterns. Ownership/lifetime guidance.
