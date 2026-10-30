# How mattpocock/skills expects wayfinder and implementation loops to be driven

Research for [jonaswre/t3code#3](https://github.com/jonaswre/t3code/issues/3), part of the map [Loops: completion-triggered skill loops for wayfinder and implementation](https://github.com/jonaswre/t3code/issues/1).

Sources pinned at `mattpocock/skills@a496f55` (2026-10-29) and `mattpocock/sandcastle@4518681` (2026-10-30).

## Answer

1. **The skills repo ships no loop runner, done signal, or loop convention.** None of the skills says "run me again", writes a stop marker, or tells you how to detect an empty frontier. Upstream refuses to add one. A queue-polling orchestrator "would be a new skill or tool", and new skills are out of scope ([#329](https://github.com/mattpocock/skills/issues/329), [#1175](https://github.com/mattpocock/skills/issues/1175), [`.out-of-scope/new-skills.md`](https://github.com/mattpocock/skills/blob/main/.out-of-scope/new-skills.md)). Loop limits belong to the harness, not to skills ([`.out-of-scope/subagent-recursion.md`](https://github.com/mattpocock/skills/blob/main/.out-of-scope/subagent-recursion.md)).
2. **The fresh-context rhythm the skills document is manual.** You run one skill invocation per session and clear between them. Wayfinder: "never resolve more than one ticket per session" (research tickets excepted). Implement: `/implement` per ticket, "`/clear`ing context between each one". The human is the loop.
3. **Upstream's AFK loop lives in a separate repo, [Sandcastle](https://github.com/mattpocock/sandcastle)**, and it is the Ralph pattern from aihero.dev. The contract: each iteration is a fresh agent session, the agent picks one task per iteration, and the loop stops early when the agent's output contains the literal string **`<promise>COMPLETE</promise>`**, or when it hits a hard iteration cap. The engine never injects the signal; the prompt documents it. Sandcastle's prompts are self-contained and do not call the skills.
4. **The skills and Sandcastle meet only through the tracker.** `to-tickets` labels tickets `ready-for-agent`, and Sandcastle's own dogfood planner polls `--label ready-for-agent`. No skill knows about Sandcastle's signal.
5. **Local copies are byte-identical to upstream HEAD.** That covers `wayfinder`, `implement`, `implement-spec`, `to-tickets`, `to-spec`, `handoff`, `research`, and `setup-matt-pocock-skills`, including the tracker docs. No local difference matters.

## Evidence

### Wayfinder: one ticket per session, the skill picks, no "map done" signal

- [`skills/engineering/wayfinder/SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/wayfinder/SKILL.md), Invocation: "Two modes. Either way, **never resolve more than one ticket per session**, with the exception of research tickets." In work-through mode a ticket is optional: "without one, you pick the next decision, not the user … take the first frontier ticket in order. **Claim it**: assign it to yourself before any work."
- The frontier is "the open, unblocked, **unclaimed** children". The claim is the assignee. On GitHub the frontier query drops issues "with an open blocker … or an assignee; first in map order wins", and the claim is `gh issue edit <n> --add-assignee @me`, "the session's first write" ([`issue-tracker-github.md`, Wayfinding operations](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/issue-tracker-github.md)).
- A resolution is a comment, then close, then a context pointer added to Decisions so far (SKILL.md step 4). The skill gives no step for an empty frontier and no "map cleared" output. The docs only say that when the map is finished, "the session that finishes the map points you toward a spec, not a pull request" and "a session resolves one ticket … Then it stops" ([`docs/engineering/wayfinder.md`, It's working if](https://github.com/mattpocock/skills/blob/main/docs/engineering/wayfinder.md)).
- Most ticket types are HITL. Grilling and prototype are HITL; task can be either; research is AFK. "A HITL ticket only resolves through that live exchange; the agent never stands in for the human's side of it." The docs recommend one ticket at a time over parallel: "one session can ask you a question you just answered in the other".
- Charting mode ends with "Stop: charting is one session's work". It fires `/research` subagents in parallel on `research/<name>` branches and opens no PR.
- Known open gaps that affect a loop:
  - Work-through mode never fires research subagents for research tickets it graduates ([#941](https://github.com/mattpocock/skills/issues/941), accepted with an agent brief).
  - Nothing says who does the git work for research findings, and an AFK resolution has no review gate before it closes ([#1241](https://github.com/mattpocock/skills/issues/1241), open).
  - Wayfinder can bypass the ticket-type workflows after charting ([#540](https://github.com/mattpocock/skills/issues/540), open).

### Implementation: `/implement` is one ticket per fresh session; `/implement-spec` is one orchestrated run

- [`ask-matt/SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/ask-matt/SKILL.md) step 3 names two ways to work the tickets. One is "**`/implement`** per ticket, **`/clear`ing context between each one** … any ticket whose blockers are done can be grabbed". The other is "**`/implement-spec`** for the whole spec in one run".
- [`implement/SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/implement/SKILL.md) takes a ticket reference from the user. It does **not** pick from the frontier and does **not** close the ticket. [`docs/engineering/implement.md`](https://github.com/mattpocock/skills/blob/main/docs/engineering/implement.md): "`implement` has no completion step. It ends at the commit and never touches the work item … If nothing gets closed, nothing ever becomes visibly unblocked." It also says "one invocation, one ticket".
- [`implement-spec/SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/implement-spec/SKILL.md) is a single session that orchestrates the whole task graph:
  - It runs a fresh implementer subagent per ticket, each in its own worktree, in the background "for maximum concurrency".
  - A merger subagent lands each one on the integration branch.
  - It runs one `code-review` at the end.
  - It closes the tickets at the end (step 8).

  It says nothing about being re-run or resumed. Its docs require "a harness that runs subagents in the background and gives each one a git worktree … on a harness that runs subagents one at a time, it is only a slower `implement`" ([`docs/engineering/implement-spec.md`](https://github.com/mattpocock/skills/blob/main/docs/engineering/implement-spec.md)). The latest commit (`a496f55`, [#1342](https://github.com/mattpocock/skills/pull/1342)) made it use "a fresh implementer subagent for each ticket and for the code-review fixes".
- On a GitHub tracker the frontier goes stale mid-run. "The tracker's blocked-by count only drops when a blocker *closes*, and tickets typically close when the PR merges". The fix the docs give is to track merged tickets in the orchestrator itself (implement-spec docs, "Blocked tickets never start").
- `implement-spec` docs, "Does this replace Sandcastle or an AFK script?": "No … `implement-spec` puts an agent in charge of orchestration inside one harness session, which needs no infrastructure and lets you watch and steer. For work that is truly AFK, a deterministic loop ([Sandcastle], a shell script, a CI job) is faster, cheaper, and more reliable, because no agent makes the orchestration decisions."
- When users asked for a fresh-session-per-ticket loop, upstream closed the issue and pointed to `/implement` plus `/clear`, or to `/implement-spec` ([#619](https://github.com/mattpocock/skills/issues/619), [#618](https://github.com/mattpocock/skills/issues/618)).

### Invocation: these skills are user-invoked only

- `wayfinder`, `implement`, `implement-spec`, `to-tickets`, and `to-spec` set `disable-model-invocation: true`, and Codex gets `policy.allow_implicit_invocation: false` in `agents/openai.yaml`. They are "reachable **only by the human typing its name** … no other skill can call it" ([`.agents/invocation.md`](https://github.com/mattpocock/skills/blob/main/.agents/invocation.md)). A loop must therefore start each iteration as a user message that carries the provider's skill-invocation syntax (for example `/wayfinder <map>` in Claude Code). An agent cannot reach these skills through a tool call.

### Sandcastle and Ralph: the actual AFK loop contract

- [Sandcastle README](https://github.com/mattpocock/sandcastle/blob/main/README.md):
  - `run({ maxIterations, completionSignal, promptFile })`. `maxIterations` defaults to 1. `completionSignal` defaults to `"<promise>COMPLETE</promise>"` and can be a string or an array of strings.
  - "When the agent outputs `<promise>COMPLETE</promise>`, the orchestrator stops the iteration loop early. This is a convention you document in your prompt for the agent to follow — the engine never injects it."
  - The result reports `iterations[]` (each with an optional `sessionId` and `usage`) and the matched `completionSignal`.
  - Once the signal is seen, a short completion timeout covers a process that hangs after it.
- Iterations are fresh sessions. "Only iteration 1 receives the resume flag; subsequent iterations (if any) start fresh", and "each iteration produces its own agent session" ([ADR 0011](https://github.com/mattpocock/sandcastle/blob/main/docs/adr/0011-resume-is-one-iteration.md)). Shell expressions in the prompt (``!`cmd` ``) are "evaluated … at the start of each iteration, so the agent always sees fresh data" ([`simple-loop/main.mts`](https://github.com/mattpocock/sandcastle/blob/main/src/templates/simple-loop/main.mts)).
- [`simple-loop/prompt.md`](https://github.com/mattpocock/sandcastle/blob/main/src/templates/simple-loop/prompt.md) is the frontier-picking AFK loop:
  - The prompt injects the filtered open-issue list each iteration.
  - The agent picks "the highest-priority open issue that is not blocked by another open issue". Rule: "Work on **one issue per iteration**."
  - It commits with a `RALPH:` prefix and closes the issue itself.
  - "If you are blocked … leave a comment on the issue and move on — do not close it."
  - It emits `<promise>COMPLETE</promise>` "when all actionable issues are complete (or you are blocked on all remaining ones), or the open-issues block … is empty".
- The [`parallel-planner` template](https://github.com/mattpocock/sandcastle/blob/main/src/templates/parallel-planner/main.mts) has three phases:
  - Plan: one iteration emits `<plan>{"issues":[…]}</plan>`; an empty list means "exit cleanly".
  - Implement: N agents run in parallel, each up to 100 iterations, and each emits `COMPLETE` when its ticket is done.
  - Merge: a merge agent merges the branches and closes the issues.
  
  An outer loop repeats up to `MAX_ITERATIONS`. The signal's meaning depends on the loop: "backlog empty" in simple-loop, "this ticket done" in the per-ticket inner loop.
- Tracker presets put the filter in the list command, not in the agent. The GitHub preset uses `gh issue list --state open --label Sandcastle …`; Beads uses `bd ready --json` ([`src/InitService.ts`](https://github.com/mattpocock/sandcastle/blob/main/src/InitService.ts)). Sandcastle's own dogfood planner filters on `--label ready-for-agent` and skips PRDs that have child issues ([`.sandcastle/plan-prompt.md`](https://github.com/mattpocock/sandcastle/blob/main/.sandcastle/plan-prompt.md)).
- Sandcastle will not bundle large third-party workflow templates such as skill packs ([`.out-of-scope/bundled-workflow-templates.md`](https://github.com/mattpocock/sandcastle/blob/main/.out-of-scope/bundled-workflow-templates.md)). There is no official Sandcastle-plus-skills preset.
- The original Ralph scripts on aihero.dev:
  - [Getting Started With Ralph](https://www.aihero.dev/getting-started-with-ralph): `afk-ralph.sh <iterations>` loops `docker sandbox run claude … -p "@PRD.md @progress.txt …"`. The prompt tells the agent to find the highest-priority task and to "ONLY WORK ON A SINGLE TASK", and ends with "If the PRD is complete, output <promise>COMPLETE</promise>." The script detects it with `[[ "$result" == *"<promise>COMPLETE</promise>"* ]]`. `ralph-once.sh` is the same prompt in a single interactive run (HITL).
  - [11 Tips](https://www.aihero.dev/tips-for-ai-coding-with-ralph-wiggum): "the agent chooses the task, not you". "For AFK Ralph, always cap your iterations." The advice is to start HITL and move to AFK once the prompt is reliable. GitHub issues can replace the PRD as the task source.

### Local copies vs upstream

`diff -r` between each `/opt/t3code/.codex/skills/<name>` (which `/opt/t3code/.claude/skills/<name>` symlinks to) and `mattpocock/skills@a496f55` shows no differences for `wayfinder`, `implement`, `implement-spec`, `to-tickets`, `to-spec`, `handoff`, `research`, or `setup-matt-pocock-skills`. The local copies already include the #1342 implement-spec change. Upstream also has an unrelated in-progress `loop-me` skill (`skills/in-progress/loop-me`). It grills you into workflow specs for recurring patterns in your life; it is not an agent loop runner.

## Implications for the T3 loop design

- **There is no upstream contract to match for "iteration done".** Ralph and Sandcastle end an iteration when the process exits, which is a turn end. The map already rejects that: "a turn ending is not done". T3's MCP done tool is new ground, and nothing upstream conflicts with it.
- **The stop signal should accept the Ralph/Sandcastle convention.** For "nothing left", detect `<promise>COMPLETE</promise>` in the final assistant message alongside the MCP report. This costs almost nothing, works on providers that can't reach T3 MCP (one of the map's open fog items), and lets existing Ralph-style prompts run unmodified as custom loop prompts.
- **Built-in presets must carry the loop contract in the prompt, not in the skill.** The upstream skills will never mention a done tool or a stop string, and upstream won't take a skill change for it (#329, #1175). Sandcastle does the same: "the engine never injects it". A preset is therefore `<skill invocation> <target>` plus a short suffix:
  - When the iteration's work is recorded (wayfinder step 4/5 or commit), report done.
  - If the frontier was empty when you started, report nothing-left instead.

  Skill files stay unmodified.
- **Wayfinder loop: the iteration unit is one upstream session.** That fits as-is: `/wayfinder <map>`, the skill picks the first frontier ticket, claims it by assignee, and resolves exactly one. Watch for these:
  - **Empty frontier is undefined upstream.** The skill has no instruction for it, so the preset suffix must define "frontier empty → nothing left". An empty frontier with open fog, or with blocked or claimed tickets, is not "map clear". Report it as nothing-left-to-take, so the loop pauses rather than declaring the map finished.
  - **Claims block retries.** A failed or stopped iteration leaves its ticket assigned, and the frontier query skips assigned tickets. A resumed loop will silently pick the next ticket. Resume should either name the stuck ticket (`/wayfinder <map> <ticket>`, which the skill supports) or unassign it first.
  - **Most iterations are HITL.** Grilling and prototype tickets need a live human, so the "waiting on user" notification fog item is the common case, not an edge case. Only research, and sometimes task, run AFK. Upstream also recommends one ticket at a time, which matches v1's sequential iterations.
  - **Research AFK resolution has no review gate** (#1241). The loop's pause and manual-override points are where a human can review it.
  - **Research tickets don't fire in this mode (#941).** In work-through mode a graduated research ticket is resolved in-session only when it is the first frontier ticket. Until #941 lands, expect research tickets to be picked as ordinary iterations.
- **Implementation loop: `/implement-spec` is not frontier-per-iteration.** Upstream designed it as one orchestrated run that fans out internally and closes tickets only at the end. Inside a fresh thread per iteration it is one long iteration that does the whole spec. A second iteration's behavior is unspecified: it would make a new integration branch and has no resume logic. This means:
  - The faithful unmodified preset is a loop that is effectively one iteration: `/implement-spec <spec>`, with nothing-left reported at the end, and a cap of about 1–2. Its value is the run's grouping, observability, and done or fail state, not repetition.
  - A sequential frontier-per-iteration build loop (the Ralph shape) maps to `/implement`, not `/implement-spec`. Upstream `/implement` neither picks a ticket nor closes one, so a preset like `/implement <spec>` leaves the tracker frontier stuck. The preset prompt must add the glue Sandcastle's `simple-loop` prompt has: pick the first unblocked `ready-for-agent` child of the spec, implement only it, close it after committing, and report nothing-left when none remain. That is prompt text around an unmodified skill, which is the composition upstream tells people to do themselves (`.out-of-scope/new-skills.md`).
  - Cleanest split: ship both presets. "Implement spec (one run)" wraps `/implement-spec`. "Implement tickets (one per iteration)" wraps `/implement` with the frontier and close suffix. Decide whether the map's "the skill, not T3, picks the ticket" constraint is met by prompt glue in the second preset.
- **Iteration cap: keep it off by default, and expose it prominently for AFK presets.** Both aihero articles insist on a cap for AFK ("always cap your iterations"), and Sandcastle's default is 1. HITL wayfinder loops self-limit through the human.
- **Ticket selection stays tracker-native.** Sandcastle filters in the list command (`ready-for-agent`, `bd ready`), and wayfinder uses the tracker doc's frontier query. T3 should not compute the frontier for built-in presets. That keeps it tracker-agnostic, as the map's out-of-scope note on tracker watching requires.
