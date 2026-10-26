---
title: The harness matters more than the model
key: harness-matters-more-than-the-model
locale: en
date: 2026-10-26
draft: false
tags: [ai, agents, tools]
excerpt: "Everyone argues about models. The harness around the model changes the output more. But even the harness is only a starting point — this is the build log for the three layers I put on top of DeepSeek Harness: workflow, orchestration, and context, with the numbers I measured."
---

Everyone argues about models. I've spent most of 2026 running the same models
through four other coding agents, and the harness changed the output more than
the model did.

But after a few months I think even that framing is too shallow. A harness is a
starting point. Mine is not vanilla DeepSeek Harness — it's three layers I built
on top of it:

1. A **workflow layer** that decides when a task is actually done.
2. An **orchestration layer** that decides how work splits and stays coordinated.
3. A **context layer** that decides what the model actually sees.

Nothing in this post is a feature I got for free from the install. The team
bundle ships; the contract, the gates, the schedules, the deploy wrapper and the
context discipline are mine. That is the part worth writing down, because it is
the part that took the time.

## Why a plugin tree

DeepSeek Harness (DSH) is an open-source, MIT-licensed agent harness from
DeepSeek. Its thesis is that the model route, the tools, the UI and the agent
loop are all plugins, composed from layered config. A profile is a stack: bundle
layers first, then your patch file, then any per-run overlay. Each layer is a
list of rows — insert a plugin, override one by id, disable one.

On my machine, `dsh --profile web --dump-config` prints 1,537 lines: 361 plugin
entries across 12 bundles. That is the whole capability list, readable and
diffable, which matters more than it sounds: I can change one layer without
forking, and the same harness runs as a web app, a headless CLI, an ACP server
for editors, or a desktop app.

The rule I hold myself to: every row has a written reason next to it in
`why-these-rows.md`, and a row that cannot justify itself gets deleted. A config
you cannot defend is a config you cannot debug.

## Layer 1: workflow — what decides a task is done

An agent can run a task. A workflow setup runs a week. Without one, you are the
scheduler, the QA and the memory, and the agent is a very fast intern who keeps
asking what to do next.

Four pieces do most of the work.

### A stop gate with teeth

The gate is a hook at the point where a turn is about to end. It looks for the
repository's own check, in order: `scripts/agent-gate.sh`, then the
`package.json` gate, then `gate:fast`, then a diff-level review stage that
blocks on leaked keys, conflict markers and `debugger` statements. A clean git
tree skips everything. A failure exits 2 and its stderr becomes the steering
reason the model sees — so the model gets the actual test failure, not a generic
"blocked". At most two forced continuations per session, then it stands down.

Those guards are not politeness. A gate that can trap a turn forever is a gate
the user disables within a day. The effect is that "the model says it is done"
becomes "the repository says it is done", which is the highest-leverage change
in the whole setup.

### Schedules that deliver instructions, not actions

I have two durable schedules: a daily brief at 13:00 and a weekly review on
Friday at 13:00, Asia/Seoul. They survive restarts because they live in the
harness, not in cron.

The important part is what they deliver. The daily brief is a script that
gathers facts — git state, CI, open PRs, what the board says is Now, the latest
Done, the live bundle, cron health, unchecked owner blockers — and then the model
writes the sentences. The brief itself is read-only. The schedule is a reminder
system with a research assistant attached, not a cron job with commit rights.

An autonomous agent with a calendar is an incident generator. Delivering an
instruction into a durable session keeps a human at the decision point.

### A deploy wrapper with more gates than I have patience for

The deploy path is one script, and it refuses: a dirty tree, a non-main HEAD, a
HEAD that differs from a fetched `origin/main`, any CI state other than a
completed success for that exact commit, pending migrations, and a skipped
figure build it cannot prove against the deploy record and the manifest.
`--dry-run` prints the mutations instead of making them.

On its first real use it shipped a deploy, proved the skipped build against a
manifest of 530 artifacts, ran a 29-check smoke, and updated the deploy record.
Rollback is named in advance.

The dangerous step is the one you automate last, and only after it can refuse.
That is the whole design: the wrapper is the only path from green main to
production, and it is allowed to say no for six different reasons.

### A night batch that defaults to dry-run

Unattended work exists, but it is fenced. The batch reads an explicit task list,
enforces a task-count cap, a per-task wall-clock cap and a whole-batch budget,
refuses task text that looks like an outbound send, gives every child exactly one
task so a poisoned task cannot rewrite the plan, and writes one local brief per
task. It defaults to a dry run; only `--run` executes.

The honest state: it is still dry-run only, because the headless route's
credential has not been refreshed. It stays that way until it authenticates. A
safety model you have not exercised is a claim, not a control.

### A second opinion from a different vendor

`consult-codex` runs one read-only question through OpenAI's Codex CLI
(`codex exec --json`, 600-second cap, separate quota) and prints the answer. It
never edits files, installs anything or writes config. For a diff review it gets
the git diff against HEAD.

Same-model self-critique shares blind spots with the thing it is reviewing. A
different vendor, a different model family and a different quota pool is the
cheapest way to get an independent read on a risky change.

The division of labour underneath all of it: scripts gather facts, models write
sentences, humans approve irreversible actions. And the whole thing is
reproducible — `bootstrap.sh` pins the harness version, installs the profile
dependencies and syncs the skills; `doctor.sh` proves the composition and fails
loudly when it is wrong.

## Layer 2: orchestration — how work splits and stays coordinated

There are two shapes of multi-agent work, and they are not competitors.

- **Conversational teams**: the model decides turn by turn who does what. Good
  for long-running peers and work that changes shape as it goes.
- **Script-defined workflows**: a JavaScript program decides the DAG. Good for
  fan-out where the shape is known up front.

DSH ships both, and I use both.

### What the team layer actually is

Nine tools: spawn, message, list, wait, interrupt, and four task-board
operations. Teammates are durable sessions with their own mailboxes. The shared
board is the coordination substrate; messages are secondary.

Two board mechanics carry the weight:

- **Dependency edges.** A task cannot start before its blockers finish.
- **Compare-and-set revisions.** A task update carries the revision it read, so
  two workers cannot silently overwrite each other's claim.

Write scopes are declared per task and they are advisory: the harness warns when
two in-progress tasks overlap, and never locks anything. That is deliberate.
Bash can bypass any advisory lock, so pretending otherwise would be security
theatre.

The shape of a task on the board:

```js
team_task_create({
  subject: "Migrate billing calls to the new client",
  description: "Scope: src/billing/**. Gate: npm run gate.",
  blocked_by: ["task-a1"],        // cannot start early
  write_scopes: ["src/billing/"], // overlap warning, not a lock
});
```

### Isolation is mechanical, not polite

Advisory scopes are not enough, so each task gets a git worktree. The worktree
tool and a small `wt.sh` wrapper create one per task and run the repository gate
on finish. That is the pattern that actually holds across every serious
orchestration setup I looked at: one worktree or branch per worker, because
filesystem isolation is the only reliable conflict fix.

Nested delegation is on: the host default depth is 1, and my orchestrator preset
raises it to 2, so a child can delegate again. Each spawn can pick its own child
preset, model and reasoning effort.

### Workflow scripts

The workflow tool takes a JavaScript program with `agent()`, `pipeline()`,
`parallel()`, `phase()` and `log()`. Intermediates live in script variables, the
result is capped at 50k characters, and the run's lifecycle is recorded to the
session.

A simplified shape of an audit:

```js
const files = await agent("list every file that imports the old client", {
  schema: filesSchema,
});

const findings = await parallel(
  files.map((f) => () => agent(`review ${f} for the old client`, {
    schema: findingSchema,
  })),
);

phase("verify");
const confirmed = await pipeline(findings.filter(Boolean), (f) =>
  agent(`verify this claim against the repo: ${f.claim}`, { schema: verdictSchema })
);

return confirmed;
```

Why this matters: the same audit run as a conversation leaves a couple of
hundred thousand tokens of search noise in my context. Run as a workflow it
returns one structured result and keeps my context clean. That is the whole
argument for scripts over conversations once the fan-out gets wider than a
handful of agents.

### Cost control

Multi-agent work costs four to twelve times the tokens of single-agent work. If
every child runs on the same expensive model, that is the entire bill.

Because each child is a composition, I route per child: a strong model as the
orchestrator, cheap fast models for the workers, reasoning effort tuned per call.
And I measure it. A `usage-meter` script reads the persisted session logs and
prints per-turn and per-session tokens, including cache reads. A `bench` script
runs one task on two model routes and keeps both transcripts, exits and
durations. Harness changes get measured, not argued.

### The operating contract

This is the part nobody ships, and the part that makes a team usable instead of a
toy.

Roles are fixed: a Lead that plans, dispatches, owns board truth, reviews and
merges; repo workers; a verification worker; a verifier; a finance scribe. The
author is never the reviewer.

Autonomy has three levels:

- **A** — autonomous, reviewed after. Branches, tests, docs, research, analysis.
- **B** — autonomous with a gate. Agents prepare branches and PRs; the Lead runs
  the verifier and merges. Production deploys and production data writes are
  Level B, but executed by the Lead only, never by a teammate.
- **C** — human-only. Money movement, every external send, partner and legal
  contact, DNS and billing, secrets, public income claims, root-cause diagnosis.
  Not permission withheld — the list of surfaces an agent cannot or must not
  write to.

Reports have a fixed shape: verdict first, then receipts — a path, a line
number, a command output — then at most one decision the owner has to make.
Brief cap is 150 words. Numbers are generated, not typed: a claim without a
generated artifact is a draft. And the rule that keeps the record honest: a
decision that only exists in chat did not happen.

The Lead is also allowed to be argued with. A delegated agent that argues with
the brief is doing its job.

## Layer 3: context — what the model actually sees

Context is the budget. Every resident token is a token not spent on the task,
and the tax is paid on every single request.

So I measured mine.

`context-audit` reads `AGENTS.md` and the skill catalog and separates what is
resident from what is on demand. On this machine, 21 files:

```
resident:   ~1,645 tokens   (AGENTS.md + ~20 skill descriptions, every request)
on-demand:  ~24,253 tokens  (skill bodies, only when invoked)
```

That is roughly a 15:1 ratio. The catalog is cheap; the bodies are not, and they
only load on trigger. The largest single body is 2,333 tokens. The six fattest
descriptions run 86-130 tokens each, and trimming those six to 40-50 tokens
would save roughly 400-450 resident tokens without touching a single body.

The audit also caught something I would not have found by reading: the session
catalog shows 18 of the 20 skills on disk. Two are unreachable, most likely a
name collision or a disabled registration. A skill you cannot select is a skill
that does not exist.

### Progressive disclosure

A skill is a directory with a `SKILL.md`. The name and description are resident;
the body loads when the model decides the skill is relevant; linked files and
scripts load deeper. The memory tree works the same way: `MEMORY.md` is an index,
one line per file with a load trigger, and the index is explicitly not truth —
load by topic, and the current message wins over memory.

The index costs a line per file. The body costs nothing until it is used.

### Isolation by construction

Subagents throw away their search noise and return a result. Workflow
intermediates live in variables. The chat stays clean because the exploration
never enters it. This is cheaper than any compaction strategy, because the
tokens were never spent in the first place.

### Per-agent composition

A worker that cannot use `schedule`, `jobs`, `present` or `web` should not pay
for their tool schemas. That is a preset-level lever, and in practice it saves
more than editing prose does. Every agent pays only for the tools it can call.

### The decision ladder

Before adding an instruction, decide which tier it belongs to:

| Tier | Where it goes | Rule |
|---|---|---|
| 0 | hook, gate, permission, test, CI | if it must hold, enforce it; prose is advisory |
| 1 | `AGENTS.md` / system prompt | only what applies on every request |
| 2 | skill | a reusable procedure, body on demand |
| 3 | repo file or doc | large or rarely needed, read on demand |
| 4 | memory file, goal, task board | state that must survive turns |
| never | duplicated, stale, or restated tool behaviour | delete it |

Three copies of a rule is a stale rule. Keep one copy and link to it.

### Why this is the right shape

The evidence is not mine, and it all points the same way:

- IFScale put 500 instructions in one prompt; the best frontier models reached
  68% at maximum density, degraded non-uniformly, and showed a bias toward
  earlier instructions. Every instruction dilutes the ones after it.
- Chroma's context-rot work showed accuracy degrading as input grows, even on
  trivial tasks.
- Anthropic's context-engineering guidance describes an attention budget and
  recommends the smallest set of high-signal tokens, retrieved just in time.

One number of my own makes the point concrete. Measured over 19 hours of real
traffic on 2026-09-22: 98.7% of tokens were served from prefix cache, the p90
prompt was around 321K, and 13% of requests were above 262K. Big contexts only
work because most of them are cached — which is exactly why stable ordering and
a small resident set matter.

The two failure modes are worth naming. Over-instruction makes a prompt brittle,
stale and diluted. Under-instruction makes the model invent conventions. The cure
for the second is not more prose — it is affordances, repo conventions, tests and
examples.

## What I took from the other harnesses

Not a spec comparison. These are the specific mechanisms I borrowed.

| Mechanism | Best example I found | What I run |
|---|---|---|
| Coordination substrate | DSH's board with dependency edges and CAS revisions | durable board plus mailboxes |
| Isolation | omp's per-task worktrees and patch artifacts | one git worktree per task |
| Script-defined fan-out | Claude Code's dynamic workflows | the workflow tool |
| Independent review | Codex on a separate quota | `consult-codex` |
| Prompt minimalism | pi's four tools and small system prompt | a measured 1,645 resident tokens |
| Nested delegation | Codex's children-spawn-children model | subagent depth raised to 2 |

The comparison that actually matters is not which harness wins. It is that all
four have converged on the same two shapes — conversational teams and
script-defined workflows — and the interesting differences are in how they
coordinate and isolate. DSH is the one that let me compose all of the above in a
single tree.

## What it costs

Honest list:

- It is a release candidate. 0.2.1-rc.2, repo created in August 2026. The
  package READMEs are good; the public docs are thinner than the product.
- The team layer is experimental. Nested teams are not supported, durable
  storage is required, and the web preset editor saves by plugin id and can
  silently shadow your file. I have been bitten by "the file says disabled, the
  session says enabled".
- 361 plugin entries is 361 liabilities. `doctor.sh` catches drift in the
  composition, not bad judgment in the design.
- Setup is an afternoon, not five minutes, and you are the maintainer afterwards.
  No vendor support, upgrades are your problem.
- The token bill is real. A heavy day on the first-party API measured about USD
  12-20, and a fan-out multiplies it.
- It is overkill for small tasks, and it is web-GUI first. If you live in a
  terminal, omp and pi are better citizens of one.
- The unattended batch is still dry-run, and the headless credential is an open
  owner task. I would rather say that than pretend the automation is finished.

## Takeaway

Models are rented. The harness is a starting point. The operating layer is the
asset.

Three questions decide whether an agent setup is real or a demo: what decides
that a task is done, what decides how the work splits, and what the model
actually sees. If those answers live in a config file you can read, a gate that
can refuse, and an index that loads by trigger, you have something you can run a
week on. If they live in a prompt you retype every morning, you have a demo.

---

*Profile repo: [github.com/typosbro/dsh-web-profile](https://github.com/typosbro/dsh-web-profile).
Harness: [github.com/deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness).
Evidence: [IFScale](https://arxiv.org/abs/2507.11538),
[Chroma context rot](https://research.trychroma.com/context-rot),
[Anthropic on context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents).
I pay for my own tokens; nobody sponsored this post. Feedback welcome on
[Twitter](https://x.com/typosbro63199).*
