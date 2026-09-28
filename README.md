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

Five subagent roles plus the `orchestrate` skill that drives them. The lead (orchestrator) is whatever model your session runs — pick it with `/model` (Opus recommended). The plugin sets only the subagents' default models; the lead can override per dispatch.

| Agent | Model | Role | Output contract |
|---|---|---|---|
| `scout` | Haiku | Locate files/symbols/call sites — never dumps contents | `FOUND / NOT FOUND` |
| `researcher` | Sonnet | Read docs/source, report verified facts, flag the rest | `VERIFIED / UNVERIFIED` |
| `builder` | Sonnet | The hand: implement a ticket in its own worktree/branch, run the suite there, commit | `RESULT / TOUCHED / SUITE / DEVIATIONS / CLAIMS` |
| `refuter` | Opus (lead passes Sonnet for low-risk tickets) | Cold reviewer: sees only diff + spec, reruns suite, adversarial | `VERDICT ACCEPT\|REWORK / CHECKED / EVIDENCE / MUST_FIX / RISK / RECOMMEND` |
| `debugger` | Opus | Diagnosis-only root cause on repeated failure; never edits | `REPRODUCED / ROOT CAUSE / CONFIDENCE / FIX FOR HAND` |

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

## Model versions

Agents use family aliases (`haiku`, `sonnet`, `opus`), never versioned names. Claude Code maps each alias to the newest model in that family, so when a new Opus/Sonnet/Haiku ships, updating Claude Code is enough — no change here. Don't pin a version unless you deliberately want to stay on an older model (then use the full model ID, e.g. `claude-opus-5-5`).
