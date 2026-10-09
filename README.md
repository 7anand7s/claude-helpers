# claude-helpers

Personal Claude Code plugin marketplace. Holds the assets that don't sync through the account (subagents today; skills can be added later) so they can be installed identically on every machine.

## Install (once per machine)

Inside Claude Code:

```
/plugin marketplace add 7anand7s/claude-helpers
/plugin install orchestrate-agents@claude-helpers
```

Then restart the session (or `claude --resume`) — agents are loaded at process start. Check with `/agents`.

## Update

After pushing changes to this repo:

```
/plugin marketplace update claude-helpers
/plugin update orchestrate-agents@claude-helpers
```

## Plugins

### orchestrate-agents

Seven subagent roles plus the `orchestrate` skill that drives them. The lead (orchestrator) is whatever model your session runs — pick it with `/model` (Opus recommended). The plugin sets only the subagents' default models; the lead can override per dispatch.

| Agent | Model | Role | Output contract |
|---|---|---|---|
| `scout` | Haiku (effort high) | Locate files/symbols/call sites — never dumps contents | `FOUND / NOT FOUND` |
| `chore` | Haiku (effort high) | Cheap hand: mechanical edits to named files, proven by a `CHECK` command | `STATUS / TOUCHED / CHECK / ROUNDS / LEFTOVER / BLOCKED` |
| `triage` | Haiku (effort high) | Runs a test/build command or reads a log, returns grouped failures — keeps raw output away from the lead | `COMMAND / TOTALS / FAILURES / GROUPS / LOG / UNCONFIRMED` |
| `researcher` | Sonnet | Read docs/source, report verified facts, flag the rest | `VERIFIED / UNVERIFIED` |
| `builder` | Sonnet | The hand: implement a ticket in its own worktree/branch, run the suite there, commit | `RESULT / TOUCHED / SUITE / DEVIATIONS / CLAIMS` |
| `refuter` | Opus (lead passes Sonnet for low-risk tickets) | Cold reviewer: sees only diff + spec, reruns suite, adversarial | `VERDICT ACCEPT\|REWORK / CHECKED / EVIDENCE / MUST_FIX / RISK / RECOMMEND` |
| `debugger` | Opus | Diagnosis-only root cause on repeated failure; never edits | `REPRODUCED / ROOT CAUSE / CONFIDENCE / FIX FOR HAND` |

### Escalation ladder

A verifier (cold reviewer, `CHECK` command, or `triage` at the merge gate) decides whether an attempt failed — never the hand itself. Every failed attempt goes to a fresh hand one rung up, carrying a short failure summary rather than the previous transcript:

```
chore (haiku) → builder @ sonnet → builder @ opus → builder @ fable → you
```

Tickets start on the rung their difficulty score routes them to (only fully specified, command-checkable chores start on Haiku). Review rigor rises as a ticket climbs. Every rung is logged in the mission brief's `DECISIONS` so you can see where Haiku holds up.

### Efficiency: read once, point many times

- **Code map** — one shared file per mission (`.claude/scratch/codemap.md`). Scout, researcher, builder and debugger append one-line `file:line` facts; the lead points each hand at the relevant sections instead of letting it search. The lead never reads its contents, so its own context stays lean.
- **Pointers, not searches** — packets carry exact `TOUCHES` + code map sections; builders and reviewers start there and read further only where needed.
- **Warm hands** — consecutive tickets in the same area continue the same builder (it already holds that code) instead of spawning a fresh one; capped at 3 tickets before a fresh start. Reviewers are always cold.
- **Reviewer by risk** — Sonnet for low-rigor reviews, Opus for medium/high.

The contracts match the `orchestrate` skill's ticket loop (mint → dispatch → cold review → judge → merge gate → fold/retire). If the skill's schemas change, change them here too.

## Layout

```
.claude-plugin/marketplace.json        # marketplace manifest (this repo)
plugins/orchestrate-agents/
  .claude-plugin/plugin.json           # plugin manifest
  agents/*.md                          # subagent definitions
  skills/orchestrate/SKILL.md          # the orchestrate skill (lead playbook)
```

## Requirements and caveats

- Claude Code ≥ v2.1.293 (the `haiku` alias → Haiku 5.5; `maxTurns` and `omitClaudeMd` frontmatter).
- On Bedrock / Vertex / Foundry, `haiku` still means Haiku 4.5: set `ANTHROPIC_DEFAULT_HAIKU_MODEL` to your provider's Haiku 5.5 ID.
- `CLAUDE_CODE_SUBAGENT_MODEL_FORCE=1` or `CLAUDE_CODE_EFFORT_LEVEL` override every agent file's model/effort — leave them unset.
- If `fable` isn't available on your plan or provider, the ladder tops out at `opus`.
- Consider denying `git push` for subagents via `permissions.deny` in your settings.

## Model versions

Agents use family aliases (`haiku`, `sonnet`, `opus`), never versioned names. Claude Code maps each alias to the newest model in that family, so when a new Opus/Sonnet/Haiku ships, updating Claude Code is enough — no change here. Don't pin a version unless you deliberately want to stay on an older model (then use the full model ID, e.g. `claude-opus-5-5`).
