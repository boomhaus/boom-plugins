# Persistent Data Patterns

Read this file when a later run of the skill should know what an earlier run
did or produced.

## Where to Store It

| Skill ships in | History belongs to | Store in |
|---|---|---|
| A plugin | The user, across projects | `${CLAUDE_PLUGIN_DATA}/<skill-name>/` |
| A plugin | One project | `.claude/<skill-name>/` in the project |
| A project's `.claude/skills/` | That project | `.claude/<skill-name>/` in the project |

Do not write into the skill's own folder. A plugin's files are replaced on
update, and everything written there is lost.

`${CLAUDE_PLUGIN_DATA}` resolves to `~/.claude/plugins/data/<plugin-id>/`. It is
kept across plugin updates, shared by every skill in the plugin (hence the
`<skill-name>` subfolder), and deleted when the plugin is uninstalled.

## Pick the Lightest Form

| Form | Use for |
|---|---|
| Append-only log, one line per run | What happened and when. The default. |
| JSON file | A small current state that is replaced whole: a cursor, a last-seen ID |
| SQLite | History the skill needs to query or aggregate |

Prefer appending to rewriting. A log that only grows cannot be corrupted by one
bad run and is easy for the user to inspect.

## Read, Act, Append

Load a bounded slice at the top of the skill, then write after the work is done:

```markdown
## History

!`tail -n 20 ${CLAUDE_PLUGIN_DATA}/standup/runs.log 2>/dev/null || echo NO_HISTORY`

Use the entries above to report only what changed since the last run.
If the line reads NO_HISTORY, this is a first run: proceed without comparison.

After posting, create `${CLAUDE_PLUGIN_DATA}/standup/` if it is missing and
append one line to `runs.log`: date, channel, one-sentence summary.
```

Read the last N entries, not the whole file. The log grows without limit and
the context window does not.

## Gotchas

- `${CLAUDE_PLUGIN_DATA}` is replaced with a path when the skill text loads,
  including inside `!` command lines. It is not an environment variable in
  commands Claude runs, so write the variable in the skill body, and pass the
  path to bundled scripts as an argument.
- It is substituted only in plugin skills. In a project or user skill it stays
  as literal text.
- The plugin's data directory is created for you; the `<skill-name>` subfolder
  is not. Create it before the first write.
- A failing `!` command aborts the whole skill. Every read needs a fallback
  (`2>/dev/null || echo NO_HISTORY`), or the first run never starts.
- Store nothing sensitive: it is a plain file on disk. No tokens, no private content.
- History is one user's record. A lesson that should change the skill for
  everyone goes in the skill's Gotchas section, not the log.
