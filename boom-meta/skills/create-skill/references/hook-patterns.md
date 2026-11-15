# Skill-Scoped Hook Patterns

Read this file when the skill has a rule that must hold no matter what: a
command that must never run, or files that must not be touched.

A hook declared in a skill's frontmatter is registered when the skill is
invoked. Until then it does nothing. Use it for guardrails that would be too
strict to leave on all the time.

## A Hook Is a Backstop

State the rule and its reason in the skill body first, so Claude does not
attempt the action. Add a hook to catch the case where it does anyway. The
hook is the second line of defence, never the only one.

| The rule | Use |
|---|---|
| Can be decided from the tool call alone: a command pattern, a file path | Prose rule plus a hook |
| Needs judgment or context | Prose rule with its reason; a hook cannot decide it |

A hook stops accidents. It is not a security boundary: a pattern cannot
anticipate every route to the same result, such as a script file that contains
the forbidden command. If an action must be impossible, tell the user it
belongs outside the skill: a permission deny rule in their settings, a sandbox,
or a session without the credentials.

## Frontmatter

```yaml
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "\"${CLAUDE_PLUGIN_ROOT}/skills/<skill-name>/scripts/guard.sh\"; s=$?; if [ $s -ne 0 ] && [ $s -ne 2 ]; then echo Guard failed to run, blocking. >&2; exit 2; fi; exit $s"
```

Keep the wrapper after the script path. Without it, a guard that cannot run
(missing, not executable, crashed) is treated as a non-blocking error and the
command goes through. With it, a broken guard blocks every matching call.

Use `matcher: "Edit|Write"` to guard file changes.

## The Guard Script

The script receives the tool call as JSON on stdin. Exit 2 to block; the text
written to stderr is shown to Claude. Exit 0 to allow.

```bash
#!/bin/bash
cmd=$(jq -r '.tool_input.command')
if echo "$cmd" | grep -qE 'GUARD_SELFTEST|rm -rf|push .*--force|DROP TABLE'; then
  echo "Blocked by <skill-name>: destructive command. Ask the user to run it." >&2
  exit 2
fi
exit 0
```

For `Edit` and `Write`, the path is in `.tool_input.file_path`.

Say in the blocking message what Claude should do instead. A bare "blocked"
leaves it to guess, and it may try a variant of the same command.

## Test Without Risk

Never test a guard by attempting the real action. Two checks cover it:

1. **The script alone.** Pipe a sample call in and read the exit code. Nothing
   is executed.

   ```bash
   echo '{"tool_input":{"command":"rm -rf /"}}' | scripts/guard.sh; echo $?
   ```

2. **The registration.** Include a harmless canary word in the pattern, as
   `GUARD_SELFTEST` above. Invoke the skill and have Claude run
   `echo GUARD_SELFTEST`. Blocked means the hook is live. If it prints, the
   hook is not registered, and all that happened is a word on the screen.

## Tell the User

State in the skill body that invoking it switches the guard on, and what it
blocks. A user who does not know will read a blocked command as a failure.

## Gotchas

- Locate the script with `${CLAUDE_PLUGIN_ROOT}`. `${CLAUDE_SKILL_DIR}` resolves
  to nothing in a hook command, so the script is not found.
- Make the script executable (`chmod +x`).
- The example needs `jq`. If it may be missing, match on the raw input instead.
- Plugin agents ignore a `hooks` field. This pattern is for skills.
