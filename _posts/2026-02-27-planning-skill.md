---
layout: post
title: "Planning Skill: Living Documents, Not Project Management"
date: 2026-02-27 11:00:00 -0600
summary: "A light way to plan work with Claude: write a plan as a markdown file, implement it, then delete it. No project management software needed."
tags: [skills, planning, workflow, claude]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

When a piece of work takes more than one session, we need to write down where we are and why, or the next session starts over. A todo list holds the "what" but not the "why" or the "how", and it goes stale. An issue tracker is built for a team working over months, and its finished tickets sit in "Done" forever. Neither fits one person, or one person plus Claude, working on one thing until it's done. We wanted to write down what we're going to do, track progress as we go, and clean up when we finish. No ceremonies, no backlog grooming, no status updates.

This approach comes from people who used to work at Basecamp, which makes project management software. For this job a markdown file is enough.

## The Solution

The planning skill keeps each plan as a markdown file in a `plans/` directory. A plan is a living document: we write it when the work starts, edit it while the work is in progress, and delete it when the work ships. An index file lists every plan. It was `plans/index.md` when this was written and has been a root `PLANS.md` since July 2026.

### Philosophy

Plans are NOT for:
- Historical documentation (use AGENTS.md or skills)
- Permanent reference (extract lessons first, then delete)
- Status reporting (no one reads those anyway)

Plans ARE for:
- Capturing implementation details before you forget
- Tracking progress during multi-step work
- Communicating context between sessions

### The Lifecycle

```
Create → Implement → Remove → (Preserve critical rationale elsewhere)
```

Once a plan is fully implemented, we delete it. Before that, any lesson or pattern worth keeping moves to a skill, a README, or AGENTS.md. The skill now lists the only three reasons to keep a plan around: the work hasn't started, the work is paused but we know what would restart it, or a decision is still open with actions pending. A plan that calls itself a "record" or a "reference" is a sign to remove it, not keep it.

## Implementation

### Plan Structure

```markdown
# [Feature Name]

## Overview
What this plan covers and why.

## Status
📝 **Planning** - Implementation pending

## Context
Problem being solved.

## Ground Truth
Key existing files/symbols (verified paths) the implementer needs.

## Implementation Steps

### 1. [Step Name]
**Files to Create**: list
**Files to Modify**: list
**Code**: complete example

### 2. [Next Step]
...

## Success Criteria
- [ ] Tests pass
- [ ] Feature works
- [ ] Deployed
```

The Ground Truth section and the complete code arrived in June 2026. That's when we started writing plans to what the skill calls a handoff-ready standard: a less capable model can implement the plan without exploring the codebase. In practice that means verified file paths, full code instead of sketches, and a `Verify:` marker on any platform or API fact the author hasn't confirmed.

### The Index File

The index lists every plan by section. A simplified example:

```markdown
# Plans

## In Progress
- **[API Rate Limiting](api-rate-limiting.md)** - Add rate limits to public API

## Pending
- **[Widget Redesign](widget-redesign.md)** - New widget UI for iOS 18

## Recently Completed
(None - implemented plans are removed)
```

Every time we create or remove a plan, we update the index in the same change. Each entry is one to three lines plus a link. A plan file with no index entry is a bug.

### Naming Convention

The skill allows numbered names, with a letter suffix for sub-plans (`26`, `26a`, `26b`). In practice every plan in the repos has a descriptive name, lowercase words joined by hyphens:
- `astronomy-caching.md`
- `alert-severity.md`
- `precipitation-refactoring.md`

### Creating a Plan

The skill's checklist:

1. **Check if functionality exists** - `grep -r "pattern" app/`
2. **Check existing plans** - `ls plans/`
3. **Write the plan to the handoff-ready standard** - verified paths, complete code, `Verify:` markers
4. **Update index**

When this post was first written, the first item was "Enable research modes: ultrathink, research mode, web search." That line was cut in July 2026, when the skills were trimmed down to content we had verified. What survives is the "Skip research" rule under Never, now worded "Search the codebase, existing plans, and the web first."

### Implementing a Plan

```bash
# 1. Get current
git checkout main && git pull origin main

# 2. Fresh branch
git checkout -b implement-api-rate-limiting

# 3. Read plan thoroughly
# 4. Show implementation summary to user before starting
# 5. Execute steps, tracking progress with your tool's todo list
# 6. Test after each major step
```

Nothing runs until the user has seen this summary and said yes:

```markdown
## Implementation Plan

**Files to Create**: 3 files
**Files to Modify**: 5 files
**Migrations**: 1 change
**Estimated Steps**: 8 steps

Ready? [Yes/No]
```

### Removing Completed Plans

When the success criteria are met:

1. **Verify success criteria** - Everything works
2. **Extract critical rationale** - Add patterns to a skill or AGENTS.md
3. **Fix inbound links** - `grep -rln "plan-name.md" plans/` and redirect each one to where the content went
4. **Delete the plan file** - `rm plans/plan-name.md`
5. **Update the index** - Remove the entry
6. **Commit the removal**

The skill's commit template:

```bash
rm plans/XX-plan-name.md
# Edit PLANS.md - remove entry
git add -A
git commit -m "Remove implemented plan: XX-plan-name

Plan implemented in PR #XXX.
Critical rationale preserved in [location]."
```

The commit message names the PR and the file that now holds the patterns, so anyone can trace where the deleted plan went. The inbound-links step was added in July 2026, after a sweep found 15 broken links left behind by earlier removals. Skills never link into `plans/` for the same reason. A link to a plan dies when the plan does, so a skill cites the PR number and carries what it needs itself.

## Why This Works

Implemented plans left in place are noise. Later on we won't need to know how the rate limiting was built. We'll need to know how it works now, and that belongs in the docs or a skill. The hard part is the deleting, because we're used to keeping everything. But an old plan goes stale, reading it wastes time, and anything that mattered was moved out before it was deleted.

## Critical Rules

As of September 2026, the web repo's list:

### Never
- Create plans for past decisions (use AGENTS.md or skills)
- Skip research (search the codebase, existing plans, and the web first)
- Implement without syncing (`git pull origin main` first)
- Skip user approval (show the implementation plan first)
- Keep implemented plans (remove and preserve rationale elsewhere)
- Forget PLANS.md (update every time you create or remove plans)
- Leave inbound links dangling (grep for the filename and redirect before removing)
- Keep a plan as a "record", "archive", or "reference library"
- Leave an orphan (every `plans/*.md` is linked from PLANS.md, or migrated and removed)

### Always
- Use research modes for plan creation
- Verify plan currency before implementing
- Update PLANS.md when creating/removing plans
- Track with your tool's todo/task list during implementation
- Preserve critical rationale before removing plans
- Create fresh branches from main

## Real Example

Here is a plan from the iOS repo that was created, implemented, and removed. The on-device AI summaries feature had a wrap-up plan for the last pieces: a glow while the summary loads, a settings screen, and two phases left for later. Abridged:

**Before implementation:**
```markdown
# AI Summaries Wrap-Up

## Status
📝 **Planning**

## Implementation Phases
1. Loading state polish (glow view modifier, respect Reduce Motion)
2. AI Summaries settings screen (toggle, availability status)
3. Voice/style customization (future)
4. Brevity control (future)

## Success Criteria
- [ ] Glow effect displays during summary generation
- [ ] AI Summaries toggle appears in Settings
- [ ] Toggle correctly enables/disables generation
```

**After implementation (February 2026):**
- Plan deleted, in a PR that removed six finished or obsolete plans at once
- Rationale preserved in `.claude/skills/summary-service/SKILL.md`
- Index entry marked complete (the skill now says to remove the entry outright)
- Feature shipped

## Lessons Learned

- **Keep the document that says how it works now, and delete the one that says how it was built.** The second goes stale as soon as the first exists.
- **Check whether it already exists before planning it.** A grep and a look at existing plans cost less than a plan for work that's already done.
- **Main keeps moving after a plan is written.** Sync and reread the plan before implementing it.
- **Update the index in the same change that creates or removes a plan.** An index that lags behind stops being trusted.
