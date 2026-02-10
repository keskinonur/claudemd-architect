# Interview Questionnaire

Use this to extract architectural decisions from the developer. Adapt questions based on answers — skip irrelevant sections, dive deeper where needed.

## Round 1: Project Identity (ask first, always)

1. **What does this project do in one sentence?**
   - Forces clarity. If they can't say it in one sentence, the vision isn't clear yet.

2. **Who is the user?** (developer tool? internal team? end users? API consumers?)
   - Determines UI needs, auth model, error handling philosophy.

3. **What's the deployment target?** (local machine, Docker, cloud, NAS, mobile, embedded?)
   - Constrains technology choices more than any other factor.

4. **Is this greenfield or extending something existing?**
   - If extending: what exists, what are the constraints of the existing system?

5. **Solo developer or team?**
   - Solo: CLAUDE.md can be more opinionated. Team: needs to be more explanatory.

## Round 2: Tech Stack Decisions (ask for DECISIONS, not preferences)

6. **What programming language(s) will this use? Why?**
   - If they say "I'm thinking about X or Y" — help them decide NOW. The CLAUDE.md needs a definitive answer.

7. **What framework(s) for the backend? Why?**
   - Follow up: "What did you consider and reject?"

8. **What framework(s) for the frontend? Why?**
   - Or: "Is there a frontend? What kind?"

9. **What database(s)? Why?**
   - Follow up: "How much data? What access patterns?"

10. **What's the API style?** (REST, GraphQL, gRPC, none?)

11. **Any external services or APIs this integrates with?**
    - For each: auth method, rate limits, critical gotchas.

## Round 3: Constraints & Non-Negotiables

12. **What are the absolute constraints?** (compliance, performance, budget, platform)
    - Example prompts: "Is there a data residency requirement?" "Must it run offline?" "Any regulatory requirements?"

13. **What should this project NEVER do or include?**
    - This becomes the MUST NOT section. Push for specifics: "No React" is better than "keep it simple."

14. **What existing patterns or conventions must be followed?**
    - Coding standards, naming conventions, directory structures, CI/CD requirements.

## Round 4: Architecture & Data Model

15. **What are the main components/modules of the system?**
    - Draw out the mental model. "If you were to draw boxes on a whiteboard, what are they?"

16. **How do these components communicate?**
    - HTTP? Channels? Events? Direct function calls?

17. **What's the data model?** (key entities, relationships)
    - Even rough: "Users have Projects, Projects have Files, Files have Versions"

18. **What are the key interfaces/boundaries?**
    - Where should implementations be swappable? What might change in the future?

## Round 5: Deployment & Infrastructure

19. **How is it deployed?** (Docker, bare metal, serverless, app store?)

20. **What's the production environment?** (specs, OS, network constraints)

21. **Any sidecar services?** (databases, caches, message queues, AI models)

22. **How is configuration managed?** (env vars, config files, UI, flags?)

## Round 6: Development Phasing

23. **What's the MVP?** (minimum viable product — the smallest thing that proves the concept works)

24. **What comes after MVP?** (Phase 2, 3, etc.)

25. **What's explicitly OUT of scope for v1?**
    - This prevents scope creep. If they mention "maybe later" features, capture them as future phases.

## Round 7: Context & History (if applicable)

26. **Have you had a conversation about this project already?** (with Claude or others)
    - If yes: "Can you share the transcript or key decisions?"

27. **Are there existing documents?** (PRD, design doc, API spec, wireframes)
    - If yes: read them and extract decisions rather than re-asking.

28. **What inspired this project?** (similar tools, competitor weaknesses, specific pain point)
    - Becomes context in the ADRs and influences the "Common Pitfalls" section.

---

## Interview Technique Notes

**Don't ask all questions sequentially.** Start with Round 1, then let the conversation flow naturally. Circle back to missed areas.

**Push for closure.** If the developer says "I'm leaning toward X but maybe Y," help them decide. A CLAUDE.md with "maybe" is worse than useless — it creates ambiguity that the LLM will fill with its own judgment.

**Capture rejected alternatives.** Every "we considered X but chose Y because Z" becomes an ADR. These are gold — they prevent the LLM from re-discovering and re-proposing X.

**Listen for implicit decisions.** If they say "it runs on a NAS," that implicitly means: no GPU, limited RAM, must be lightweight, probably Docker. Surface these implications.

**Ask "what could go wrong?"** The answers become the Common Pitfalls section and inform error handling patterns.
