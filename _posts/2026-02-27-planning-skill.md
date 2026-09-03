---
layout: post
title: "Planning Skill: Living Documents, Not Project Management"
date: 2026-02-27 11:00:00 -0600
summary: "A lightweight approach to planning with Claude: create plans, implement them, remove them. No project management software required."
tags: [skills, planning, workflow, claude]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Work that spans more than one session needs its context written down somewhere. A todo list holds the "what" but not the "why" or the "how," and it goes stale. An issue tracker is built for multiple people over extended time, and its tickets sit in "Done" forever. Neither fits one person (or one person plus Claude) working on one thing until it is done. We wanted to write down what we're going to do, track progress, then clean up when finished. No ceremonies, no backlog grooming, no status updates.

This comes from people who used to work at Basecamp, makers of project management software. Sometimes the best tool is a markdown file.

## The Solution

A planning skill that treats plans as **living documents** in a `plans/` directory. Create them, implement them, then remove them. An index file is the navigation: `plans/index.md` when this was written, a root `PLANS.md` since July 2026.

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

A fully implemented plan gets deleted. Any lesson or pattern worth keeping goes to a skill, README, or AGENTS.md first. The skill now spells out the only three reasons to keep a plan: unstarted future work, dormant work with a live reopen trigger, or an active decision with pending actions. A plan that calls itself a "record" or "reference" is a signal to remove it, not to keep it.

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

The Ground Truth section and the complete code came in June 2026, when plans were raised to a handoff-ready standard: written so a less capable model can implement them without exploring the codebase, with verified file paths, full code instead of sketches, and `Verify:` markers on any platform or API fact the author has not confirmed.

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

Every plan creation or removal updates the index. This is non-negotiable. Entries stay at one to three lines plus a link, and a plan file no index entry points to is a bug.

### Naming Convention

The skill allows sequential numbers with a letter suffix for sub-plans (`26`, `26a`, `26b`), but in practice every plan in the repos is a descriptive kebab-case name:
- `astronomy-caching.md`
- `alert-severity.md`
- `precipitation-refactoring.md`

### Creating a Plan

The skill's checklist:

1. **Check if functionality exists** - `grep -r "pattern" app/`
2. **Check existing plans** - `ls plans/`
3. **Write the plan to the handoff-ready standard** - verified paths, complete code, `Verify:` markers
4. **Update index**

When this post was written the first item was "Enable research modes: ultrathink, research mode, web search." That line was cut in July 2026 when the skills were trimmed to verified content; the rule that survives is "Skip research" under Never, now worded "Search the codebase, existing plans, and the web first."


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

Nothing is executed until the user has seen the summary and agreed:

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

The commit message names the PR and the file that now holds the patterns, so the deleted plan leaves a trail. The inbound-links step was added in July 2026 after a sweep found 15 links that had rotted across earlier removals. Skills never link into `plans/` for the same reason: the link dies when the plan does, so a skill cites the PR number and carries what it needs itself.

## Why This Works

Keeping implemented plans creates noise. Future you doesn't need to know how the rate limiting was implemented. They need to know how it works now, and that belongs in documentation or skills. The hardest part is deleting, because we're trained to keep everything. But old plans get stale, reading them wastes time, and anything that mattered was extracted first. Plans are scaffolding: essential during construction, removed when the building is done.

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

A plan from the iOS repo that was created, implemented, and removed. The on-device AI summaries feature had a wrap-up plan covering the last mile: a loading-state glow, a settings screen, and two future phases. Abridged:

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

The plan served its purpose and was removed.

## Lessons Learned

- **Keep the document that says how it works now; delete the one that says how it was built.** The second goes stale the moment the first exists.
- **Check for the thing before planning the thing.** A grep and a look at existing plans are cheaper than a plan for work already done.
- **A plan is written against a moving main.** Sync and reread it before implementing.
- **Make the index update part of creating and removing, not a separate chore.** An optional index stops being navigation.
