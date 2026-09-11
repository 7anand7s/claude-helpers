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

Five subagent roles that pair with the account-level `orchestrate` skill (lead = Fable; hands and reviewers = cheaper tiers):

| Agent | Model | Role | Output contract |
|---|---|---|---|
| `scout` | Haiku | Locate files/symbols/call sites — never dumps contents | `FOUND / NOT FOUND` |
| `researcher` | Sonnet | Read docs/source, report verified facts, flag the rest | `VERIFIED / UNVERIFIED` |
| `builder` | Sonnet | The hand: implement a ticket in its own worktree/branch, run the suite there, commit | `RESULT / TOUCHED / SUITE / DEVIATIONS / CLAIMS` |
| `refuter` | Opus | Cold reviewer: sees only diff + spec, reruns suite, adversarial | `VERDICT ACCEPT\|REWORK / CHECKED / EVIDENCE / MUST_FIX / RISK / RECOMMEND` |
| `debugger` | Opus | Diagnosis-only root cause on repeated failure; never edits | `REPRODUCED / ROOT CAUSE / CONFIDENCE / FIX FOR HAND` |

The contracts match the `orchestrate` skill's ticket loop (mint → dispatch → cold review → judge → merge gate → fold/retire). If the skill's schemas change, change them here too.

## Layout

```
.claude-plugin/marketplace.json        # marketplace manifest (this repo)
plugins/orchestrate-agents/
  .claude-plugin/plugin.json           # plugin manifest
  agents/*.md                          # subagent definitions
```
