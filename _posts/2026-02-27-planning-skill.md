---
layout: post
title: "Planning Skill: Living Documents, Not Project Management"
date: 2026-02-27 11:00:00 -0600
summary: "A lightweight approach to planning with Claude: create plans, implement them, remove them. No project management software required."
tags: [skills, planning, workflow, claude]
---

## The Problem

Work that spans more than one session needs its context written down somewhere. A todo list holds the "what" but not the "why" or the "how," and it goes stale. An issue tracker is built for multiple people over extended time, and its tickets sit in "Done" forever. Neither fits one person (or one person plus Claude) working on one thing until it is done. We wanted to write down what we're going to do, track progress, then clean up when finished. No ceremonies, no backlog grooming, no status updates.

This comes from people who used to work at Basecamp, makers of project management software. Sometimes the best tool is a markdown file.

## The Solution

A planning skill that treats plans as **living documents** in a `plans/` directory. Create them, implement them, then remove them. An index file (`plans/index.md`) is the navigation.

### Philosophy

Plans are NOT for:
- Historical documentation (use CLAUDE.md or skills)
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

A fully implemented plan gets deleted. Any lesson or pattern worth keeping goes to CLAUDE.md or a new skill first.

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

## Implementation Steps

### 1. [Step Name]
**Files to Create**: list
**Files to Modify**: list
**Code patterns**: brief example

### 2. [Next Step]
...

## Success Criteria
- [ ] Tests pass
- [ ] Feature works
- [ ] Deployed
```

### The Index File

`plans/index.md` lists every plan by status:

```markdown
# Plans

## In Progress
- **[API Rate Limiting](23-api-rate-limiting.md)** - Add rate limits to public API

## Pending
- **[Widget Redesign](24-widget-redesign.md)** - New widget UI for iOS 18

## Recently Completed
(None - implemented plans are removed)
```

Every plan creation or removal updates the index. This is non-negotiable.

### Naming Convention

Sequential numbers with descriptive names, and a letter suffix for sub-plans:
- `23-api-rate-limiting.md`
- `24-widget-redesign.md`
- `24a-widget-accessibility.md` (sub-plan of 24)

### Creating a Plan

The skill's pre-implementation checklist:

1. **Enable research modes** - ultrathink, research mode, web search
2. **Check existing plans** - `ls plans/`
3. **Check if functionality exists** - `grep -r "pattern" app/`
4. **Write plan with clear steps**
5. **Update index**

### Implementing a Plan

```bash
# 1. Get current
git checkout main && git pull origin main

# 2. Fresh branch
git checkout -b implement-api-rate-limiting

# 3. Read plan thoroughly
# 4. Show implementation summary to user before starting
# 5. Execute steps with TodoWrite tracking
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
2. **Extract critical rationale** - Add patterns to CLAUDE.md or skills
3. **Delete the plan file** - `rm plans/23-api-rate-limiting.md`
4. **Update the index** - Remove the entry
5. **Commit the removal**

```bash
rm plans/23-api-rate-limiting.md
# Edit plans/index.md to remove the entry
git add -A
git commit -m "Remove implemented plan: 23-api-rate-limiting

Plan implemented in PR #456.
Rate limiting patterns preserved in .claude/skills/api/SKILL.md"
```

The commit message names the PR and the file that now holds the patterns, so the deleted plan leaves a trail.

## Why This Works

Keeping implemented plans creates noise. Future you doesn't need to know how the rate limiting was implemented. They need to know how it works now, and that belongs in documentation or skills. The hardest part is deleting, because we're trained to keep everything. But old plans get stale, reading them wastes time, and anything that mattered was extracted first. Plans are scaffolding: essential during construction, removed when the building is done.

## Critical Rules

### Never
- Create plans for past decisions (use CLAUDE.md)
- Skip research before planning
- Implement without syncing from main
- Skip user approval before starting
- Keep implemented plans around
- Forget to update the index

### Always
- Use research modes for plan creation
- Verify plan is current before implementing
- Update index when creating/removing plans
- Preserve critical rationale before deletion
- Create fresh branches from main

## Real Example

A plan that was created, implemented, and removed:

**Before implementation:**
```markdown
# API Rate Limiting

## Status
📝 **Planning**

## Implementation Steps
1. Add redis-based rate limiter
2. Configure per-endpoint limits in YAML
3. Add rate limit headers to responses
4. Add dashboard for viewing limits
5. Deploy behind feature flag

## Success Criteria
- [ ] Tests pass
- [ ] Rate limits enforced in production
- [ ] Dashboard accessible to admins
```

**After implementation:**
- Plan deleted
- Patterns added to `.claude/skills/api/SKILL.md`
- Index updated to remove entry
- Feature working in production

The plan served its purpose and was removed.

## Lessons Learned

- **Keep the document that says how it works now; delete the one that says how it was built.** The second goes stale the moment the first exists.
- **Check for the thing before planning the thing.** A grep and a look at existing plans are cheaper than a plan for work already done.
- **A plan is written against a moving main.** Sync and reread it before implementing.
- **Make the index update part of creating and removing, not a separate chore.** An optional index stops being navigation.

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Source: `.claude/skills/planning/SKILL.md`

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on work that spans sessions instead of on project management tools in general, the todo-list and issue-tracker comparisons moved from Why This Works into The Problem, and Lessons Learned replaces the bullets that repeated the title and the Critical Rules with rules that transfer. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
