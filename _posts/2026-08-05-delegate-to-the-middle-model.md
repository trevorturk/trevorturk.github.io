---
layout: post
title: "Your Best Model Is the Wrong One to Delegate To"
date: 2026-08-05 08:00:00 -0600
summary: "Judgment stays with the best model in the main session. Well-specified briefs go to a pinned Opus 4.8 agent at medium effort, because the newest model adds work nobody asked for. Bulk mechanics go to the cheapest model that can do them."
tags: [ai-agents, workflow, delegation, model-selection]
---

## The Problem

Give the newest, most capable model a small, well-defined task and it comes back with more than you asked for. "Hoist these two computed vars" (compute them once instead of on every pass) comes back as the hoist plus a new abstraction layer. "Apply this diff verbatim" comes back as the diff plus a cache and a protocol for later. "Audit these claims" comes back as the audit plus fixes nobody asked for. The work gets done, but the diff is three times the size of the problem. We saw this for months across the three [Hello Weather](https://helloweather.com) repos (web, iOS, Android). We were following the obvious rule for multi-agent work: use the best model you can afford for everything.

It wasn't just us. When we went looking on 2026-08-04, the vendor's own migration notes and third-party evaluation writeups described the same thing. The newest models add scope to delegated tasks. They over-engineer and pad the deliverable when handed a small brief. The same writeups described Claude Opus 4.8 at medium reasoning effort as the one that does what the brief says. To be clear about what we know: the vendor and eval claims are things we read, and the over-engineering is what we saw in our own repos. We don't have a benchmark to hand you. The two agreed, and that was enough to act on.

The owner put it this way. The newest model is the right one to run the main session because it questions premises, notices nearby problems, and generalizes. Those same habits are the ones you don't want in a subagent that's carrying out a brief someone already scoped. An agent that "improves" the brief isn't being helpful. It's making the diff harder to review.

So we flipped the rule. The best model stays in the session, where the judgment calls are made. Delegated work goes to a less ambitious model, pinned in an agent file checked into each repo.

## The Fix: A Pinned Mid-Tier Agent

All three repos now have `.claude/agents/opus-4-8.md`. It's a Claude Code subagent definition that pins `model: claude-opus-4-8` with `effort: medium`. It's the default target for implementation work we hand off, and its description states the reason in the place every future session will read it:

> The preferred tier for dispatched implementation work — Opus 5 tends to over-engineer; this pin plus bounded effort biases toward simpler, brief-faithful output. Use for clean-room implementations, mechanical refactors, and bounded PR work.

The body of the agent file is a short set of rules about doing the least that satisfies the brief. It's short enough to quote almost whole (lightly sanitized):

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

Each line is there because something went wrong without it. Four need explaining.

"Note it in your report — do not implement it" gives a capable model somewhere to put its urge to improve things. Out-of-scope ideas are welcome, as findings in the report. There the session model, which has the full context, can decide what to do with them.

Scope has two edges, and the rules state both. The agent has to stay inside the brief. It also has to make the changes the brief's change needs in order to compile, even when the brief didn't name them, and it has to follow the repo's standing rules in AGENTS.md. Without that second half, "follow the brief exactly" fails a different way. The agent ships a diff that doesn't compile, or breaks a house rule, because "the brief didn't mention it."

The session that dispatches the agent owns the environment. It sets up the worktree, the branch, and the secrets symlinks. The agent works at the absolute paths it was handed and touches nothing else. That keeps parallel agents from colliding, and it lets the same agent file work in repos with different checkout rules.

Read-only briefs get the same treatment. The pinned agent runs audits and verification passes under the same scope rules, which matters for the routing table below.

## The Routing Matrix: By Kind of Work, Not by Price List

The pin only makes sense as part of a rule for what goes where. Each repo has a model-selection rule. It lived in AGENTS.md when this shipped, and since 2026-08-13 it lives in CLAUDE.md, when agent setup was made tool-agnostic and AGENTS.md kept only the line about never delegating judgment. The rule splits work three ways, by the kind of work rather than by cost:

| Kind of work | Where it runs |
|---|---|
| **Judgment**: planning, architecture, design decisions, ambiguous debugging, review *verdicts*, judgment-heavy reviewer lenses (fresh-eyes reads, devil's advocate) | The frontier session model — never delegated |
| **Bounded execution**: well-specified implementation briefs, mechanical reviewer lenses (claims audits, cross-reference checks), clean-room applies, read-only verification passes | The pinned Opus 4.8 agent, `effort: medium` |
| **Bulk mechanics**: wide searches, boilerplate edits, doc fetching | The cheapest capable tier (Sonnet, via the model override) |

The row that matters most is the split inside review. Our [adversarial review rounds](/adversarial-review-rounds/) post argued that you can hand off a reviewer's brief but not the verdict. This table goes one step further: the reviewer briefs themselves split by kind. (A lens is one reviewer's assigned angle.) A claims audit ("verify every factual assertion in this PR body against the code") is bounded execution. The pinned agent may even be better at it than the newest model, because taking things literally is the job. A devil's-advocate lens ("argue this decision is wrong") is judgment, and stays on the session model. Both are review, but they go to different places because the kind of work differs.

The rule also says the reverse outright: never delegate judgment-heavy work to a lesser model; keep it in the primary session on the most capable model available. The table isn't a ladder you climb down when the budget is tight.

One detail spoils the simple "older models are humbler" story. The same writeups that flagged the newest model as a scope-expander flagged a slightly older neighbor the same way. Opus 4.8 at medium effort was a specific pick, not "anything older than the newest." If you copy this pattern, treat the pin as a guess about one model at one effort setting, and check it on your own briefs.

## Wrinkles We Hit (and Kept)

The pin starts working in the next session, not the one that creates it. Claude Code loads its list of agents when a session starts, so the session that writes `.claude/agents/opus-4-8.md` can't dispatch to it. The plan doc written in that session records the workaround: the first dispatch in the next session also serves as the test that the model ID exists. We had docs saying `claude-opus-4-8` was a valid ID, but we hadn't run it. Whether Claude Code honors `effort` from the agent file can't be checked from the file either. We checked that the same way: dispatch, and see whether the output looks like medium-effort work. If you pin a model in config, expect the first use to be a test, and have a fallback ID ready in case the first one doesn't exist.

We added a second pin, then deleted it. An earlier version also pinned a cheaper model between Opus 4.8 and Sonnet as a fallback. We dropped it for simplicity before copying the setup to the other repos. Two pinned agents means one more choice at every dispatch, and we had no evidence the extra model earned its place. One pin, one override, and the session model is already three destinations.

We kept the model's personality instead of prompting it away. Opus 4.8 is more deliberate than the newest model and quicker to ask a clarifying question. That's what you want in a contractor working from someone else's spec, so the agent file deliberately leaves out "don't ask questions, just proceed." An agent that stops to ask about an unclear brief costs less than one that guesses and moves on.

The agent file went through its own process. The iOS PR that introduced all this (#1468) got a three-reviewer round before merging, and one lens was a claims audit on the agent file itself. We accepted six of roughly thirty findings, including the AGENTS.md and worktree clauses quoted above, which the first draft didn't have. We rejected the rest with reasons. The same PR shipped the review-checkpoint rule that governs that accept-or-reject step, covered in ["Nothing to Change" Is a Valid Verdict](/nothing-to-change-valid-verdict/).

## Did It Work?

It's early, but what we've seen since points the right way. The first real job after the pin started working was an iOS performance PR (#1474). It applied hoists we'd saved up earlier to two widget views. The QA section of the merged PR reads, in part:

> Opus 4.8 mini-review (correctness / side-effects, read-only): **PASS**, zero findings; one completeness nit (empty `hourlyData` now evaluates the three hoisted getters once instead of zero times — safe defaults, unreachable via the `Fallback.hourlyData` path) and a note that the `legendType` hoist is safe because the widget `Entry` viewModel is immutable per view instance.

That's what we wanted from the bounded-execution row. A read-only pass stayed read-only. It returned a clean verdict without inventing work to justify itself, and it still found one useful nit and explained it. No added scope, no "while I was in there." The session model looked at the nit, accepted it as harmless, and merged.

That's one data point. But the problem it replaced, briefs coming back with architecture nobody asked for, showed up every week, so the bar for "better" isn't high.

## Lessons Learned

- **The best model isn't the best one to delegate to.** The habits that make it a good session model, taking initiative and questioning the brief, make it a bad subagent for a scoped brief.
- **Write both edges of scope into the agent file.** "Stay inside the brief" on its own ships diffs that don't compile. Say that the changes needed to compile, and the repo's standing rules, are in scope even when the brief doesn't name them.
- **Pin one model at one effort, and test it.** A slightly older model was flagged as a scope-expander too, so "use an older model" isn't enough on its own.
- **Expect the first dispatch to be a test.** The agent list loads at session start, and a setting like `effort` can only be checked by watching the output.
- **Start with one pin.** Each extra pinned model adds a choice to every dispatch. Add a second only when the first clearly fails at some kind of work.

---

## How This Post Was Made

**Prompt 1:** "see recent work in ~/Code/helloweather, perhaps a blog post about our opus 4.8 agents and why we decided to do that? perhaps something about the swift testing + snapshots inspired by minitest-snapshots? anything else? bring me a list of potential post ideas for review."

**Prompt 2:** "skip 4, 5, 6, 9 but create posts for each of the others in the 1-9 list. also add Four Answers to One Question, and Write the Rule, Not the Story -- show me a concise version of your plan and then I can approve" — then "proceed, one pr per post"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the over-engineered diff instead of the rule that produced it, the bold paragraph lead-ins are plain sentences, and Lessons Learned went from nine bullets to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The agent-file description quote now reads "Opus 5 tends to over-engineer" as the source does, and the PR #1474 quote names the `Fallback.hourlyData` path. The routing rule is now attributed to CLAUDE.md as well as AGENTS.md (it moved there on 2026-08-13), its corollary is quoted without the "to save time" clause the source never had, the bounded-execution row says "well-specified" to match the rule, and the session-start workaround is attributed to the plan doc written in the same session rather than to a plan for the pin itself.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 1, run after the pilot (#67) merged. The prose now reads as someone explaining the setup across a desk: the owner's framing is subject-first with the reason after "because", "contract" became "rules", "hoist" and "lens" are defined where they first appear, "banked" and "before the ports" became "saved up earlier" and "before copying the setup to the other repos", and the newest model is called that throughout instead of frontier, flagship, and top-tier by turns. The title, headings, table, blockquotes, dates, numbers, and links are unchanged, and the one retired-word hit left is the link text "adversarial review rounds". Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"
