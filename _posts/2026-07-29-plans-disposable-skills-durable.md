---
layout: post
title: "Plans Are Disposable, Skills Are Durable"
date: 2026-07-29 09:20:00 -0600
summary: "A lifecycle policy for agent-facing docs: plans written to a handoff-ready standard, deleted when the work ships, an index that ranks instead of narrates, and a what's-next rule that lets an agent answer 'nothing'."
tags: [ai-agents, planning, workflow, documentation]
---

## The Problem

One repo's `plans/` directory reached 135 files. Many of them said `Status: Implemented` or called themselves a "durable record", and those labels are exactly why nobody had deleted them.

We wrote about [our planning skill](/planning-skill/) in February: a markdown file in `plans/`, an index, and a rule to delete the plan when the work ships. Five months across three [Hello Weather](https://helloweather.com) repos showed that covered how plans are born, not how they live and die. Three more problems came with the pile-up:

1. **Docs rot.** A plan written in March cites `app/models/thing.rb:117`. By June that line is 145, the method was renamed, and the library version in the plan is two majors stale. An agent implementing from it will confidently build the wrong thing.
2. **Sessions end.** The agent that wrote the plan has all the context. The one that implements it three weeks later has none, and may be a smaller model. Anything left implicit is an exploration tax paid on every read.
3. **Agents always find something to do.** Ask "what's next?" and an agent will answer. If the honest answer is "nothing, wait for the trigger," an agent without a rule authorizing that answer invents work, usually a refactor, usually the most expensive option on the table.

The fix was not a better template. It was a lifecycle policy: what a plan must contain, when it must die, what survives it, and what the index is allowed to say.

## The Lifecycle

```
Research → Write (handoff-ready) → Rank in the index → Implement
    → Migrate durable knowledge into a skill → DELETE the plan → Update the index
```

Three document types make this work, and confusing them is the root of most doc rot:

| Type | Lifespan | What it holds |
|---|---|---|
| **Plan** | One-shot — deleted after it ships | This specific change: steps, verified paths, success criteria |
| **Playbook** | Permanent, versioned | A procedure that recurs verbatim per instance |
| **Skill** | Permanent, versioned | A workflow, plus the invariants and gotchas plans leave behind |

Plans are scaffolding. Skills are the building. Playbooks are the jig you keep in the shop because you'll cut this joint again.

## Ground Truth: Write for a Reader Who Won't Explore

Every plan now carries a `## Ground Truth` section. All three planning skills state the standard the same way:

> Write plans so a less capable model can implement them without exploring the codebase.

If the implementer has to grep to find where things register, the plan has failed. A smaller model may grep, guess wrong, and ship the guess. Four rules follow.

**1. Cite verified paths with line numbers.** A real section, verbatim except for the header:

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
> *Change required:* `truncate(3)` → `truncate(2)` on both lines.

The skill's rule is *never cite a symbol you haven't confirmed exists.* The header carries the verification date, so a stale citation is visible rather than merely wrong.

**2. Name the registration points.** The expensive part of an unfamiliar codebase is not the new file but the four places that register it. A pass across 32 plans in the web repo added key-files tables, the constant lists a new entry lands in, and the factory method that maps a name to a class. The same pass corrected patterns the plans had invented: a base class that didn't exist, an environment-variable lookup where the codebase used a helper. Plans written from memory hallucinate house conventions.

**3. Date every external fact with a `Verify:` marker.** Library versions, upstream payload shapes, platform policy deadlines, pricing: anything the codebase can't confirm gets flagged inline.

```markdown
| cloud_cover | `cloud_cover` (%) | Recorded; `Verify:` 0-100 integer encoding per the API spec |
| alert id | `Verify:` exact key name (`id` vs `alert_id`) | Verify |
```

The skill's principle is *never present external facts as timeless; date them and flag them for re-checking.* The implementation checklist resolves `Verify:` markers before building on them. A marker that survives to code review is a bug in the review, not the plan.

**4. Mark what's cuttable.** When a plan feeds a deadline-bound release, each item is labeled required or cuttable:

> **Effort honesty**: mark items cuttable vs required when they feed a release block, so time pressure cuts the right things.

Something gets dropped under time pressure. The question is whether the person dropping it has the author's judgment. One release block also ends with an anti-list, *"Explicitly NOT required for this release (common trap — these feel related but aren't)"*, followed by six tempting adjacent upgrades. Naming the trap is cheaper than re-litigating it every session.

One more rule keeps Ground Truth honest at implementation time:

> **If the plan conflicts with existing patterns** — follow the patterns, update the plan. The codebase outranks the plan when they disagree.

## Deletion: A "Record" Label Is a Delete Signal

The February post said "delete when done." The rule that survived contact with 135 files is sharper:

> A plan is **removed** once the work it describes is **fully implemented / shipped**, or it is a **completed one-time analysis** — *regardless of what the plan calls itself*. A "Status: Implemented", "implementation record", "durable record", or "reference" self-label is **not** a reason to keep a plan; it is a signal to remove it (after migrating anything durable).

Documents don't self-label as records because they're valuable. They do it because someone finished the work and felt bad deleting the file. The label is sentiment, not classification.

Only three things justify keeping a plan:

1. **Future / unstarted work** — not built yet.
2. **Triggered / dormant work** — reopens on a monitoring or external trigger, *and the plan still holds the live reopen playbook*.
3. **An active in-flight decision** — pending actions or dates.

Anything else is done: migrate, then remove. Being linked from the index is not a reason to keep; that's circular. Genuine uncertainty gets a deliberate asymmetry: *when uncertain, prefer keep and say why — deleting curated plans is the risky direction.*

### Migrate before you delete

> Durable knowledge a finished plan holds — non-obvious decisions, API facts and gotchas, operational procedures, hard-won rationale — must be **moved into the relevant skill** *before* the plan is removed. Never keep a plan alive just to hold reference material, and never delete without first rehoming what's worth keeping.

The web repo's pass removed 35 plans in one pull request and migrated their contents into a dozen skills. What moved outlives the change: negative results (three optimizations that didn't work, so nobody tries them again), upstream API quirks, capacity thresholds and their reopen gates, rejected vendor evaluations. What got deleted was the narrative of how we got there.

A multi-agent audit classified all 135 plans, then adversarially verified every removal candidate: a second agent argued the keep case against the first agent's delete recommendation. Migrations and inbound-link repointing fanned out one agent per file. Integrity was checked by script, not by eye:

```bash
# Orphans: a plan file the index doesn't point to is a bug
for f in plans/*.md; do
  b=$(basename "$f")
  grep -q "$b" PLANS.md || echo "ORPHAN: $b"
done
```

An earlier round of deletions had left 15 dangling links across surviving plans, so removal now has a mandatory step *before* `rm`:

```bash
# Find every plan linking to the one being removed, redirect them in the same commit
grep -rln "plan-name.md" plans/
```

## The Index Is a Ranking Surface, Not a Document Store

`PLANS.md` had grown to 380 lines of duplicated prose. The policy that cut it in half:

> PLANS.md is a navigation and ranking surface, not a document store. Duplicated prose goes stale the moment the underlying plan moves.
>
> - **Every entry is 1–3 lines + a link.** If you're writing more than that in the index, the extra belongs in the linked plan.
> - **No implemented-work tracking, dated pass logs, or history in the index** — record only the current state and what's remaining.
> - **Status updates replace, they don't accrete.** When state changes, rewrite the entry's current-state line; don't append another dated bullet under the old ones.
> - **Content with no plan doc to link to stays very concise.** If it needs more, create the plan doc and link it.

"Replace, don't accrete" is the rule that does the work. An agent updating a status file naturally appends. It feels safer, it preserves history, and it turns a 3-line entry into a 40-line changelog nobody reads. Dated history is fine *inside* an in-flight plan, because it dies with the plan.

The same restructuring moved per-item status boards next to the procedure they describe. A multi-step rollout used to tick progress in the central index. Now the board lives inside the runbook, and each rollout PR updates its own line in its own diff. Procedure and state sit in one document, updated by the change that alters them, instead of two documents kept in sync by discipline.

Every plan also needs a ranking slot, not just a catalog line:

> Every plan creation/removal MUST update `PLANS.md` — both the **Plan Catalog** entry AND its slot in the ranking machinery (a release-block item, a Triggered Work row, or a Not Next row with an explicit reopen condition). A plan that exists only in the catalog has no priority and will never be picked.

## The What's-Next State Machine

The Android index answers "what's next?" with a deterministic four-step check:

```markdown
### What's Next Rule

1. Check **Hard Deadlines** — anything due within ~90 days that isn't
   already in an active release block?
2. Check **Recurring Work** for any row with `Next due` on or before today.
3. Check **Triggered Work** for any fired trigger.
4. Otherwise: work the first unchecked item in the lowest-numbered
   **Release** block. If all release blocks are done, this repo is
   idle by design — confirm against the cross-repo index before
   starting anything in **Not Next**.
```

The second half of step 4 is the reason the rule exists. The skill spells out the obligation:

> If all release blocks are complete, [the repo] is **idle by design** — check the cross-repo index before proposing anything from **Not Next**, and say so explicitly rather than inventing work.

"Nothing" is a legitimate terminal state, and the system has to *authorize* it, because a helpful model asked for a recommendation will produce one.

The **Not Next** table keeps deferred work deferred. Every row carries a reopen condition:

| Work | Reopen when |
|---|---|
| Modernization track (language/DI/toolchain majors) | Feature investment reopens, **or** a required library forces it (none currently does). Order inside the track is fixed by hard dependencies. |
| Map-rendering rewrite | The packaging-compliance check fails (→ jumps into the active release), the vendor SDK breaks or EOLs, or feature investment reopens |
| Localization Phase 2 (native UI chrome) | Only when the platform decision lands *and* Phase 1 has shipped — highest throwaway of the three phases |

A matching hard rule: **never propose Not Next work as next; its reopen condition has to fire first.** Without it, "not now" degrades into "not now, unless the agent is feeling ambitious," which is not a policy.

Triggered and recurring work are first-class rows:

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

Both tables answer what a stale todo list can't: *is this actionable today, and if not, what exactly would make it actionable?* Completing recurring work updates its `Next due` in the same change, so the schedule can't drift from the work.

The cached answer lives at the top of the index as a **What's Next** section. The skill says to re-derive it only when *a checkpoint arrived, a trigger fired, or the user asks for fresh eyes.* Re-running full triage on every question burns context to arrive at the same answer.

## Playbooks: The One Exception to Deletion

One document in the Android `plans/` directory is permanent, and it says why:

> This playbook stays in `plans/` permanently as a living checklist (unlike one-shot plans, it is reused per source, not removed).

It is an integration playbook for adding a normalized upstream provider. A server-side API normalizes every provider to one schema, so the client work is identical every time. Its parts:

- **The contract**, stated once: what the client sends, what's normalized server-side, and the standing rule that no per-provider parsing exists on the client *or should be added*.
- **Upstream source-of-truth files to watch** in the other repo, where the change originates.
- **A table of every local file that changes**, with approximate line anchors: a constant, a display string, an asset, a settings row, a mapping entry. Five files, then *"That's the entire surface"*, then the things that need no changes because they key off a single stored value.
- **A naming convention with rationale**, including historical patterns not to imitate and retired identifiers never to reuse.
- **A copy-paste checklist** for the implementation PR.
- **A removal/rename section**, the inverse operation, because it's the same knowledge.

The test: *will this exact procedure run again, verbatim, for the next instance?* If yes, it survives the work. If "something like it," the workflow generalizes but the steps don't, and that's a skill. If it runs once, it's a plan, and it dies on merge. A playbook is not a plan with a longer life; writing a plan and then declaring it permanent is how you get the 135-file directory.

## Lessons Learned

- **Migrate before you delete, and delete anyway.** The two halves are one rule. Migration without deletion is hoarding; deletion without migration is data loss.
- **Verify removals adversarially, verify integrity by script.** A second agent arguing the keep case catches judgment errors. `grep` catches orphans and dangling links. Neither substitutes for the other.
- **Put state next to the procedure that changes it.** Two documents kept in sync by discipline will not stay in sync.
- **Design for the answer "nothing."** An agent asked what's next will always produce something. Make idle a legitimate terminal state and gate every deferred item behind a reopen condition, or "later" quietly becomes "now."
- **Classify at write time: one-shot, recurring, or workflow.** Plan, playbook, or skill. Getting that call right is what keeps a plans directory from becoming an archive.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying skill files and plan indexes, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the 135-file plans directory instead of the earlier post, the prose around each quoted rule was cut to what the quote does not already say, a stale "Step 5" reference to a four-step rule was corrected, and Lessons Learned went from nine bullets to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
