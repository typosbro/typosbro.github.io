---
title: The harness matters more than the model
key: harness-matters-more-than-the-model
locale: en
date: 2026-10-26
draft: false
tags: [ai, agents, tools]
excerpt: I ran Claude Code, Codex, pi and omp this year. The model is maybe 40% of the result; the other 60% is the harness around it. Why I moved my whole workflow to DeepSeek Harness, what it costs, and when I would tell you to pick something else.
---

I've spent most of 2026 switching between coding agents: Claude Code, Codex,
pi, omp, and for the last few months DeepSeek Harness.

The lesson I keep relearning is that the model is maybe 40% of the result.
The other 60% is the harness — the agent loop, the tools, how context is
managed, and whether you can see what the agent is doing while it does it.
The same model under a different harness produces visibly different work.

Most harnesses pick a shape and you live with it. DeepSeek Harness is the
first one I've used where the shape itself is a config file.

## Everything is a plugin

DeepSeek Harness (DSH) is an open-source, MIT-licensed agent harness from
DeepSeek. Its thesis is that the model route, the tools, the UI and the agent
loop are all plugins, composed from layered config.

A profile is a stack of layers: the bundle layers first, then your profile's
patch file, then any per-run overlay. Each layer is a list of rows — insert a
plugin, override one by id, or disable one. The dependency tree that falls out
of that is the agent.

On my machine, `dsh --profile web --dump-config` prints 1,537 lines: 361
plugin entries across 12 bundles. That is the entire capability list,
readable and diffable.

Three things follow from that:

1. I can read what my agent can do. No hidden behaviour in a compiled
   binary, no mystery defaults.
2. I can change one layer without forking. The stop gate, the phone relay,
   the permission presets, the cross-session messaging — all rows I added.
   The upstream bundle is untouched.
3. The same harness runs as a web app, a headless CLI, an ACP server for
   editors, or a desktop app. Same plugins, different composition.

The cost is that you own the config. There is no default that matches what I
want, so I maintain a profile repo that reproduces my machine from scratch.

## What my setup actually is

| Piece | What I run |
|---|---|
| Host | A Mac running `dsh web` on loopback, started by a LaunchAgent so it survives reboots |
| Reach | The web GUI in a browser, the same UI on my tailnet, and on my Android phone through a relay plugin over Tailscale |
| Model | DeepSeek V4.1 Flash through an OpenCode Go route, reasoning effort max. The first-party DeepSeek route is also registered. Any OpenAI-compatible or Anthropic-compatible provider is one config row away |
| Bundles | base, web app, agent teams, session search, session titles, worktree tool, voice input, auto-review, ralph loop, relay, local web search, an orchestrator preset |
| Skills | About 20 markdown skills, loaded by trigger rather than all at once |
| Multi-agent | Durable teammates with mailboxes, a shared task board with dependency edges, nested subagents, and JS workflow scripts |
| Guardrails | A stop gate that refuses to end a turn while the repo's own gate fails, advisory write scopes, a worktree per task, an auto-review pass |
| Second opinion | A `consult-codex` skill that shells out to OpenAI's Codex CLI for one read-only review on a separate quota |
| Memory | A separate markdown repo loaded by trigger through an index file, so long-term context costs one line per file |
| Reproducibility | A profile repo with `bootstrap.sh` and a `doctor.sh` that fails loudly when the composition is wrong |

I run it with full filesystem access and approvals off. That is a deliberate
trade for a single-user machine, and I would not hand that default to a
stranger. The public default in the profile repo is workspace-write with
approvals on.

## The four things that changed how I work

### 1. Teams are a product surface, not a prompt trick

Every harness can spawn a subagent. Most of them give you one thing back: a
final message. You cannot see the work while it happens, and you cannot hand
a teammate a task that another teammate is still working on.

DSH ships a real team layer: teammates are durable sessions with their own
mailboxes, and there is a shared task board with dependency edges and
compare-and-set revisions. In the web UI I can see the roster, see the board,
and open any teammate's transcript. When one task depends on another, the
board knows it and will not start it early.

That is the difference between "delegate and hope" and "delegate and watch".

### 2. Scripts, not conversations, for big jobs

Conversational delegation does not scale past a handful of agents. A script
does. DSH has a workflow tool where a JavaScript program decides the DAG —
`agent()`, `pipeline()`, `parallel()`, `phase()` — and intermediate results
live in script variables instead of the chat.

That matters more than it sounds. A twenty-file audit run as a conversation
leaves two hundred thousand tokens of search noise in my context. The same
audit as a workflow returns one structured result and keeps my context clean.
Claude Code has a version of this. Codex and pi do not.

### 3. Different models for different jobs

Multi-agent work costs four to twelve times the tokens of single-agent work.
If every child runs on the same expensive model, that is the whole bill.

Because each child is a plugin composition, I can route per child: a strong
model as the orchestrator, cheap fast models for the workers, and reasoning
effort tuned per call. The harness being model-agnostic is what makes that
possible — it also means I can mount Codex, Claude Code or an ACP backend as
a one-shot child where their tools or models are better.

### 4. One pane of glass, including my phone

The web UI shows the roster, the task board, teammate sessions and workflow
progress. The relay puts the same UI on my phone over Tailscale, so I can
start a job at my desk and check it from the train.

I did not expect to care about this. I care about it a lot. Watching a
fan-out stall in real time beats discovering at the end that three workers
spent an hour on the wrong file.

## What it costs you

Honest list:

- It is a release candidate. 0.2.1-rc.2, repo created in August 2026. The
  package READMEs are good; the public docs are thinner than the product.
  Expect to read source.
- The team layer is experimental. Nested teams are not supported, durable
  storage is required, and the web preset editor saves by plugin id and can
  silently override your file. I have been bitten by "the file says disabled,
  the session says enabled".
- Setup is an afternoon, not five minutes. Patches, credential store, relay,
  tailnet, gate script, profile repo. The bootstrap and doctor scripts make it
  repeatable, not small.
- You are the maintainer. No vendor support, upgrades are your problem, and
  the plugin ecosystem is large and uneven.
- The token bill is real. A heavy day on the first-party API measured about
  USD 12-20, with 98.7% of tokens served from prefix cache and p90 prompts
  around 321K. A fan-out multiplies that.
- It is overkill for small tasks. "Fix this bug in one file" is less friction
  in Codex or Claude Code. The power is in orchestration; if you are not
  orchestrating, you are paying overhead for nothing.
- It is web-GUI first. If you live in a terminal, omp and pi are better
  citizens of one.

## How it compares

| | DeepSeek Harness | Claude Code | Codex CLI | pi | omp |
|---|---|---|---|---|---|
| What it is | open-source harness, plugin tree | Anthropic's coding agent product | OpenAI's coding agent (CLI, IDE, cloud) | minimal terminal harness | pi fork with "the IDE wired in" |
| Licence | MIT | closed source | Apache-2.0 | MIT | MIT |
| Models | any provider row | Claude models only, unofficial proxies aside | OpenAI by default, plus any OpenAI-compatible endpoint, and local models in OSS mode | many providers, switch mid-session | 60+ providers |
| Multi-agent | durable teams plus a shared board plus nested subagents | subagents, experimental teams, dynamic workflows | multi-agent v2: spawn, message, follow-up; children can spawn | none, by design | nested task tree, IRC peer messaging, eval-kernel fan-out |
| Scripted workflows | yes, JS orchestration tool | yes, dynamic workflows with a progress UI | no user-authored script layer | no | close: eval kernel REPL |
| UI | web GUI, plus phone | terminal plus agent panel | CLI, IDE, web app | TUI | TUI plus Agent Hub |
| Maturity | release candidate, about two months old | mature paid product | mature | 1.x, large install base | very fast moving |

So, pick one if:

- Codex CLI — you already pay for a ChatGPT plan, want the least ceremony, and
  mostly want one strong model working one problem.
- Claude Code — you want the most polished productized version of this idea,
  and you are fine with closed source and a subscription.
- pi — you want a small, mature substrate to build your own thing on. It
  deliberately ships no subagents, no plan mode, no MCP. That is the point.
- omp — you want the richest single-agent terminal harness: LSP and DAP wired
  in, hashline edits, nested subagents, an IRC-style peer roster.
- DSH — you want the control plane: durable teams, a shared board, workflow
  scripts, a GUI, phone access, and any model you like, and you accept
  release-candidate status and owning the config.

Maturity ranking: pi, then omp, then DSH. Orchestration surface: DSH, then
omp, then pi. Single-agent polish: omp and Claude Code lead.

## What I'd steal even if you don't switch

1. A stop gate. Refuse to end the turn while the repository's own gate fails.
   It converts "the model says it is done" into "the repository says it is
   done", and it is the highest-leverage line in my whole setup.
2. A second opinion from a different vendor. One read-only review from another
   model on another quota catches a different class of mistake than more
   self-critique does.
3. A memory index. Long-term notes in markdown, one line per file, loaded by
   trigger. Progressive context beats a giant system prompt.
4. One worktree per task. Mechanical isolation beats asking agents to be
   careful.
5. Skills as versioned files with a clear trigger line, not prompts pasted
   into chat.

## Takeaway

Models are rented. The harness is the part you own.

A year of switching taught me that the interesting decisions are not "which
model" but "what can I see, what can I compose, and what survives the next
model release". DSH is the first harness I have used where those answers are
in a file I can read.

It is young, it is a release candidate, and it is more work than installing a
CLI. For what I do with it, that trade has been worth it.

---

*Profile repo: [github.com/typosbro/dsh-web-profile](https://github.com/typosbro/dsh-web-profile).
Harness: [github.com/deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness).
I pay for my own tokens; nobody sponsored this post. Feedback welcome on
[Twitter](https://x.com/typosbro63199).*
