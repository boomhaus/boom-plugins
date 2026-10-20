# Placement rules

Reference content for `update-context`. Load during Phase 3 (rule extraction) and Phase 4 (placement decisions).

## Litmus test

For every candidate rule, ask: **would removing this rule cause Claude to make a specific, named mistake?**

- If yes, the rule passes.
- If you cannot name the mistake concretely, drop the rule.

Inverse check: would a competent engineer figure this out by reading the surrounding code in five minutes? If yes, drop the rule. Code self-documents; CLAUDE.md captures only what code cannot.

Examples:

| Candidate | Passes? | Reason |
|---|---|---|
| "All mutations must go through `withTransaction()`" | Yes | Forgetting it causes silent data corruption Claude has hit before. |
| "Use TypeScript strict mode" | No | Visible in `tsconfig.json`. |
| "Write clean code" | No | No specific mistake gets prevented. |
| "Charge handlers must reject on idempotency-key collision before any side effects" | Yes | Specific mistake (double-charge) is named. |

## Scope rubric

Choose the smallest scope that captures the rule's applicability. The rubric below is a guide for *what kind* of content typically lives at each level, not a constraint.

### Root (`CLAUDE.md` at repo root)

- Architectural map: what services/packages exist and how they relate (one paragraph).
- Cross-cutting invariants that apply to every package (auth model, error handling discipline).
- Pointers to nested context files for known hotspots.
- Hard cap: ~200 lines.

### Package / service (one or two directories down in a monorepo)

- Scoped commands: test, lint, build, dev-server for this package only.
- What this package owns; what it must not own (responsibility boundaries).
- Framework-specific gotchas (e.g., "Next.js 15 app router, no `getServerSideProps`").
- Public-vs-internal API surface.

### Module (feature directories, domain modules)

- Local naming/structural conventions ("handlers in `routes/`, logic in `services/`, never the reverse").
- Domain vocabulary glossary for bounded contexts.
- Local invariants ("every mutation goes through `withTransaction()`").
- Pointers to canonical example files.

### Leaf (only when warranted)

- "This directory is generated from `schema.proto`. Edit the proto, run `make gen`."
- "Runs in a Cloudflare Worker -- no Node APIs, no top-level await."
- Security-sensitive code with non-obvious review requirements.

## Smallest-scope heuristic

Walk the touched-files list. Find the **lowest common ancestor directory** that contains every file the rule applies to. That is the rule's scope.

- Rule applies to files in `services/payments/lib/` only -> scope is `services/payments/lib`.
- Rule applies to files in `services/payments/lib/` and `services/payments/handlers/` -> scope is `services/payments`.
- Rule applies to files in `services/payments/` and `services/billing/` -> scope is `services`.
- Rule applies across the whole repo -> scope is the root.

Do not place a rule at root just because it "might apply elsewhere someday." Place it where the evidence sits. If the rule later proves universal, it can move up.

## CLAUDE.md vs AGENTS.md selection

`context-locator` returns both kinds of files at each scope. Selection logic:

1. If the chosen scope has only `CLAUDE.md` -> propose updating `CLAUDE.md`.
2. If the chosen scope has only `AGENTS.md` -> propose updating `AGENTS.md`.
3. If the chosen scope has both -> propose updating `CLAUDE.md` (default for new content; user can move it manually).
4. If the chosen scope has neither -> propose creating `CLAUDE.md` at that scope.

The rule content itself is identical in either file kind. Only the filename and tool affinity differ.

## Anti-patterns

Drop candidate rules that match these patterns. They corrode context-file value over time.

| Anti-pattern | Why drop |
|---|---|
| Restates content already in code (type signatures, config files, lint rules) | Code is the source of truth. Duplication invites drift. |
| Aspirational rule the team does not actually enforce | Claude follows the rule, users ignore it, friction follows. |
| Tutorial prose ("To run tests, first install dependencies, then...") | Context files are rules, not onboarding docs. |
| Generic coding advice ("write small functions", "avoid magic numbers") | Applies to every codebase. Not context. |
| Style preference enforced by a formatter | The formatter is the enforcement layer. |
| Rule that paraphrases another rule already in the same file | Pure noise. |
| Rule whose scope is "everywhere" but evidence is from one file | Premature generalization. Place at the evidence's scope. |

## Rule extraction signals

When extracting candidate rules from the conversation in Phase 3, look for these signals.

### Strong signals (extract as candidates)

- **Repeated correction.** The user corrected Claude on the same thing twice or more.
- **Explicit named convention.** "We always do X here." "We never use Y." "Our pattern is Z."
- **Rejected approach with reason.** Claude proposed something; the user rejected it and explained why.
- **Explicit save request.** "Remember this." "Add this to CLAUDE.md." "Don't forget X next time."
- **Discovered invariant.** A non-obvious property (security, performance, ordering, transactionality) was named or violated during debugging.
- **Recurring error.** Claude hit the same class of error twice and the resolution was the same.

### Weak signals (do not extract)

- **One-off context.** Information specific to the immediate task that won't recur.
- **Public knowledge.** Framework conventions documented in the framework's own docs.
- **Visible-in-code facts.** Anything a reader of the file would learn from reading the file.
- **Aspirational statements.** "We should..." without enforcement evidence.

### Phrasing for extracted rules

Phrase each candidate as a **terse imperative**, not as a narrative explanation.

- Good: "Charge handlers must reject on idempotency-key collision before any side effects."
- Bad: "It's important that when you implement a charge handler, you should think carefully about idempotency, because we discovered a bug where..."

The justification field captures the reasoning. The rule itself is the directive only.
