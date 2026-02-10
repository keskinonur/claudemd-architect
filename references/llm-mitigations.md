# LLM Reasoning Failure Mitigations

Based on: Song, Han & Goodman, "Large Language Model Reasoning Failures", TMLR 2026.
Applied to: CLAUDE.md document design for AI-assisted software development.

## Why This Matters

When an LLM reads a CLAUDE.md and generates code, it is performing **compositional reasoning** over a long context window. Every known failure mode in the literature can manifest:

- The model forgets a constraint stated 400 lines ago → **working memory limitation**
- The model assumes its first approach is correct → **confirmation bias**
- The model adds features not requested → **proactive initiative**
- The model handles the happy path but not errors → **inverse inference**
- The model breaks when input is empty/malformed → **robustness failure**
- The model loses coherence across multi-step logic → **compositional reasoning failure**

Each section of the CLAUDE.md template is designed to prevent specific failure modes.

---

## Failure Mode → Mitigation Map

### 1. Working Memory Limitations

**What happens:** The LLM "forgets" instructions from earlier in the document as it generates code, especially during long implementation sessions.

**CLAUDE.md mitigations:**
| Section | How It Helps |
|---|---|
| AI Reading Guide (top) | First thing read, sets behavioral frame before any code generation |
| Critical Constraints | Short, scannable MUST/MUST NOT list — easy to re-check |
| Verification Checkpoints | Force the model to stop and re-verify at phase boundaries |
| Quick Reference table | One-glance summary at document end — works as "reminder" during long tasks |
| Phase exit criteria | Natural breakpoints where accumulated drift gets caught |

**Writing guidance:**
- Keep constraints SHORT. One line each. The model can scan a 10-item list faster than parsing 3 paragraphs.
- Repeat critical constraints in relevant ADRs. "As stated in constraints, MUST NOT use external databases" in the database ADR reinforces the rule.
- Place Quick Reference at the BOTTOM — it's the last thing the model sees before generating, so it stays in recent context.

### 2. Confirmation Bias / Path Dependency

**What happens:** Once the LLM generates code following one approach, it's reluctant to abandon that approach even when problems appear. It will rationalize issues rather than reconsider.

**CLAUDE.md mitigations:**
| Section | How It Helps |
|---|---|
| ADR "Rejected alternatives" | Pre-closes paths the model might otherwise explore |
| MUST NOT rules | Hard boundaries that can't be rationalized away |
| "Common Pitfalls" section | Explicit "this will seem right but is wrong" warnings |

**Writing guidance:**
- Always include REJECTED alternatives in ADRs. "We rejected Python because..." prevents the model from rediscovering Python and finding it appealing.
- Frame pitfalls as "This will SEEM correct but is WRONG because..." — directly addresses the bias mechanism.
- Use strong language: "Do not," "Never," "This is a bug" — not "try to avoid," "prefer not to," "ideally."

### 3. Proactive Initiative / Scope Creep

**What happens:** The LLM adds features, abstractions, frameworks, or "improvements" that were never requested. It tries to be helpful by anticipating needs.

**CLAUDE.md mitigations:**
| Section | How It Helps |
|---|---|
| MUST NOT rules | "MUST NOT add React" is unambiguous |
| Phase boundaries | "Do NOT implement Phase 3 features while in Phase 1" |
| "Ask the developer" guidance | Redirects initiative to the human |
| Pitfall: "Don't add while-we're-at-it features" | Directly names the behavior |

**Writing guidance:**
- The MUST NOT section should explicitly list common "helpful" additions the model might make: ORMs, caching layers, logging frameworks, test utilities, CI/CD configs.
- Include: "If a technology/pattern is not mentioned in this document, do not introduce it. Ask the developer first."
- Phase-gating is the strongest defense: it converts a nebulous "don't do too much" into a concrete "these specific tasks only."

### 4. Compositional Reasoning Failures

**What happens:** When combining multiple concepts (auth + pagination + error handling + throttling), the LLM drops or incorrectly implements one of the components.

**CLAUDE.md mitigations:**
| Section | How It Helps |
|---|---|
| Concrete code examples | Show the EXACT pattern for complex compositions |
| Interface definitions | Define contracts explicitly so compositions have clear boundaries |
| Integration Notes | API-specific patterns with real code, not abstract descriptions |

**Writing guidance:**
- For any pattern that combines 3+ concerns (e.g., "retry with backoff AND respect Retry-After header AND log attempts AND support context cancellation"), provide the COMPLETE code example. Don't describe it in prose.
- For API integrations, always show: auth, pagination, error handling, and rate limiting in a single code block. These are the four components that compositionally fail when described separately.
- Interface definitions should include the actual method signatures. "An embedder interface" is vague. `Embed(ctx, text) ([]float32, error)` is unambiguous.

### 5. Inverse Inference / Verification Failures

**What happens:** The LLM assumes that because code compiles or runs without errors, it is correct. "It works for my test case" → "It works for all cases" is a common false inference.

**CLAUDE.md mitigations:**
| Section | How It Helps |
|---|---|
| Phase exit criteria | "Verify X, Y, Z" — not just "complete tasks" |
| Pitfall: "Test error paths, not just happy paths" | Directly counters the inference |
| Pitfall: "Test restore, not just backup" | Application-specific inverse inference |
| Verification Checkpoints | "Before moving on, verify..." |

**Writing guidance:**
- Exit criteria should include NEGATIVE tests: "Verify that invalid input is rejected," "Verify that the system handles network timeout."
- Include at least one "the thing that seems to work but actually doesn't" example specific to the project. E.g., "A backup that only captures the first page of API results appears to work but is silently incomplete."
- Phrase criteria as "X is verified" not "X is implemented." Implementation ≠ correctness.

### 6. Robustness / Sensitivity to Input

**What happens:** Code works for typical inputs but fails for edge cases: empty strings, null values, zero-length arrays, non-UTF-8 encoding, extremely large inputs, concurrent access.

**CLAUDE.md mitigations:**
| Section | How It Helps |
|---|---|
| Pitfalls: explicit edge cases | "Users with zero emails must not crash the system" |
| Constraints: "MUST handle" | Make robustness a constraint, not an afterthought |
| Interface definitions | `error` return types force error consideration |

**Writing guidance:**
- In the Pitfalls section, list CONCRETE edge cases specific to the project. Don't say "handle edge cases." Say "Handle: empty mailbox, zero-byte attachment, filename with non-UTF-8 characters, email with no body."
- For each external integration, identify the "empty response" case. What happens when the API returns zero results? An empty array? A null?
- If the project involves iteration/pagination, explicitly state: "Continue until termination condition, never assume a fixed number of items."

---

## The Five-Layer Defense Model

These mitigations work in layers. No single layer is sufficient, but together they create defense in depth:

```
Layer 1: AI Reading Guide    → Sets behavioral expectations BEFORE code generation
Layer 2: Constraints          → Hard boundaries that can't be rationalized away
Layer 3: ADRs with rationale  → Anchors reasoning to documented decisions
Layer 4: Code examples        → Reduces composition errors with concrete patterns
Layer 5: Pitfalls + Checkpoints → Catches failures that slip through layers 1-4
```

**The key insight:** Layers 1-3 are PREVENTIVE (stop the LLM from making the mistake). Layers 4-5 are DETECTIVE (catch mistakes that happen anyway). Both are needed because no prevention is perfect.

---

## Applying This to Your Project

When generating the Common Pitfalls section, walk through each failure category and ask:

1. **Compositional:** "What multi-step operations could lose a step?" (API calls with pagination + auth + retry)
2. **Confirmation bias:** "What incorrect first approach might seem right?" (Using a simple loop instead of pagination)
3. **Working memory:** "What constraint might be forgotten during long implementation?" (Error handling, MUST NOT rules)
4. **Inverse inference:** "What could appear to work but be silently wrong?" (Backup that captures first page only)
5. **Robustness:** "What inputs could be empty, null, malformed, or extreme?" (Zero emails, no body, huge attachments)
6. **Scope creep:** "What might the model add 'helpfully' that wasn't asked for?" (Caching, ORMs, extra endpoints)

Generate 2-3 specific pitfalls per category, tailored to the project's domain.
