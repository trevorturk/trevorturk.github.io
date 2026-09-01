---
layout: post
title: "Your Best Model Is the Wrong One to Delegate To"
date: 2026-08-05 08:00:00 -0600
summary: "Route delegated work by kind, not capability: judgment stays on the frontier session model, bounded briefs go to a pinned mid-tier model with bounded effort, and bulk work goes to the cheapest tier."
tags: [ai-agents, workflow, delegation, model-selection]
---

## The Problem

Hand the newest, most capable model a bounded brief and it comes back with more than the brief. "Hoist these two computed vars" returns the hoist plus a new abstraction layer. "Apply this diff verbatim" returns the diff plus a cache and a protocol for future extensibility. "Audit these claims" returns the audit plus fixes nobody asked for. The work is done, and the diff is three times the size of the problem. We saw this for months across the three [Hello Weather](https://helloweather.com) repos (web, iOS, Android), because we were following the intuitive rule for multi-agent work: use the best model you can afford, everywhere.

It wasn't only our anecdote. When we went looking (2026-08-04), the model vendor's own migration notes and third-party eval writeups described the same behavior. The newest top-tier models are documented scope-expanders on delegated tasks: they over-engineer and inflate deliverables when handed a bounded brief. The same writeups described Claude Opus 4.8 at medium reasoning effort as the literal, brief-faithful executor. To state the epistemic status plainly: the vendor and eval claims are things we read, and the over-engineering is our observed behavior in our repos, not a benchmark result we can hand you. But the two agreed, which was enough to act on.

The project owner's framing stuck. What makes a frontier model the right *session* model is that it questions premises, notices adjacent problems, and generalizes. That is exactly the trait you don't want in a subagent executing a brief someone else already scoped. A dispatched agent that "improves" the brief isn't being smart; it's making the diff unreviewable.

So we inverted the rule. The best model stays in the session, where judgment lives. Delegated work goes to a deliberately *less* ambitious tier, pinned in a checked-in agent file.

## The Fix: A Pinned Mid-Tier Agent

All three repos now carry `.claude/agents/opus-4-8.md`, a Claude Code subagent definition pinning `model: claude-opus-4-8` with `effort: medium`. It is the preferred target for dispatched implementation work, and the frontmatter description states the reasoning where every future session will read it:

> The preferred tier for dispatched implementation work — the top tier tends to over-engineer; this pin plus bounded effort biases toward simpler, brief-faithful output. Use for clean-room implementations, mechanical refactors, and bounded PR work.

The body of the agent file is a "least machinery" contract, short enough to quote nearly whole (lightly sanitized):

> You are an implementation agent for this repository. Follow the brief you are given exactly and stay strictly inside its stated scope. Briefs may also be read-only (audits, verification); the same scope discipline applies.
>
> Core discipline, non-negotiable:
>
> - Bias to the least machinery that satisfies the brief. No abstractions, generalizations, caches, protocols, or "future-proofing" beyond what the brief names. If you notice a possible improvement outside scope, note it in your report — do not implement it.
> - Changes required to make the brief's change correct and compiling — call sites, target membership, affected tests — are in scope even when unnamed.
> - Read the repository's AGENTS.md before changing anything and obey it. Its invariants (crash prevention, localization, comment policy, testing) are in scope for every brief even when the brief does not name them.
> - The caller owns branch and worktree setup: work only at the absolute paths in your brief, and never create worktrees or branches unless the brief says to.
> - Prefer small, obvious code over clever code. A reviewer must be able to read your diff top to bottom in one sitting.
> - Pass absolute paths in every file operation.

Every line exists because something went wrong without it. Four are worth unpacking.

"Note it in your report — do not implement it" is the release valve for a capable model's urge to improve things. The observation isn't suppressed; out-of-scope ideas are welcome as *findings*. It is rerouted from the diff to the report, where the session model with full context can judge it.

Scope has two edges, and the contract states both. It is strict about staying inside the brief, and equally explicit that *consequential* changes are in scope even when the brief forgot to name them: call sites, target membership, affected tests, and the repo's standing invariants from AGENTS.md. Without those clauses, "follow the brief exactly" produces a different failure. A technically compliant agent ships a non-compiling diff, or violates a house rule, because "the brief didn't mention it."

The caller owns the environment. Worktrees, branches, and secrets symlinks are set up by the dispatching session. The subagent works at the absolute paths it was handed and touches nothing else, which keeps parallel dispatches from colliding and keeps the contract portable across repos with different checkout rules.

Read-only briefs are first-class. The same pinned agent runs audits and verification passes under the same scope discipline, which matters for the routing matrix below.

## The Routing Matrix: By Kind of Work, Not by Price List

The pin only makes sense inside a policy for *what goes where*. The AGENTS.md model-selection rule in each repo now encodes a three-way split, and the axis is the kind of work, not cost:

| Kind of work | Where it runs |
|---|---|
| **Judgment**: planning, architecture, design decisions, ambiguous debugging, review *verdicts*, judgment-heavy reviewer lenses (fresh-eyes reads, devil's advocate) | The frontier session model — never delegated |
| **Bounded execution**: implementation briefs with verified scope, mechanical reviewer lenses (claims audits, cross-reference checks), clean-room applies, read-only verification passes | The pinned Opus 4.8 agent, `effort: medium` |
| **Bulk mechanics**: wide searches, boilerplate edits, doc fetching | The cheapest capable tier (Sonnet, via the model override) |

The line that does the most work is the split *within review*. Our [adversarial review rounds](/adversarial-review-rounds/) post already argued that reviewer briefs are delegable and the verdict is not. This matrix sharpens it: reviewer lenses themselves divide by kind. A claims audit ("verify every factual assertion in this PR body against the code") is bounded execution, and the literal-minded mid-tier is arguably *better* at it than a frontier model, because literalism is the job. A devil's-advocate lens ("argue this decision is wrong") is judgment, and stays on the session model. Same activity, opposite routing, because the underlying kind of work differs.

The rule states the corollary outright: never delegate judgment-heavy work to a lesser model to save time. The matrix is not a budget ladder you slide down under pressure. Each row goes where it goes.

One nuance breaks the tidy "older models are humbler" story. The same writeups that flagged the newest tier as a scope-expander flagged an *adjacent, slightly older* version the same way. Opus 4.8 at medium effort was a specific pick, not "anything older than the flagship." If you adopt this pattern, the pin is a hypothesis about one model at one effort setting. Validate it on your own briefs.

## Wrinkles We Hit (and Kept)

The pin activates next session, not this one. Claude Code loads the agent registry at session start, so the session that *writes* `.claude/agents/opus-4-8.md` cannot dispatch to it. The plan doc for the program that shipped this records the workaround as policy: the first dispatch in the next session doubles as the model-ID existence probe. We had documentation that `claude-opus-4-8` was a valid model ID, but we hadn't run it. Whether the harness honors `effort` from agent frontmatter isn't observable from the file at all, so that too was validated behaviorally: dispatch, and watch whether the output looks like medium-effort work. If you pin a model tier in config, plan for the first use to be a probe, and have a fallback ID in mind if the primary doesn't exist.

We shipped a second pin, then deleted it. An earlier revision also pinned a step-down model as a cheaper fallback tier between the mid-tier and Sonnet. It was dropped for simplicity before the ports. Two pinned agents means every dispatch decision has one more branch, and we had no evidence the middle-middle tier earned its slot. One pin, one override, one session model: three destinations is already enough matrix.

The model's personality was kept, not prompted away. Opus 4.8's observed character, more deliberate and quicker to ask clarifying questions than the flagship, is the temperament you want in a contractor executing someone else's spec. So the agent prompt deliberately does not include "don't ask questions, just proceed." A dispatched agent that stops to ask about a genuinely ambiguous brief is cheaper than one that confidently guesses.

The policy text ate its own dogfood. The iOS PR that introduced all this (#1468) was stress-tested by a live three-reviewer adversarial round before merging, and one lens was a claims audit *on the pinned agent file itself*. Six of roughly thirty findings were accepted, including the invariants and worktree clauses quoted above, which the first draft lacked. The rest were rejected with reasoned objections. The same PR shipped the review-checkpoint policy that governs that filtering flow, covered in a sibling post, ["Nothing to Change" Is a Valid Verdict](/nothing-to-change-valid-verdict/).

## Did It Work?

Early, but the follow-through evidence points the right way. The first real workload after the pin activated was an iOS performance PR (#1474), applying banked loop-invariant hoists to two widget views. The QA section of the merged PR reads, in part:

> Opus 4.8 mini-review (correctness / side-effects, read-only): **PASS**, zero findings; one completeness nit (empty `hourlyData` now evaluates the three hoisted getters once instead of zero times — safe defaults, unreachable via the fallback path) and a note that the `legendType` hoist is safe because the widget `Entry` viewModel is immutable per view instance.

That is the shape we wanted from the mechanical-lens row of the matrix. A read-only verification pass stayed read-only, returned a clean verdict without inventing work to justify itself, and still surfaced one useful nit with the reasoning attached. No scope expansion, no "while I was in there." The session model adjudicated the nit (accepted as harmless) and merged.

One data point is one data point. But the failure mode this replaced, delegated briefs coming back with uninvited architecture, produced its evidence weekly, so the bar for "better" is not high.

## Lessons Learned

- **Capability and delegability are different axes.** Initiative, generalization, and premise-questioning make the best session model and the worst subagent for a scoped brief.
- **Write both edges of scope into the contract.** "Stay inside the brief" alone ships non-compiling diffs. Consequential changes and repo invariants are in scope even when unnamed.
- **The pick is specific, not directional.** An adjacent older model was flagged as a scope-expander too. "Use a humbler model" means one model at one effort, validated on your own work.
- **Expect config to lag reality.** Agent registries load at session start, so the first dispatch is your existence probe, and settings like `effort` may only be verifiable behaviorally.
- **Prefer one pin over two.** Every extra pinned tier adds a branch to every dispatch decision. Add a second only when the first demonstrably fails a class of work.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the over-engineered diff instead of the rule that produced it, the bold paragraph lead-ins are plain sentences, and Lessons Learned went from nine bullets to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
