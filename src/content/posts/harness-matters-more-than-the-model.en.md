---
title: The harness matters more than the model
key: harness-matters-more-than-the-model
locale: en
date: 2026-10-26
draft: false
tags: [ai, agents, tools]
excerpt: "The first version of this post argued about 2026 models using 2025 evidence. That was the tell. Here is the corrected version: a harness has two kinds of machinery, compensatory and structural, and the generation boundary decides which one is load-bearing. An audit of my own setup against that test."
---

The first version of this post was wrong in a specific way. I cited a 2025
instruction-following study and a 2025 context-rot study to justify a harness I
run on 2026 models. The evidence was real. It was also about a different
generation, and using it to argue about this one was the tell.

The correction is not "the harness does not matter". It is that a harness has
two kinds of machinery, and a model generation boundary decides which kind is
load-bearing.

- **Compensatory machinery** exists because the model cannot do something yet.
  It expires when the model improves.
- **Structural machinery** exists because the thing is outside what model
  weights can hold: permissions, isolation, durability, coordination,
  verification, accountability, cost.

My setup has both. The honest audit below tags each piece, and about a third of
what I wrote last week fails the test — not the code, the justification for it.

## What the 2026 generation actually changed

This autumn's frontier models are a different animal from the ones the 2025
papers measured.

- Claude Opus 5.5 shipped on 22 September 2026 with a 1M-token context, 128K
  output tokens, always-on adaptive thinking, and roughly 40% lower cost than
  Opus 5 on typical workloads.
- Claude Fable 5.1 is the frontier-reasoning member of the same family: 1M
  context, 128K output, aimed at long-horizon agentic work.
- DeepSeek V4.1 Flash (10 September 2026) is a 552B MoE with 8B/16B active
  parameters, a 1M-token context, MIT weights, and prices that make a fan-out
  affordable.

A million tokens of context is now table stakes. That alone should make anyone
suspicious of a context-budget argument written in 2025.

The more interesting result is instruction following. Arize re-ran the IFScale
benchmark in May 2026 and found that the ceiling had moved an order of
magnitude: where 2025 frontier models started dropping instructions at 200-300
simultaneous constraints, 2026 models hold closer to 2,000, and the strongest
ones stay near-perfect to 5,000. The failure modes changed too — Claude refuses
outright on unlucky word combinations, Gemini burns its output budget on
reasoning and emits nothing, GPT politely explains that the request is
unreasonable and stops.

Arize's own conclusion is the sentence this post should have been built around:

> A year ago, skills files were a compression problem. They aren't anymore. Now
> they're a verification problem.

That is the generation boundary in one line. Compression was compensatory: it
existed because models could not track many constraints. Verification is
structural: it exists because a model's output is not evidence.

## But context rot did not disappear. It moved.

The lazy reading of the above is "long context is solved, delete the context
layer". The 2026 evidence says no.

ContextRotBench (2026) measures long-horizon degradation in coding agents
specifically: 213 tasks across 17 open-source repositories, built on the
observation that a static needle-in-a-haystack test is not what breaks an agent.
What breaks an agent is a trajectory that accumulates — dozens of tool calls,
failed attempts, stale file contents, a growing transcript.

There is a companion result that should worry anyone running unattended agents:
when frontier models are used as classifiers over agent transcripts, they fail
to notice dangerous actions more often as the transcript gets longer, on
transcripts that routinely exceed 500K tokens.

So the problem changed shape. It is no longer "the prompt does not fit". It is
"the trajectory outgrows the model's attention even when it fits the window".
A 1M window does not fix a 40-hour run; it just moves the failure later.

And there is a cost nobody mentions: over-scaffolding now hurts. The 2026
consensus among people running frontier models in production is that heavy
bespoke control flow around a capable model produces worse results than atomic
tools and a plan the model makes itself.

## The classification that matters

The harness-engineering literature has a name for this split. Compensatory
mechanisms wrap a current-generation limitation and are expected to expire.
Structural mechanisms remain valuable regardless of capability — sandboxing,
permission gates, feedback loops, observability. The rule of thumb from that
work is blunt: more model capability is a stronger argument for sandboxing, not
a weaker one.

There is also a test that takes thirty seconds per row:

> If the model got ten times better tomorrow, would this row get cheaper to keep
> or more valuable?

Compensatory rows get cheaper. Structural rows get more valuable. A row that
gets neither is decoration.

And a third category that is neither: **cost**. A resident token costs money on
every request whether or not the model can handle it. Cost is not a capability
problem, so it does not expire with a model release. Confusing the two is how
you end up deleting a cache-friendly design because "the model is smart now".

## The audit

Here is my setup, row by row, against that test.

| Mechanism | Class | Why it exists | Obsoleted by |
|---|---|---|---|
| Stop gate | structural | a model's claim of "done" is not evidence | nothing — more autonomy makes it more necessary |
| Deploy wrapper | structural | irreversible actions need a refusal path | nothing |
| Autonomy A/B/C, receipts | structural | accountability cannot live in weights | nothing |
| Shared board, dependency edges, CAS revisions | structural | coordination across sessions and processes | nothing |
| One worktree per task | structural | parallel writers need filesystem isolation | nothing |
| Schedules delivering instructions | structural | durability plus a human at the decision point | nothing |
| Night-batch fences | structural | unattended risk | nothing |
| Per-child model and effort routing | structural | cost control | partially — cheaper models may flatten the need |
| Independent review (`consult-codex`) | structural | different failure modes, different vendor | partially — if self-verification improves |
| Memory index | structural | state that must survive sessions | nothing |
| Instruction budget, tier ladder | compensatory | 2025 instruction-following limits | 2026 models: ~2,000 constraints |
| Skill-body compression | mixed | attention (compensatory) plus token cost (structural) | the attention half is obsolete; the cost half is not |
| "The model invents conventions" fear | compensatory | under-instruction | tests, affordances, examples — not more prose |
| Subagent isolation | mixed | attention (compensatory) plus trajectory control (structural) | the attention half is obsolete; ContextRotBench says the trajectory half is not |
| `context-audit` trimming | cost | resident tokens per turn, per agent | still valid — but it is cost, not capability |

The pattern is uncomfortable and clear. My Layer 1 (workflow) and Layer 2
(orchestration) are almost entirely structural. My Layer 3 (context) is where
the previous generation is hiding: the instruction budget, the compression
instinct, the fear-driven prose.

The mechanisms in Layer 3 mostly survive. The reasons I gave for them do not.
Progressive disclosure is still worth doing because resident tokens cost money,
not because a 2026 model cannot hold a long instruction file. The memory index
is still worth doing because it is durable state, not because the model needs
its context saved. Subagent isolation is still worth doing because a long
trajectory degrades, not because a single prompt will overflow.

## What I would delete, and what I would keep

If I started today, on this generation:

**Keep, without hesitation.** The gate, the permissions and autonomy levels,
the receipts, the worktree isolation, the board with dependency edges, the
independent reviewer, the durable schedules, the cost routing. None of these
gets easier to skip as models improve. If anything, a model that can run for
hours unattended needs more of them, not fewer.

**Delete.** The belief that a skills file must be compressed. Long, detailed
instruction files are now viable — they are just expensive. The ritual of
trimming a catalog to save 400 resident tokens is a cost optimisation worth a
few minutes, not a design principle. And the habit of adding prose when the
model misbehaves: the 2026 answer to under-instruction is affordances, tests and
examples, not another paragraph.

**Re-examine.** Anything I built to fix a model weakness I observed in 2025 and
have not re-observed since. That is the list that quietly rots.

## So am I re-inventing the wheel?

Half of it, yes — and the half that is a wheel is being absorbed into the
models. The other half is not a wheel, and that distinction is the actual answer
to the question.

The people who study this describe the harness as a layered construct whose
components age at different rates. The runtime layer — infrastructure,
permissions, efficiency tuning — shows no sign of obsolescence, because it sits
structurally outside what weights can absorb. The context and orchestration
layer is domain-dependent: it withers in mature domains like coding, where the
data flywheel has already converted scaffolding into native model behaviour, and
it expands in nascent domains where the model has no training signal.

Coding is a mature domain. That is the uncomfortable part for me: my Layer 3 is
in the layer that is expected to wither in exactly the domain I work in.

But two results cut the other way. First, harness-induced efficiency
differences persist even when accuracy differences vanish: independent
benchmark studies found significant cost-per-task divergence across harnesses
with statistically similar scores. The harness may stop changing what a model
can do before it stops changing what the work costs. Second, permission
scoping and trust boundaries cannot be absorbed by weights at all. A model can
become trustworthy; that does not make it an accountable party.

There is a name for the discipline I should have applied from the start:
harness impermanence. Author scaffolding assuming a future model will subsume
it, and optimise for the cost of removal rather than elegance. Tag every
compensatory mechanism with what limitation it compensates for and what
capability would obsolete it. Keep the seam shallow — a flag and a module, not a
pipeline threaded through the loop. Phil Schmid's version is the one I want
tattooed somewhere: build harnesses that let you rip out the smart logic you
wrote yesterday.

The concrete change I am making to my own repo: every patch row gets an
`obsoleted_by` field. If I cannot fill it in, the row is structural. If I can,
I have a removal condition and I should schedule the re-test.

## What this means for the three layers

The three-layer build log from last week still describes what I run. What
changes is the justification:

1. **Workflow** is the refusal layer. Gates, deploy wrappers, autonomy levels,
   receipts. It appreciates as models get more autonomous, because the more work
   you hand over, the more you need something that can say no.
2. **Orchestration** is the coordination layer. Boards, dependency edges,
   isolation, per-child routing. Models are still one context at a time;
   coordination across contexts is not a model capability.
3. **Context** is the layer that needs the most rethinking. The mechanisms stay
   — cost, durability, trajectory control — but the 2025 rationale has to go.
   Verification replaced compression, and the budget I should be watching is
   the cost of a long trajectory, not the model's attention span.

## What it costs

Unchanged from the last version, and still honest: this is a release candidate,
the team layer is experimental, the web preset editor can silently shadow your
config, 361 plugin entries is 361 liabilities, setup is an afternoon and you are
the maintainer afterwards. A heavy day measured USD 12-20 of tokens on the
first-party API. The unattended batch is still dry-run.

The new line item is attention. Every generation boundary invalidates part of
your rationale, and if you do not re-audit, you carry scaffolding nobody can
justify. The audit above is the price of admission, and it should be run every
time the models move.

## Takeaway

Models are rented. Weights absorb scaffolding. What survives is the layer that
encodes what you refuse to let a model decide alone.

The test is one question: if the model got ten times better tomorrow, does this
row get cheaper to keep or more valuable? Compensation gets cheaper. Structure
gets more valuable. Cost stays cost. Everything else is decoration, and the
first version of this post was decoration with good citations.

Yesterday's harness may be dead. The next one is not.

---

*Profile repo: [github.com/typosbro/dsh-web-profile](https://github.com/typosbro/dsh-web-profile).
Harness: [github.com/deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness).
Evidence: [Arize's 2026 IFScale replication](https://arize.com/blog/llm-instruction-following-benchmark-2026/),
[ContextRotBench](https://openreview.net/forum?id=DiZ67nkfay),
[harness impermanence](https://www.agentpatterns.ai/patterns/agent-design/harness-impermanence/),
[Is the Agent Harness Dead?](https://www.sean-weldon.com/blog/2026-10-22-is-the-agent-harness-dead-jediah-katz-spacexai),
[Claude Opus 5.5](https://www.anthropic.com/claude-opus-5-5),
[Claude Fable 5.1](https://platform.claude.com/docs/en/models/fable-5-1/overview).
I pay for my own tokens; nobody sponsored this post. Feedback welcome on
[Twitter](https://x.com/typosbro63199).*
