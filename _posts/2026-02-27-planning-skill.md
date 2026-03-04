---
layout: post
title: "Planning Skill: Living Documents, Not Project Management"
date: 2026-02-27 11:00:00 -0600
summary: "A lightweight approach to planning with Claude: create plans, implement them, remove them. No project management software required."
tags: [skills, planning, workflow, claude]
---

## The Problem

Project management tools are designed for teams and timelines. But sometimes you just need to plan something, implement it, and move on. Todo lists get stale. Kanban boards accumulate cruft. Tickets sit in "Done" forever.

We wanted something simpler: a way to write down what we're going to do, track progress, then clean up when we're finished. No ceremonies, no backlog grooming, no status updates.

(Ironic note: this comes from people who used to work at Basecamp, makers of project management software. Sometimes the best tool is a markdown file.)

## The Solution

A planning skill that treats plans as **living documents** in a `plans/` directory. Create them, implement them, then remove them. The index file (`plans/index.md`) serves as navigation.

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

When a plan is fully implemented, it gets deleted. If there are important lessons or patterns, extract them to CLAUDE.md or create a skill before deleting.

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

`plans/index.md` is the navigation hub:

```markdown
# Plans

## In Progress
- **[API Rate Limiting](23-api-rate-limiting.md)** - Add rate limits to public API

## Pending
- **[Widget Redesign](24-widget-redesign.md)** - New widget UI for iOS 18

## Recently Completed
(None - implemented plans are removed)
```

Every plan creation or removal must update the index. This is non-negotiable.

### Naming Convention

Plans use sequential numbering with descriptive names:
- `23-api-rate-limiting.md`
- `24-widget-redesign.md`
- `24a-widget-accessibility.md` (sub-plan of 24)

### Creating a Plan

The skill includes pre-implementation checklist:

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

Before starting, present:

```markdown
## Implementation Plan

**Files to Create**: 3 files
**Files to Modify**: 5 files
**Migrations**: 1 change
**Estimated Steps**: 8 steps

Ready? [Yes/No]
```

### Removing Completed Plans

When a plan is fully implemented:

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

## Why This Works

### Versus Todo Lists

Todo lists are good for quick tasks. Plans are good for multi-step work that spans sessions. They capture the "why" and "how," not just the "what."

### Versus Kanban/Issue Trackers

Issue trackers are designed for multiple people over extended time. Plans are designed for one person (or one person + Claude) working on one thing until done.

### Versus Keeping Everything

Keeping implemented plans creates noise. Future you doesn't need to know how the rate limiting was implemented - they need to know how rate limiting works NOW. That goes in documentation or skills, not old plans.

### The Cleanup Discipline

The hardest part is deleting. We're trained to keep everything. But:
- Old plans get stale
- Reading outdated plans wastes time
- If something matters, extract it first

Think of plans like scaffolding: essential during construction, removed when the building is done.

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

Here's a real plan that was created, implemented, and removed:

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

- **Living documents, not archives** - Delete when done
- **Index is the hub** - Keep it current
- **Research before planning** - Bad plans from bad research
- **Extract before deleting** - Don't lose critical patterns
- **Fresh branches always** - Plans assume current main

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Source: `.claude/skills/planning/SKILL.md`
