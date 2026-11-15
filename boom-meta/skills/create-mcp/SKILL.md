---
name: create-mcp
description: Build MCP servers from use case through deployment. Interrogates the use case, picks deployment model (remote HTTP or local stdio), selects tool design pattern, picks framework, scaffolds the server, and guides through testing.
when_to_use: Use when the user wants to build an MCP server, create an MCP integration, wrap an API for Claude, expose tools to an LLM, connect a service to Claude Desktop, make a Claude connector, or discusses anything involving the Model Context Protocol.
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
---

# Build an MCP Server

Guide the developer through designing and building an MCP server. MCP servers come in different shapes. Picking the wrong one early causes painful rewrites. Discovery comes before code.

Do not start scaffolding until Phase 1 questions are answered. If the user's opening message already answers them, acknowledge that and skip to the recommendation.

---

## Phase 0: Resume and route

> **Entry:** Skill loaded. **Exit:** Intent extracted, routed to the correct phase.

Check for existing MCP server work in the project before asking anything:

1. **Resume detection.** Search the working directory for `McpServer`, `FastMCP`, `wrangler.toml` with MCP bindings, or `@modelcontextprotocol/sdk` in package.json. If found, summarize what exists and ask the user what phase they are at: extending tools, fixing a bug, adding auth, or testing. Route to the appropriate phase.
2. **Scope assessment.** Read the user's opening message. Extract any answers to Phase 1 questions that are already provided (deployment target, audience, tool count, auth, content types). Mark those questions as answered.
3. **Route.** If all Phase 1 questions are already answered, skip to Phase 2. If partial, proceed to Phase 1 with only the unanswered questions. If the user's message is vague ("build me an MCP server"), proceed to Phase 1 in full.

---

## Phase 1: Interrogate the use case

> **Entry:** No existing MCP server detected, or user starting fresh. **Exit:** All 5 questions answered.

Ask only unanswered questions in one message. Batch them. Do not interrogate one at a time. Adapt wording to what the user has already said.

### 1. What does it connect to?

| Connects to | Direction |
|---|---|
| Cloud API (SaaS, REST, GraphQL) | Remote HTTP server |
| Local process, filesystem, desktop app | Local stdio |
| Hardware, OS-level APIs | Local stdio |
| Nothing external (pure logic) | Either; default to remote |

### 2. Who will use it?

| Audience | Direction |
|---|---|
| Just me / my team | Local stdio is acceptable |
| Anyone who installs it | Remote HTTP (strongly preferred) |
| Broad distribution | Remote HTTP |

### 3. How many distinct actions?

This determines the tool design pattern in Phase 3.

| Count | Pattern |
|---|---|
| Under ~15 | One tool per action |
| 15-30 | Audit for near-duplicates, one-per-action if clean |
| 30+ | Search + execute |

The count is provisional; Phase 3 challenges the list before it is final.

### 4. What auth does the upstream service use?

| Auth type | Approach |
|---|---|
| None / API key | Read from env var, done |
| OAuth 2.0 | Remote server with CIMD auth; see `references/auth-cimd.md` |

### 5. Does the server need to return rich content?

| Content | Approach |
|---|---|
| Text / JSON only | Standard text content blocks |
| Images, audio, diagrams | Use typed content blocks (image, audio); see `references/tool-design.md` |
| Structured typed output | Use outputSchema + structuredContent with text fallback |

---

## Phase 2: Recommend a deployment model

> **Entry:** All Phase 1 questions answered. **Exit:** Deployment model chosen by user, relevant anti-patterns surfaced.

Based on answers, recommend **one** path. Be opinionated.

### Remote streamable-HTTP server (default recommendation)

A hosted service speaking MCP over streamable HTTP. Recommended for anything wrapping a cloud API.

Why it wins:
- Zero install friction for users. They add a URL.
- One deployment serves all users. You control upgrades.
- OAuth flows work properly. The server handles redirects and token storage.
- Works across Claude Desktop, Claude Code, claude.ai, and third-party MCP hosts.

Choose this unless the server must touch the user's local machine.

Fastest deploy: Cloudflare Workers. See `references/cloudflare-workers.md`.
Portable Node/Python: See `references/remote-http-scaffold.md`.

### Local stdio (prototype only)

A script launched via npx/uvx on the user's machine. Acceptable for personal tools and prototypes.

**Not recommended for distribution.** Users need the right runtime installed. No push updates. Fragile across environments. If the user intends to distribute, push toward remote HTTP.

See `references/local-scaffold.md`.

**After the user chooses a deployment model, read `references/anti-patterns.md` and surface the common anti-patterns plus the ones for the chosen path.**

---

## Phase 3: Pick a tool design pattern

> **Entry:** Deployment model chosen. **Exit:** Tool pattern selected, user has confirmed tool list.

**Constraint: push back on tool count.** Every tool schema consumes context tokens on every turn; thirty tools with rich schemas can eat 3-5k tokens before the conversation starts. Before accepting the user's list, question whether each tool carries its own weight: could it merge with an existing tool, is it a variant that differs by one parameter, are two tools always called in sequence? If the user cannot name a case where Claude would pick this tool over another, it probably should not be separate. Make the case for consolidation, then defer to the user.

### Pattern A: One tool per action (small surface, default)

When the action space is small (under ~15 operations), give each a dedicated tool with a tight description and schema.

```
create_issue    -- Create a new issue. Params: title, body, labels[]
update_issue    -- Update an existing issue. Params: id, title?, body?, state?
search_issues   -- Search issues by query string. Params: query, limit?
add_comment     -- Add a comment to an issue. Params: issue_id, body
```

Each tool's schema validates inputs precisely. No discovery round-trips.

### Pattern B: Search + execute (large surface)

When wrapping a large API (30+ endpoints), listing every operation floods the context. Expose two tools:

```
search_actions  -- Given a natural-language intent, return matching actions
                   with their IDs, descriptions, and parameter schemas.
execute_action  -- Run an action by ID with a params object.
```

The server holds the full catalog internally. Claude searches, picks, executes. Context stays lean.

### Hybrid

Promote the 3-5 most-used actions to dedicated tools. Keep the long tail behind search/execute.

See `references/tool-design.md` for schema examples, description writing, and annotation requirements.

---

## Phase 4: Pick a framework

> **Entry:** Tool pattern selected. **Exit:** Framework chosen.

| Framework | Language | Use when |
|---|---|---|
| Official TypeScript SDK (`@modelcontextprotocol/sdk`) | TS/JS | Default choice. Best spec coverage, first to get new features. |
| FastMCP (`fastmcp` on PyPI) | Python | User prefers Python or is wrapping a Python library. Decorator-based, low boilerplate. |

**Do not use the frozen FastMCP copy bundled in the official `mcp` Python SDK.** Install jlowin's standalone `fastmcp` package from PyPI.

**Versions.** Always use the latest stable release of the chosen SDK and target the latest stable MCP spec revision. Check the current docs or package registry for both before scaffolding; do not rely on a remembered version. Function names and snippets in the reference files may have changed since they were written, so confirm them against the installed SDK.

If the user already has a language preference, go with it. Both produce identical wire protocol.

---

## Phase 5: Scaffold and configure

> **Entry:** Framework chosen. **Exit:** Server scaffolded, all tools registered with required annotations.

**Constraint: every tool MUST have `readOnlyHint`, `destructiveHint`, and `title` annotations.** Read and write operations MUST be separate tools. Every parameter MUST have `.describe()`. These are non-negotiable for Anthropic Directory acceptance.

Once deployment model, tool pattern, and framework are settled, scaffold the server.

### Scaffold selection

| Decision | Reference |
|---|---|
| Remote HTTP, TypeScript | `references/remote-http-scaffold.md` |
| Remote HTTP, Python | `references/remote-http-scaffold.md` |
| Remote HTTP, Cloudflare Workers | `references/cloudflare-workers.md` |
| Local stdio, TypeScript | `references/local-scaffold.md` |
| Local stdio, Python | `references/local-scaffold.md` |

### Mandatory for every tool

Apply these requirements from `references/tool-design.md` to every tool registered:

- `readOnlyHint`, `destructiveHint`, and `title` annotations on every tool.
- Read and write operations in **separate** tools. A single tool handling both GET and POST/PUT/DELETE is rejected by the Anthropic Directory.
- Tight parameter schemas with `.describe()` on every parameter. Use enums, regex, min/max, defaults.
- Disambiguating descriptions. Say what the tool does and what it does NOT do.
- Recovery hints in error responses: `"Item not found. Use search_items to find valid IDs."`

### Server-level features

See `references/server-capabilities.md` for details on each.

- **`instructions` field**: Set on the server for cross-tool behavioral guidance. This is where "always call search before get" belongs. Not in individual tool descriptions.
- **Sampling**: Delegate LLM calls to the host instead of shipping your own model client. Check `clientCapabilities.sampling` first.
- **Progress reporting**: For long-running tools, emit progress notifications with `progress`, `total`, and `message`. Check if `progressToken` was provided.
- **Resource templates**: For parameterized data access (e.g., `weather://forecast/{city}/{date}`). Support parameter completion.

### Auth

If OAuth is needed, see `references/auth-cimd.md`. Use CIMD (Client ID Metadata Document), the auth mechanism the MCP spec prefers.

---

## Phase 6: Test and validate

> **Entry:** Server scaffolded. **Exit:** All three testing layers completed, testing checklist passed.

See `references/testing.md` for the full testing workflow.

Three layers, in order:

1. **Headless JSON-RPC**: Feed raw JSON-RPC messages into the server. Parse responses. Fast iteration for protocol correctness.
2. **MCP Inspector**: Official dev tool. Interactive verification of tools, resources, prompts. Catches capability mismatches and malformed responses.
3. **Real Claude**: Connect the server to a real host. Remote: add it as a custom connector at claude.ai. Local stdio: register it with `claude mcp add <name> -- <command>` in Claude Code, or in Claude Desktop's config. This catches what unit tests cannot: tool selection accuracy, description ambiguity, parameter inference failures.

Do not skip layer 3. Type checking and test suites verify code correctness, not feature correctness. A tool can pass every unit test and still be unusable because Claude never selects it.

---

## Beyond tools: the other primitives

Tools are one of three server primitives. Most servers start with tools and never need the others. Knowing they exist prevents reinventing them.

| Primitive | Who triggers it | Use when |
|---|---|---|
| **Resources** | Host application (not the LLM) | Exposing docs, files, data as browsable context. Read-only. URI-based. |
| **Prompts** | User (slash command, command palette) | Canned workflows ("/summarize-thread"). Parameterized templates. |

Resources are application-controlled. The host decides how to discover, retrieve, and present them. Each resource has a URI and MIME type. Resource templates support dynamic URIs with parameter completion.

Prompts are user-controlled. They require explicit invocation. Context-aware: they can reference available resources and tools.

See `references/server-capabilities.md` for implementation details.

---

## Reference files

- `references/tool-design.md` -- tool descriptions, schemas, annotations, return shapes, output size limits (`anthropic/maxResultSizeChars`), error handling, content types
- `references/auth-cimd.md` -- CIMD auth flow, token validation, audience checking, scope design
- `references/remote-http-scaffold.md` -- remote server scaffolds in TypeScript SDK and FastMCP Python
- `references/local-scaffold.md` -- local stdio scaffolds in TypeScript SDK and FastMCP Python
- `references/server-capabilities.md` -- instructions, sampling, progress, resource templates, capability guards
- `references/testing.md` -- headless JSON-RPC, MCP Inspector, real Claude testing
- `references/cloudflare-workers.md` -- Cloudflare Workers deployment scaffold
- `references/anti-patterns.md` -- common, remote-only, and local-only anti-patterns to surface after Phase 2
