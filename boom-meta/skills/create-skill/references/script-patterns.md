# Bundled Script Patterns

Read this file when there is plumbing Claude would otherwise rewrite on every
run: fetching data, parsing a format, driving a tool.

## Tool or Library

| Kind | What Claude does with it | Suits |
|---|---|---|
| **Tool** | Runs it as-is | One fixed job: validate, convert, drive a check |
| **Library** | Imports it into a script it writes for the task | Open-ended work where the questions vary but the plumbing does not: analysis, reporting |

A library lets Claude spend its turns composing an answer, not rebuilding
the fetch-and-parse code each time.

## List What Exists

Claude uses a helper only if the skill says it is there. List each one in
SKILL.md with one line on what it does. Claude reads the source when it needs
the detail.

```markdown
## Helpers

Python helpers are in `${CLAUDE_SKILL_DIR}/lib/`. Add that folder to the
import path in any script you write.

| Helper | Does |
|---|---|
| `fetch(start, end)` | Loads events for a date range |
| `by_referrer(events)` | Groups events by traffic source |
```

Here the variable is meant to be substituted: Claude needs the real path.

## Put Quirks in Docstrings

A data quirk belongs on the function that exposes it. Claude reads it at the
moment it calls the function.

```python
def fetch(start, end):
    """Load events for a date range.

    user_id is null until after signup; filter on session_id for anonymous users.
    """
```

## Keep Output Small

Whatever a script prints lands in the context window. Return a summary, or
write the full result to a file and print its path.

## Gotchas

- Plugin variables are not in the environment of commands Claude runs. A
  script cannot read them; pass any path it needs as an argument.
- If the helpers need packages, say how to install them. Install into
  `${CLAUDE_PLUGIN_DATA}`, not the skill folder, which is replaced on update.
- Give every tool script a usage line or `--help`, so Claude does not have to
  read the source to call it.
