# MCP Server Anti-Patterns

Surface these to the user once the deployment model is chosen in Phase 2. The common ones apply to every server. The deployment-specific ones apply only to the chosen path.

## Common (always surface)

**Tool sprawl.** Do not create one tool per API endpoint. Consolidate commonly chained operations. Merge similar tools that differ by one parameter.

**Raw API responses.** Do not return unfiltered API payloads. Strip UUIDs, MIME types, and low-level identifiers the LLM cannot act on. Return semantic names and high-signal data. Truncate large result sets with a message: "Showing 10 of 847 results. Refine the query."

**Behavioral instructions in descriptions.** Do not put "always do X", "you must call Y first", or product promotion in tool descriptions. The Anthropic Directory treats this as prompt injection and rejects it. Cross-tool behavioral guidance belongs in the `instructions` field on the server.

**Skipping capability checks.** Before using sampling, roots, or any advanced feature: check `clientCapabilities` after connection. The SDK throws `CapabilityNotSupported` at runtime if the client lacks support. Every capability-dependent code path needs a guard and a working fallback. Different hosts implement different capability subsets. A server tested only against one host will crash on another.

## Remote HTTP (surface only when remote is chosen)

**Token passthrough.** Never accept a token and forward it to a downstream API. The MCP authorization spec explicitly forbids this. If your server calls another service, exchange the token or use server-own credentials. A token issued for `api.other-service.com` must be rejected even if the signature checks out.

**Wildcard scopes.** Do not define catch-all scopes (`*`, `all`, `full-access`). Do not bundle unrelated privileges into one scope. Broad scopes expand blast radius on token compromise, increase revocation friction, and obscure audit trails. Start with minimal scopes (e.g., `mcp:tools-read`), escalate with targeted challenges.

**Hand-rolling auth or transport.** The SDK ships `mcpAuthMetadataRouter()`, `requireBearerAuth()`, and transport handlers. Use them. Hand-rolling token validation, OAuth flows, or JSON-RPC framing introduces security bugs and falls behind spec changes.

**Uncontrolled DCR.** Dynamic Client Registration is not recommended (use CIMD instead). If DCR is enabled for backward compatibility, restrict to trusted hosts, audit registrations, and apply rate limits. Unauthenticated DCR means anyone can register a client with your authorization server. The confused deputy attack specifically exploits DCR + static client IDs + consent cookies.

**Monolithic servers.** Do not combine database access, file operations, API calls, and email in one server. Each server should have one clear purpose. Isolated failures, independent scaling, clear ownership.

## Local stdio (surface only when local is chosen)

**Local stdio for distribution.** Do not distribute local stdio servers to end users (reasons in Phase 2 of SKILL.md). MCPB (bundled local servers with packaged runtime) is the sanctioned way to ship local servers when local execution is required, but this skill does not yet cover MCPB packaging.
