---
layout: post
title: "Plans Are Disposable, Skills Are Durable"
date: 2026-07-29 09:20:00 -0600
summary: "How we manage the plans agents write for other agents: what a plan has to contain so a later session can build from it, when we delete it, what we keep, and how the index answers 'what's next?' when the answer is 'nothing'."
tags: [ai-agents, planning, workflow, documentation]
---

## The Problem

One of our repos had 135 files in its `plans/` directory. Many of them said `Status: Implemented` or called themselves a "durable record", and nobody had deleted them because of those labels.

We wrote about [our planning skill](/planning-skill/) in February. A plan was a markdown file in `plans/`, listed in an index, and deleted when the work shipped. After five months of using it across three [Hello Weather](https://helloweather.com) repos, we could see it covered how to write a plan and not what happens to it afterward. The pile-up brought three more problems with it:

1. **Plans go stale.** A plan written in March cites `app/models/thing.rb:117`. By June that line has moved to 145, the method has a new name, and the library version in the plan is two major versions behind. An agent building from that plan will build the wrong thing without noticing.
2. **Sessions end.** The agent that wrote the plan knew the codebase. The agent that implements it three weeks later knows nothing, and may be a smaller model. Anything the plan leaves out, every later reader has to go and find again.
3. **Agents always find something to do.** Ask "what's next?" and an agent will answer. Sometimes the true answer is "nothing, wait for the trigger." Unless a rule says that answer is allowed, the agent invents work, usually a refactor, and usually the most expensive option available.

We didn't fix this with a better template. We wrote a policy for the whole life of a plan: what it has to contain, when we delete it, what we keep from it, and what the index is allowed to say.

## The Lifecycle

```
Research → Write (handoff-ready) → Rank in the index → Implement
    → Migrate durable knowledge into a skill → DELETE the plan → Update the index
```

The policy uses three kinds of document. Most doc rot comes from mixing them up:

| Type | Lifespan | What it holds |
|---|---|---|
| **Plan** | One-shot — deleted after it ships | This specific change: steps, verified paths, success criteria |
| **Playbook** | Permanent, versioned | A procedure that recurs verbatim per instance |
| **Skill** | Permanent, versioned | A workflow, plus the invariants and gotchas plans leave behind |

Playbooks come back at the end of this post. They're the one thing in `plans/` we don't delete.

## Ground Truth: Write for a Reader Who Won't Explore

Every plan now has a `## Ground Truth` section. The planning skill in all three repos states the standard the same way:

> Write plans so a less capable model can implement them without exploring the codebase.

If the implementing agent has to grep to find where things register, the plan has failed. A smaller model might grep, guess wrong, and ship the guess. Four rules follow from that.

**1. Cite verified paths with line numbers.** Here is a real section, verbatim except for the header:

> **Ground Truth (codebase-verified 2026-06-11)**
>
> *Key files*
>
> | File | Line | Purpose |
> |------|------|---------|
> | `app/models/api/weather.rb` | 26-27 | Coordinate truncation: `truncate(3)` |
>
> *Current implementation*
>
> ```ruby
> # app/models/api/weather.rb:26-27
> @lat = (@args[:lat] || @source_class.default_lat).to_f.truncate(3)
> ```
>
> *Change required*
>
> Change `truncate(3)` to `truncate(2)` on both lines.

The skill's rule is *never cite a symbol you haven't confirmed exists.* The header carries the date the paths were checked, so a reader can tell when a citation might have gone stale.

**2. Name the registration points.** In an unfamiliar codebase, writing the new file is cheap. Finding the four places that have to know about it is expensive. We went through 32 plans in the web repo and added a key-files table to each one. The tables name the constant lists a new entry goes in and the factory method that maps a name to a class. The same pass fixed patterns the plans had made up: a base class that didn't exist, and an environment-variable lookup where the codebase uses a helper. A plan written from memory invents house conventions.

**3. Date every external fact with a `Verify:` marker.** Anything the codebase can't confirm, such as a library version or a platform deadline, gets flagged where it appears.

```markdown
| cloud_cover | `cloud_cover` (%) | Recorded; Verify: 0-100 integer encoding per the API spec |
| alert id | Verify: exact key name (`id` vs `alert_id`) | Verify |
```

The skill's rule is *never present external facts as timeless; date them and flag them for re-checking.* The implementation checklist has the agent resolve every `Verify:` marker before building on it. If a marker is still there at code review, the plan did its job and the implementation skipped a step.

**4. Mark what's cuttable.** When a plan feeds a release with a deadline, each item is labeled required or cuttable:

> **Effort honesty**: mark items cuttable vs required when they feed a release block, so time pressure cuts the right things.

Under time pressure, something gets dropped. Whoever drops it usually knows less than the author did, so the author marks it in advance. One release block also ends with a list of things not to do, *"Explicitly NOT required for this release (common trap — these feel related but aren't)"*, followed by seven upgrades that look related. Writing that list once is cheaper than having every session rediscover it and argue about it.

One more rule applies when the plan is implemented:

> **If the plan conflicts with existing patterns** — Follow the patterns (AGENTS.md, skills), update the plan.

When the plan and the codebase disagree, the codebase wins.

## Deletion: A "Record" Label Is a Delete Signal

The February post said "delete when done." After 135 files, the rule got more specific:

> A plan is **removed** once the work it describes is **fully implemented / shipped**, or it is a **completed one-time analysis** — *regardless of what the plan calls itself*. A "Status: Implemented", "implementation record", "durable record", or "reference" self-label is **not** a reason to keep a plan; it is a signal to remove it (after migrating anything durable).

A plan doesn't call itself a record because it's valuable. It calls itself a record because someone finished the work and didn't want to delete the file.

Only three things justify keeping a plan:

1. **Future / unstarted work** — not built yet.
2. **Triggered / dormant work** — reopens on a monitoring or external trigger, *and the plan still holds the live reopen playbook*.
3. **An active in-flight decision** — pending actions or dates.

Anything else is finished: move out what's worth keeping, then remove the plan. Being linked from the index isn't a reason to keep a plan, because the index links to whatever exists. When we're not sure, the skill says to keep the plan and say why: *when uncertain, prefer keep and say why — deleting curated plans is the risky direction.*

### Migrate before you delete

> Durable knowledge a finished plan holds — non-obvious decisions, vendor/API facts and gotchas, operational procedures, hard-won rationale — must be **moved into the relevant skill** (**preferred**), or README/AGENTS.md, *before* the plan is removed. Never keep a plan alive just to hold reference material, and never delete without first rehoming what's worth keeping.

In the web repo, one pull request removed 35 plans and moved what they held into a dozen skills. We kept the things that still matter after the change ships: three optimizations that didn't work, so nobody tries them again; quirks in upstream APIs; capacity thresholds and the conditions for reopening them; vendor evaluations we rejected. We deleted the story of how we got there.

We ran the audit with agents. One classified all 135 plans. For every plan marked for removal, a second agent argued the case for keeping it. Then one agent per file did the migration and repointed the links that led to the removed plan. We checked the result with a script rather than by reading:

```bash
# Orphans: a plan file the index doesn't point to is a bug
for f in plans/*.md; do
  b=$(basename "$f")
  grep -q "$b" PLANS.md || echo "ORPHAN: $b"
done
```

An earlier round of deletions had left 15 broken links in the plans that survived, so removal now has a required step *before* `rm`:

```bash
# Find every plan linking to the one being removed, redirect them in the same commit
grep -rln "plan-name.md" plans/
```

## The Index Is a Ranking Surface, Not a Document Store

`PLANS.md` had grown to 380 lines, much of it prose copied from the plans. This policy cut it to 240:

> PLANS.md is a navigation and ranking surface, not a document store. Duplicated prose goes stale the moment the underlying plan moves.
>
> - **Every entry is 1–3 lines + a link.** If you're writing more than that in the index, the extra belongs in the linked plan.
> - **No implemented-work tracking, dated pass logs, or history in the index** — record only the current state and what's remaining.
> - **Status updates replace, they don't accrete.** When state changes, rewrite the entry's current-state line; don't append another dated bullet under the old ones.
> - **Content with no plan doc to link to stays very concise.** If it needs more, create the plan doc and link it.

The rule that matters most is "replace, don't accrete." An agent updating a status file wants to append, because appending feels safer and keeps the history. It also turns a 3-line entry into a 40-line changelog nobody reads. Dated history is fine *inside* a plan that's still in progress, because it goes away with the plan.

The same pass moved each status board next to the procedure it tracks. A multi-step rollout used to tick off progress in the central index. Now the checklist lives inside the runbook, and each rollout PR updates its own line. The procedure and its state are in one document, and the change that moves the work along also updates the state. Before, they were two documents that stayed in sync only if someone remembered.

Every plan also needs a place in the ranking, not just a catalog entry:

> Every plan creation/removal MUST update `PLANS.md` — both the **Plan Catalog** entry AND its slot in the ranking machinery (a Release block item, a Triggered Work row, or a Not Next row with an explicit reopen condition). A plan that exists only in the catalog has no priority and will never be picked.

## The What's-Next State Machine

The Android index answers "what's next?" with a fixed five-step check. Step 4 was added the day after this post, when we started using GitHub issues to capture the tangents that come up mid-session:

```markdown
### What's Next Rule

1. Check **Hard Deadlines** — anything due within ~90 days that isn't
   already in an active release block?
2. Check **Recurring Work** for any row with `Next due` on or before today.
3. Check **Triggered Work** for any fired trigger.
4. List **open GitHub issues** (`gh issue list --state open ...`) —
   unconditional, in every answer; presentation rules in the
   `gh-issues` skill.
5. Otherwise: work the first unchecked item in the lowest-numbered
   **Release** block. If all release blocks are done, this repo is
   idle by design — confirm against the cross-repo index before
   starting anything in **Not Next**.
```

The second half of step 5 is why the rule exists. The skill spells it out:

> If all release blocks are complete, [the repo] is **idle by design** — check the cross-repo index before proposing anything from **Not Next**, and say so explicitly rather than inventing work.

"Nothing" is a valid answer, and the rule has to say so in writing, because a model asked for a recommendation will always give one.

The **Not Next** table keeps deferred work deferred. Every row says what would reopen it:

| Work | Reopen when |
|---|---|
| Modernization track (language/DI/toolchain majors) | Feature investment reopens, **or** a required library forces it (none currently does). Order inside the track is fixed by hard dependencies. |
| Map-rendering rewrite | The packaging-compliance check fails (→ jumps into the active release), the vendor SDK breaks or EOLs, or feature investment reopens |
| Localization Phase 2 (native UI chrome) | The platform decision landed and the strings are implemented in open PRs held on a review decision; reopen anything further only after Release 1 ships |

A matching rule: **never propose Not Next work as next; its reopen condition has to fire first.** Without it, "not now" turns into "not now, unless the agent feels like it."

Triggered and recurring work get their own tables:

```markdown
## Triggered Work

| Trigger | Action |
|---|---|
| The API ships a new upstream provider | Add it via the integration playbook — small, mechanical, visible product win |
| The packaging-compliance check fails | Escalate the rendering migration into the active release |
| 30 days after Release 1 | Re-triage surviving crash groups; write new plans only if a signature dominates |

## Recurring Work

| Workstream | Cadence | Next due |
|---|---|---|
| Platform compliance check | Annually, every June (deadlines land Aug 31) | 2027-06-01 |
| Crash triage | After each release reaches ~50% rollout; otherwise quarterly | After Release 1 ships |
| Plans index review | After any release or material re-rank | After Release 1 ships |
```

Both tables answer a question a todo list can't: *is this actionable today, and if not, what exactly would make it actionable?* When recurring work is done, the same change updates its `Next due` date, so the schedule stays in step with the work.

The current answer sits at the top of the index in a **What's Next** section. The skill says to recompute it only when *a checkpoint arrived, a trigger fired, an issue was added or closed, or the user asks for fresh eyes.* Running the full check on every question spends context to get the same answer.

## Playbooks: The One Exception to Deletion

One document in the Android `plans/` directory is permanent, and it says so:

> This playbook stays in `plans/` permanently as a living checklist (unlike one-shot plans, it is reused per source, not removed).

It's the playbook for adding a new upstream data provider to the app. Our server normalizes every provider to one schema, so the client-side work is the same every time. The playbook has:

- **What the server promises**, stated once: what the client sends, what the server normalizes, and the standing rule that no per-provider parsing exists on the client *or should be added*.
- **Which files to watch in the server repo**, since that's where a change starts.
- **A table of every local file that changes**, with rough line numbers. There are five, and the table is followed by *"That's the entire surface"* and a list of things that don't need changes because they read one stored value.
- **A naming convention and the reason for it**, including old patterns not to copy and retired identifiers not to reuse.
- **A copy-paste checklist** for the implementation PR.
- **A section on removing or renaming a provider**, because it's the same steps in reverse.

The test is: *will this exact procedure run again, verbatim, for the next instance?* If yes, it's a playbook and it stays. If something like it will run again, the workflow carries over but the steps don't, and that's a skill. If it runs once, it's a plan, and we delete it on merge. A playbook isn't a plan we decided to keep. Writing a plan and then calling it permanent is how the directory got to 135 files.

## Lessons Learned

- **Migrate before you delete, and then delete.** If you only migrate, the plans directory keeps growing. If you only delete, you lose what the plan knew.
- **Have a second agent argue for keeping each plan you remove, and check links by script.** The second agent catches judgment errors. `grep` catches orphans and broken links. You need both.
- **Put state next to the procedure that changes it.** Two documents that someone has to remember to keep in sync won't stay in sync.
- **Allow the answer "nothing."** An agent asked what's next will always name something. Write down that idle is a valid answer, and give every deferred item a reopen condition, or "later" becomes "now."
- **Decide what a document is when you write it: one-shot, recurring, or workflow.** That's a plan, a playbook, or a skill. A plans directory turns into an archive when nobody makes that call.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying skill files and plan indexes, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the 135-file plans directory instead of the earlier post, the prose around each quoted rule was cut to what the quote does not already say, a stale "Step 5" reference to a four-step rule was corrected, and Lessons Learned went from nine bullets to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The What's Next Rule is now five steps (an open-issues check was added the day after this post), so the quoted block, the step reference, and the re-derive rule were updated to the current wording; the Not Next localization row was updated to its current reopen condition; the anti-list count was corrected from six to seven; the index trim was corrected from "in half" to 380 to 240 lines; and four quotes (the Ground Truth change line, the Verify markers, the migrate-before-delete rule, the conflicts-with-patterns rule) were aligned to the source text.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 1, run after the pilot (#67) merged. The prose was rewritten from an ELI5 of the post. The scaffolding-and-jig metaphor after the lifecycle table, the "sentiment, not classification" closer, and the "adversarially verified" phrasing are gone; the audit is now described as one agent classifying and a second arguing the keep case. The playbook bullet that began "The contract" now begins "What the server promises", and its list of five example file types was trimmed to the category and the count. "Surface" stays in the h2 (which is held fixed) and in the quoted phrase "That's the entire surface". Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"
