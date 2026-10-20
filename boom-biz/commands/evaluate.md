---
description: Evaluate a business idea, product plan, or feature through a panel of investor personas
argument-hint: <idea description or file path>
---

Evaluate the user's idea through a panel of seven investor personas, each running as a parallel subagent.

## Input

The user provided: `$ARGUMENTS`

This is either:
- **File path**: a path to a document containing the idea. Detect this by checking whether the argument is a single token that looks like a filesystem path (starts with `/`, `./`, `../`, or `~`, or ends with a file extension like `.md`, `.txt`, `.doc`). If detected, read the file. If the file does not exist, tell the user and stop.
- **Inline text**: anything else is treated as a direct description of the idea, product, plan, or feature.

If the input is empty or missing, ask the user to describe their idea.

## Pre-flight: Gather Missing Information

Before launching the panel, you need three pieces of context. Check whether the input already covers them. For any that are missing, ask the user in a single batch:

1. **Stage**: What stage is this? (raw concept / MVP in progress / live product adding a feature / pivot / something else)
2. **Constraints**: Any known constraints? (budget, timeline, team size, technical limitations, regulatory)
3. **Target market**: Who is this for? (if not already clear)

Do not launch agents until you have answers for all three, or the user explicitly says to proceed without them.

## Execution

Once you have the full brief, assemble the complete pitch text: the idea description plus the stage, constraints, and target market context.

Launch all seven investor agents **in parallel** using the Agent tool. The agents are:

1. `graham` -- Paul Graham
2. `thiel` -- Peter Thiel
3. `altman` -- Sam Altman
4. `andreessen` -- Marc Andreessen
5. `naval` -- Naval Ravikant
6. `balaji` -- Balaji Srinivasan
7. `collison` -- Patrick Collison

For each agent, use this prompt structure:

```
Evaluate the following idea from your investor perspective. Here is the complete brief:

<brief>
{the assembled pitch text}
</brief>

Respond with raw JSON exactly matching the schema in your output format section. No surrounding prose, no code fences, no preamble. Output JSON only.
```

## Synthesis

After all seven agents return, parse each response as JSON. If any agent returned malformed JSON, note the agent name, skip it for the structured tables, and include the raw response in an appendix at the end.

Before writing the synthesis, also read each agent's static lens scope by reading `${CLAUDE_PLUGIN_ROOT}/agents/*.md` and extracting the `## Angles you do not weigh` section from each. You will use these to surface the panel's collective blind spots.

### Individual Verdicts

| Investor | Verdict | Conviction | Bull Case (1-line) | Bear Case (1-line) |
|----------|---------|------------|--------------------|--------------------|
| Graham     | `verdict` | `conviction` | `bull_case` truncated to one sentence | `bear_case` truncated to one sentence |
| Thiel      | ... | ... | ... | ... |
| Altman     | ... | ... | ... | ... |
| Andreessen | ... | ... | ... | ... |
| Naval      | ... | ... | ... | ... |
| Balaji     | ... | ... | ... | ... |
| Collison   | ... | ... | ... | ... |

### Bias Flags Fired

For each agent whose `bias_flags` array is non-empty, list the agent and the biases fired. This signals where verdicts are partly bias-driven rather than analysis-driven. If no biases fired across the panel, write "No biases flagged."

### Lens Scope of the Panel

Two passes:

1. **Per-idea omissions** -- aggregate `angles_not_weighed_for_this_idea` across the seven agents. These are angles within an agent's lens that it intentionally did not apply to this pitch.
2. **Structural blind spots** -- aggregate the static `## Angles you do not weigh` sections from each agent file. List angles that 4+ agents do not weigh by lens. These are dimensions the panel collectively does not assess.

This section converts silent omissions into explicit signal. A unanimous "invest" where all seven do not weigh unit economics is a different verdict than a unanimous "invest" where multiple agents weighed them.

### Consensus

- **Where they agree**: themes that multiple investors raised on the same side. Mark whether the agreement is broad (different lenses converging) or narrow (same lens applied by multiple agents).
- **Where they diverge**: points of genuine disagreement and why their frameworks produce different answers. Cite the lens that drove each side.
- **Recurring risks**: concerns raised by 3+ investors via `bear_case` or `key_question`.

### Key Questions

Deduplicated list of the most important questions raised across all investors, drawn from `key_question` and `lens_specific_question`. Group by theme, not by investor.

### Synthesis

Your own integrated assessment. Not a vote count. Weigh the arguments by their quality and relevance, accounting for the bias flags fired and the lens scope of the panel. State what the strongest path forward looks like and what the founder should resolve first.
