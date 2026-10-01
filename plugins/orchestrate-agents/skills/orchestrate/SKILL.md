---
name: orchestrate
description: "Invoke at the start of ANY non-trivial, multi-step task — features, refactors, migrations, multi-file changes, investigations, build-outs — whenever you are the driving model and could accomplish the work through subagents rather than doing it yourself. This is the default operating mode for orchestration-capable sessions: you hold the big picture and delegate execution, you do NOT write the bulk of the work yourself. Trigger even when the user never says \"orchestrate\", \"delegate\", or \"subagents\" — if the task has more than one step or touches more than one file, route it through this skill. Only skip for genuinely atomic requests (a one-line factual answer, or a trivial single-file tweak the user explicitly wants done inline). Requires real subagent/Task tooling (Claude Code or Cowork); without it, say so rather than simulating subagents in one context."
---

# Orchestrate — drive through subagents, stay lean, review everything

You are the lead, not the hand. Your job is to keep the whole mission in view, delegate execution to subagents, review what comes back rigorously, and never let bad work through. You do **not** write the bulk of the code yourself.

Three guarantees this skill exists to hold:
1. **Big picture** — you never lose the thread, because you always hold a current mission brief.
2. **No bloat** — your context stays lean, because raw work never enters it.
3. **Rigorous review** — nothing crappy gets accepted, because review is a first-class, mandatory step with its own dial.

## The invariant that makes all three true

**Your context = mission brief + task ledger + the task packets in flight. Nothing else.**

Raw file contents, terminal output, and subagent scratch work **never** enter your context. You read *decisions and verdicts*, never raw work. Every rule below serves this invariant — if something would pull raw work into your context, dispatch a subagent to absorb it instead.

**Read once, point many times.** What subagents learn about the code goes into the shared code map (below), not your context and not their dead context. You hold only its path and section names, and you point each hand at exactly the sections and `file:line` locations it needs, so no agent re-discovers what another already found.

## The loop

```
1. Plan                → plan-first, ALWAYS: a short plan of tasks + acceptance, no interview
   Huge build only     → after the user explicitly confirms: phased clarification → to-prd → to-issues
2. Write the mission brief                    → the durable big picture
3. For each task in the current slice:
   a. Scout (if scope isn't known)   → cheap locate-only pass populates TOUCHES + the code map
   b. Route                          → model size / thinking tier / review rigor
   c. Mint                           → branch + worktree + seat for this ticket
   d. Dispatch                       → fresh hand, or the warm hand from the previous ticket in the same area
   e. Cold review                    → fresh reviewer gets diff + packet only → ACCEPT | REWORK
   f. Judge                          → REWORK to same hand once → then escalate; ACCEPT → merge gate
   g. Merge gate                     → merge to main, full suite on main; red = revert + escalate
   h. Fold + retire                  → fold outcome into brief; remove worktree, delete branch; keep hand warm only if the next ticket is in the same area
4. On repeated failure, escalate; at the cap, surface to the user.
```

**Planning is never skipped.** Every non-trivial request gets a `plan-first` pass — tasks, order, acceptance, disjoint scopes — before anything is dispatched. What is reserved for **huge builds only, and only after the user explicitly says yes**, is the heavy front-end: multi-phase clarifying questions, a written brief, `to-prd`, then `to-issues`. If a request looks that big, ask once — *"this looks like a big build; want the full PRD → issues treatment, or plan-and-go?"* — and default to plan-and-go if unanswered.

Between dispatches, folding the brief **is your real job**. A stale brief means reviewers grade against an old picture.

---

## The three artifacts

### Mission brief (durable — this is your entire world)
```
GOAL:        one paragraph — what "done" means for the whole mission
SLICE:       the vertical slice currently in flight
CONSTRAINTS: architectural invariants, conventions, non-negotiables
SEATS:       N — max hands in flight at once (default 4; lower on a small box)
CODEMAP:     .claude/scratch/codemap.md — shared code map for this mission (path only; never its contents)
DONE:        [ticket → verdict]  ledger of accepted work, folded and terse
OPEN:        [ticket → state]    remaining work with its ticket state
DECISIONS:   routing/override log — why each non-obvious call was made
```

### Task ledger — ticket states
Lives in `DONE`/`OPEN` above, one line per ticket. Each ticket moves through a fixed state machine; log every transition so the run is auditable:

```
OPEN → MINTED → IN_PROGRESS → REPORTED → REVIEW → (REWORK → IN_PROGRESS)* → ACCEPTED → MERGED | REVERTED → CLOSED
```

### Task packet (per ticket — ephemeral, dies after the fold)
```
TICKET:     id + one-line title
TASK:       what to build, precisely (the spec)
ACCEPTANCE: exactly how the reviewer will verify it (tests / schema / DoD subset)
TOUCHES:    expected files / surface area — must be disjoint from every other in-flight ticket
CONTEXT:    minimal brief-derived context the hand needs — NOT raw file dumps
CODEMAP:    path + the section headings relevant to this ticket, plus scout's file:line hits
ROUTING:    the difficulty read and the resulting size / thinking / rigor
BRANCH:     ticket/<id>-<slug>     WORKTREE: path     SEAT: k of N
```

---

## Scout — populate TOUCHES without doing it yourself

If you don't already know precisely which files/symbols a task touches, don't grep it out yourself and don't hand a work subagent an unscoped task. Dispatch a cheap, locate-only scout first: smallest capable model, find-only tools, strict output contract — paths, line numbers, symbol names, one-line context each. No file dumps, no explanations, no fixing anything.

```
FOUND:     [path:line — symbol/pattern — 1-line context]
NOT FOUND: [anything requested but not located]
```

The scout's report becomes the task packet's `TOUCHES`. Skip this step only when the packet's scope is already fully known. Scouts are read-only and always safe to run in parallel.

---

## Code map — the shared memory that stops re-reading

One file per mission, `.claude/scratch/codemap.md` in the main checkout, outside any ticket worktree — pass its absolute path. At mission start, add `.claude/scratch/` to `.git/info/exclude` so it is never committed. Scouts and researchers **append** to it; builders and the debugger read it before opening code and append anything durable they learned. Sections are keyed by area, so you can point a hand at just the ones it needs:

```
## <area> (e.g. auth/session)
- path:line — symbol — what it does / how it's called (1 line)
- invariant / convention / gotcha discovered (1 line, with file:line)
```

Rules:
- Terse facts with `file:line`, never code bodies. It is an index into the code, not a copy of it.
- Parallel hands append only under their own area's heading; if an edit fails because the file changed, re-read and retry.
- Before dispatching a scout, check whether the map already has the area (`grep '^## ' <codemap>` — headings only, never the contents). If it does, skip the scout and point straight at the section.
- When a merge changes code the map describes, the hand that made the change updates those entries in the same run. A stale map is worse than none.
- You never read the map's contents yourself — only its headings. It exists so the hands share memory without routing it through you.

---

## Routing — three independent knobs, not one score

The common mistake is collapsing everything into a single difficulty number. Keep three separate decisions:

- **Model size** ← capability the task demands + blast radius
- **Thinking tier** ← reasoning depth + ambiguity (apply expensive reasoning only where it pays)
- **Review rigor** ← blast radius + reversibility  *(independent — a task can be trivial to do but catastrophic to get wrong)*

Rate each signal 0–2:

| Signal | 0 (easy) | 1 | 2 (hard) |
|---|---|---|---|
| Reasoning depth | mechanical / lookup | some multi-step logic | novel or deep chains |
| Ambiguity | fully specified | minor gaps | underspecified |
| Blast radius | throwaway / isolated | one module | cross-cutting / irreversible |
| Context breadth | single file | a few files | sprawling |
| Output determinism | verifiable (tests/schema) | semi | subjective |
| Escalation history | first attempt | — | already failed once |

**Deterministic score, logged override.** Compute the sum every time and route by the table below — so every decision is auditable and the rubric can be tuned against real outcomes. You *may* override the tier, but only if you write the reason into `DECISIONS`.

| Score (0–12) | Model | Thinking |
|---|---|---|
| 0–2 | `haiku` | low |
| 3–5 | `sonnet` | medium |
| 6–8 | `sonnet` | high |
| 9–12 | `opus` | high / ultra |

Route by family alias (`haiku` / `sonnet` / `opus`), never a versioned name — the alias resolves to the newest model in that family, so the table stays correct as new models ship. The agent file's `model:` is only the default; pass the routed model on each dispatch call.

Blast radius **also** independently sets review rigor (below), regardless of the size/thinking result.

---

## Mint — branch, worktree, seat

Every ticket that edits code gets its own isolation before a hand is dispatched:

- **Branch** `ticket/<id>-<slug>` off main.
- **Worktree** for that branch (the runtime's worktree isolation option if the Task tool offers one, otherwise `git worktree add <path> <branch>`). The hand works *only* there. Isolation is the default, not the exception — it is what makes parallel hands safe and makes ACCEPT/REVERT a clean git operation.
- **Seat** — one of `SEATS` concurrency slots. If all seats are taken, the ticket waits in `OPEN`; don't exceed the cap.

If the task needs running services (containers, ports, volumes), a worktree alone does **not** isolate them — namespace the run by branch (e.g. compose project name = branch slug, distinct ports) so parallel hands don't collide.

Read-only tickets (research, review, scouting) don't need a branch or worktree — just a seat.

---

## Dispatch — the hand

Give it: **the mission brief + the task packet.** Nothing more. It boots in its worktree, reads the code map sections and `file:line` hits it was pointed at, opens only what it still needs, implements, **runs the suite in its own tree**, commits to its branch, and reports. Deviations from the spec must be flagged, never silently absorbed.

**Point, don't make it search.** A packet whose `TOUCHES` and `CODEMAP` are specific lets the hand go straight to the right lines. If you can't make them specific, run a scout first — don't hand a Sonnet builder a repo-wide search.

**Warm hands for related work.** If the next ticket touches the same area as a hand that just finished (overlapping files or code map sections), **continue that hand** (SendMessage to its agent id) with the new packet and new worktree instead of spawning a fresh one — it already holds the code in context. Start fresh instead when the area changes, the hand has done 3 tickets in a row, or its reports are getting sloppy (a long context degrades quality). Warm reuse never applies to reviewers: every review is cold.

Require a **bounded** report, not a raw dump:
```
RESULT:      what was built, in 3–5 lines
TOUCHED:     files changed, one line each
SUITE:       command → pass/fail summary (counts, not logs)
DEVIATIONS:  anything that differs from the spec, and why
CLAIMS:      verification-relevant assertions the reviewer should check
```
If a hand tries to hand back raw file bodies or full terminal logs, that is a re-dispatch — the report contract was not met.

**Parallelism is the default.** Compute the set of `OPEN` tickets whose `TOUCHES` are pairwise disjoint, mint and dispatch as many as there are free seats, together. Only serialize tickets that genuinely overlap. Because each hand is in its own worktree, overlap is a merge-conflict risk rather than a corruption risk — but still keep scopes disjoint; conflicts cost a rebase round.

---

## Cold review — the load-bearing step

Never skip it. Not even for Haiku-tier work. This is the quality estimator, and the quality estimator is what determines whether this whole architecture produces good output or garbage.

The reviewer is **cold**: a fresh context that receives **only the diff and the task packet** (spec + acceptance) — not the hand's report, not its narrative, not your conversation. It may read anything in the worktree it wants (master read access), and it reruns the suite itself rather than trusting `SUITE:`. Rigor (set by blast radius) decides how hard it looks:

- **Low rigor** — `sonnet` reviewer. Check the diff against `ACCEPTANCE`; rerun the suite.
- **Medium rigor** — `opus` reviewer. Run the full `definition-of-done` checklist; evidence required at `file:line` or `command → result`, never "should work".
- **High rigor** — medium, **plus a second independent cold reviewer** (`opus`) blind to the first.

Cold means no narrative, not no pointers: give the reviewer the diff, the packet, and the code map path. It starts from the diff and the files in `TOUCHES` and reads further only where something looks wrong. If the review is a genuine *design* fork (multiple defensible answers, costly to reverse) rather than a correctness check, suggest `council` to the user rather than auto-running it — council costs 6–7× and wants consent.

The reviewer returns **only** this schema, then is retired:
```
VERDICT:        ACCEPT | REWORK
CHECKED:        which criteria / DoD items were actually run
EVIDENCE:       [file:line | command → result]   concrete, not prose
MUST_FIX:       [...]   empty iff ACCEPT — specific, actionable
RISK:           none | low | med | high
RECOMMEND:      accept | rework | escalate | surface-to-user
```
You read the verdict, not the reviewer's scratch work. That stays in the reviewer's dead context.

---

## Judge — REWORK and escalation

**REWORK, first round → same hand.** The hand still holds the context; continue *it* (SendMessage to its agent id) with `MUST_FIX` as the body. It fixes in its worktree, reruns the suite, commits, reports again. A **new** cold reviewer reviews the new diff. This is not a blind retry — it carries the reviewer's specific findings — so it does not violate the never-retry-blind rule.

**REWORK, second round → escalate.** Classify why it failed twice:
- **Simple miss** (mechanical gap the hand keeps missing) → bump the model **one tier**, fresh hand in the same worktree, feedback packet = both rounds' `MUST_FIX`.
- **Unclear root cause** (no obvious mechanical fix, or failing the same way) → dispatch a **diagnosis-only** subagent to reproduce and state the root cause — it does not fix anything:
  ```
  REPRODUCED:  yes/no — how
  ROOT CAUSE:  specific mechanism, with file:line
  CONFIDENCE:  high/medium/low
  ```
  Its report becomes the feedback packet for the next hand.

**Cap at 2 escalations.** Then stop and surface to the user: the failing verdict **plus your own recommendation** (change approach, relax a constraint, take it manual). Don't loop Opus calls against a wall.

---

## Merge gate

On ACCEPT:
1. Merge the ticket branch into main (rebase first if main moved).
2. Run the **full suite on main** — not just the ticket's tests. Integration breakage is what per-ticket review cannot see.
3. **Green** → ticket `MERGED`. **Red** → revert the merge, ticket `REVERTED`, and treat it as a REWORK with the failing suite summary (counts + failing test names, not logs) as `MUST_FIX`. Escalation rules apply.

Don't merge while another ticket's suite is still running on main; queue merges.

---

## Experiments — competing hands

When comparing N approaches to the same problem: mint N tickets with the same spec and a one-line approach hint each, N branches, N worktrees, N seats. Each gets its own cold review. You pick the winner on the verdicts (and `EVIDENCE`), merge only that branch, and retire the rest — worktrees removed, branches deleted. Experimental branches never share a working tree.

---

## Fold and retire

After `MERGED`: fold the delta into the brief using `context-compressor`'s scratchpad pattern — update `DONE` with the terse outcome, update `CONSTRAINTS`/`DECISIONS` if the ticket established anything durable, discard the packet. Then **retire**: worktree removed, branch deleted, seat freed. The hand is released too, unless the next ticket qualifies for warm reuse (see Dispatch). Ticket → `CLOSED`. At mission end, delete the code map.

Because raw work **never entered your context** in the first place, this fold is the *only* compaction you ever need. That is the entire payoff of routing through subagents: the isolation is structural, not something you have to clean up after.

Release steps (changelog, version bump, tag, image) are the lead's job **only when the brief or the user calls for a release** — not after every ticket.

---

## Never-do list (these keep the guarantees true)

- **Never** read raw files or terminal output into your own context to "just quickly check" — dispatch a reviewer (or scout, for locating). The moment you read raw work, guarantee #2 is gone.
- **Never** skip `plan-first`. Never run the interview → brief → `to-prd` → `to-issues` front-end without the user explicitly confirming a huge build.
- **Never** accept a verdict or report that isn't in-schema. Enforce the bound.
- **Never** skip cold review, at any tier. Never let the reviewer see the hand's narrative — diff + spec only.
- **Never** retry blind. Rework carries `MUST_FIX`; the second failure escalates.
- **Never** merge without the full suite on main; never leave a red main — revert.
- **Never** dispatch against a stale brief — fold first.
- **Never** dispatch two hands with overlapping `TOUCHES`, and never let competing/experimental hands share a worktree.
- **Never** serialize tickets that don't need to be serial — fill free seats with disjoint tickets.
- **Never** exceed `SEATS`.
- **Never** leave a closed ticket's worktree or branch behind.
- **Never** paste code map contents into a packet or read them yourself — pass the path and section names.
- **Never** dispatch a hand with a vague `TOUCHES` into a large repo — scout first, or point it at the code map.
- **Never** dispatch a subagent for a genuinely atomic fix (one line, one lookup) — just do it inline.
- **Never** simulate subagents in a single context if real Task tooling is absent — say so instead.