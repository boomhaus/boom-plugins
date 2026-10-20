# Proposal schema

Reference content for `update-context`. Load during Phase 5 (emit output).

The skill emits exactly one JSON envelope per invocation. The envelope conforms to the schema below. A human-readable summary follows the JSON block in the same response.

## Envelope schema

```json
{
  "schema_version": 1,
  "proposals": [
    {
      "target_file": "string (repo-relative path)",
      "scope": "string (repo-relative directory, empty string for root)",
      "kind": "claude-md | agents-md",
      "action": "append | create",
      "rule": "string (terse imperative)",
      "justification": "string (one sentence)",
      "confidence": "high | medium | low"
    }
  ]
}
```

## Field reference

### Envelope fields

| Field | Type | Required | Description |
|---|---|---|---|
| `schema_version` | integer | yes | Currently `1`. Bumped on breaking schema changes. |
| `proposals` | array | yes | Array of proposal objects. May be empty. |

### Proposal fields

| Field | Type | Required | Description |
|---|---|---|---|
| `target_file` | string | yes | Repo-relative path to the file the rule should land in. Includes the filename. |
| `scope` | string | yes | Repo-relative directory containing `target_file`. Empty string for repo root. |
| `kind` | enum | yes | `"claude-md"` or `"agents-md"`. Must match the filename in `target_file`. |
| `action` | enum | yes | `"append"` if `target_file` exists, `"create"` if it does not. |
| `rule` | string | yes | The rule itself, phrased as a terse imperative. No leading bullets or markdown. |
| `justification` | string | yes | One sentence pointing to the conversation evidence that motivated the rule. |
| `confidence` | enum | yes | `"high"` / `"medium"` / `"low"`. Reflects strength of evidence. |

## Validation rules

The model must self-enforce these before emitting. If any check fails, fix the proposal or drop it; do not emit invalid output.

### Structural

- The JSON must parse. No trailing commas, no comments, no unquoted keys.
- Every required field must be present and non-empty (except `scope` for root, which is `""`).
- Enum fields accept only the listed literal strings. Lowercase, hyphenated where shown.

### Cross-field consistency

- `target_file` must end with `/CLAUDE.md` or `/AGENTS.md` (or just `CLAUDE.md` / `AGENTS.md` at root).
- `kind: "claude-md"` <-> `target_file` ends with `CLAUDE.md`.
- `kind: "agents-md"` <-> `target_file` ends with `AGENTS.md`.
- `scope` is the directory containing `target_file`. For `scope: "services/payments"`, `target_file` must be `services/payments/CLAUDE.md` or `services/payments/AGENTS.md`.
- For `scope: ""`, `target_file` must be `CLAUDE.md` or `AGENTS.md`.

### Content

- `rule` is a single imperative sentence. No "you should", "we try to", "it would be nice if". Use "must", "never", "always", or bare imperatives ("Reject X before Y").
- `rule` is one sentence. If you need two, you have two rules -- split them into separate proposals.
- `justification` references the session evidence, not the rule itself. Bad: "Because the rule says X." Good: "Claude double-charged in two attempts before the user enforced idempotency manually."
- `confidence`:
  - `high` -- multiple corrections / explicit user instruction / named convention.
  - `medium` -- single correction / strong inferred pattern.
  - `low` -- one-off signal that might still be worth surfacing.

### Empty proposals

When no candidate rule survives Phases 3-4, emit:

```json
{
  "schema_version": 1,
  "proposals": []
}
```

Followed by a summary line: `No proposals -- no rules met the litmus test.`

## Examples

### Example 1: Single rule, existing file, high confidence

```json
{
  "schema_version": 1,
  "proposals": [
    {
      "target_file": "services/payments/CLAUDE.md",
      "scope": "services/payments",
      "kind": "claude-md",
      "action": "append",
      "rule": "All mutations must go through withTransaction() -- direct DB writes are forbidden.",
      "justification": "Claude wrote three direct DB updates; user corrected each one and explained that withTransaction wraps audit logging plus rollback.",
      "confidence": "high"
    }
  ]
}
```

### Example 2: New file, mixed kinds

```json
{
  "schema_version": 1,
  "proposals": [
    {
      "target_file": "workers/edge/CLAUDE.md",
      "scope": "workers/edge",
      "kind": "claude-md",
      "action": "create",
      "rule": "Code in this directory runs in a Cloudflare Worker -- no Node APIs, no top-level await.",
      "justification": "Claude used fs.readFileSync; user explained the Worker runtime constraint and pointed to a deployment failure.",
      "confidence": "high"
    },
    {
      "target_file": "AGENTS.md",
      "scope": "",
      "kind": "agents-md",
      "action": "append",
      "rule": "Run `pnpm test:security` before any commit that touches files under services/auth/.",
      "justification": "User invoked the security test suite manually after Claude proposed a commit; explained it is a required pre-commit step.",
      "confidence": "medium"
    }
  ]
}
```

### Example 3: Empty result

```json
{
  "schema_version": 1,
  "proposals": []
}
```

Summary: `No proposals -- no rules met the litmus test.`
