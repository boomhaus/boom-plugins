---
name: update-context
description: Capture learnings from the current session as proposed updates to CLAUDE.md / AGENTS.md files at the right scope. Use when the user says "remember this", "add this to CLAUDE.md", "capture what we learned", "update context files", or finishes a debugging/refactor session and wants the lessons preserved. Discovers existing context files via the context-locator agent, then applies placement rules to decide whether each rule belongs in an existing file or warrants creating a new nested one. Outputs JSON proposals plus a human-readable summary in-session for the user to apply manually. Do NOT use for: writing new code, editing arbitrary docs, summarizing the session without intent to update context files.
disable-model-invocation: true
allowed-tools: Task Read Glob Bash(git diff:*) Bash(git status:*) Bash(git ls-files:*)
argument-hint: "[optional hints about what to capture]"
arguments: hints
---

# update-context

Capture learnings from the current session as proposed updates to `CLAUDE.md` / `AGENTS.md` files at the smallest appropriate scope. Output is a JSON proposal envelope plus a human-readable summary. The user applies proposals manually -- this skill never edits context files directly.

## Inputs

### User hints (optional)

$hints

If the user supplied hints, treat them as steering for which learnings to prioritize. If empty, infer candidate rules from the active conversation.

### Changed files (git, injected at skill load)

Tracked changes since HEAD:
```!
git diff --name-only HEAD 2>/dev/null || echo ""
```

Staged changes:
```!
git diff --name-only --cached 2>/dev/null || echo ""
```

Untracked files:
```!
git ls-files --others --exclude-standard 2>/dev/null || echo ""
```

Use these as the set of files touched in this session. If the lists are empty (no git repo, or no changes), proceed using touched files inferred from the conversation history (Edit / Write / MultiEdit tool calls).

## Procedure

Execute these phases in order. Do not skip phases.

### Phase 1: Discover changed files

Build a deduplicated list of touched files from the three git outputs above plus any Edit/Write/MultiEdit/NotebookEdit tool calls visible in the conversation. Filter out noise paths: `.claude/`, `.git/`, `node_modules/`, `dist/`, `build/`, `*.lock`, `*.log`, snapshot/fixture directories. If the filtered list is empty, emit an empty-proposals envelope and stop.

### Phase 2: Discover existing context files

Compute scopes from the touched files. For each touched path, enumerate every parent directory up to the repo root. Deduplicate across all touched files. Include the empty string (repo root) as one of the scopes.

Invoke the `context-locator` agent via the Task tool with `subagent_type: boom-engineering:context-locator`. Pass one `<scope>...</scope>` tag per directory in the prompt. The agent returns JSON with `claude_md` and `agents_md` arrays of paths.

The locator may return `.claude/CLAUDE.md` (or other paths under `.claude/`, `.git/`, etc.) in its output. Filter those out before Phase 4 -- they are tooling-internal files, not project context that user-authored rules should target.

### Phase 3: Extract candidate rules

Scan the conversation for candidate rules. A candidate rule is a piece of guidance that, if Claude had known it at the start of the session, would have prevented a specific mistake or wasted exchange. Strong signals:

- The user corrected Claude on the same thing more than once.
- Claude violated a convention the user named explicitly ("we always do X here").
- Claude proposed an approach that was rejected with a reason.
- The user explicitly said "remember this" or "add this to CLAUDE.md".
- A non-obvious invariant was discovered or reinforced (security, performance, ordering, transactionality).

Weak signals (drop these):

- General coding advice not specific to this repo.
- Restatements of things visible in the code.
- Style preferences without enforcement (formatter handles it).

Produce a list of candidate rule strings, each phrased as a terse imperative. See `references/placement-rules.md` for extraction details.

### Phase 4: Decide placement for each candidate

For each candidate rule, decide where it belongs by applying the placement heuristics from `references/placement-rules.md`. The decision produces six fields: `target_file`, `scope`, `kind`, `action`, `justification`, `confidence`.

Procedure per candidate:

1. **Litmus test.** Would removing this rule cause Claude to make a specific mistake? If no, drop the candidate.
2. **Anti-pattern check.** Is it aspirational, self-evident, or duplicating existing context? If yes, drop.
3. **Scope.** Find the smallest directory that captures the rule's applicability. Walk up from the touched files until you reach the lowest common ancestor of every file the rule applies to.
4. **Kind.** Choose `claude-md` or `agents-md` based on what already exists at the chosen scope (from Phase 2). If both exist, prefer `claude-md`. If neither, default to `claude-md`.
5. **Action.** `append` if the target file exists, `create` if not.
6. **Justification.** One sentence pointing to the conversation evidence.
7. **Confidence.** `high` / `medium` / `low` based on strength of evidence.

Drop any candidate where the litmus test or anti-pattern check eliminates it. Keep the remainder as proposal entries.

### Phase 5: Emit output

Emit the JSON envelope per `references/proposal-schema.md`, then a human-readable summary below it.

The JSON envelope is a single fenced code block with language `json`. The summary is plain markdown listing each proposal's target file and rule in one line, followed by a count by action (`append` / `create`).

## Output format

````
```json
{
  "schema_version": 1,
  "proposals": [
    {
      "target_file": "<repo-relative path>",
      "scope": "<directory>",
      "kind": "claude-md | agents-md",
      "action": "append | create",
      "rule": "<terse imperative>",
      "justification": "<one sentence pointing to evidence>",
      "confidence": "high | medium | low"
    }
  ]
}
```

## Summary

- `services/payments/CLAUDE.md` (append): All mutations must go through withTransaction()
- `services/payments/lib/CLAUDE.md` (create): Charge handlers must short-circuit on idempotency-key match

Total: 2 proposals (1 append, 1 create)
````

If `proposals` is empty, emit the envelope with an empty array and a summary line stating "No proposals -- no rules met the litmus test."

## References

- `references/placement-rules.md` -- litmus test, scope rubric, smallest-scope heuristic, CLAUDE.md vs AGENTS.md selection, anti-patterns, rule extraction signals.
- `references/proposal-schema.md` -- JSON schema, field reference, validation rules, example proposals.

Load these references only when you need them: `placement-rules.md` during Phase 3 and Phase 4, `proposal-schema.md` during Phase 5.
