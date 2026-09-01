---
name: post-ideas
description: Research the helloweather repos for work done since the last blog post and propose candidate posts. Use when asked "what should we blog about?" or "find blog post ideas".
---

# Post Ideas

Research recent work across the Hello Weather repos and produce a proposed list of blog
posts for the Mechanical Turk blog. This is research + proposal only — writing a chosen
post is the `blog-post-generator` skill's job.

## When to Use

- "It's been a while since we added blog posts — what could we write about?"
- "Research helloweather and propose posts"
- "Find blog post ideas since the last post"

## Workflow

1. **Find the cutoff date**: the most recent file in `_posts/` (`ls _posts | tail -1`).
   Also collect the full list of existing post topics so agents avoid re-proposing them.

2. **Dispatch one research agent per repo** (in parallel, read-only/Explore agents,
   Opus-class model is a good fit). Repos: `~/Code/helloweather/web`, `~/Code/helloweather/ios`,
   `~/Code/helloweather/android`. Each agent should:
   - Run `git log --since=<cutoff> --oneline` (with and without `--merges`) for the landscape,
     plus `--stat` on the bigger changes.
   - Check `.claude/skills/`, `plans/`/`PLANS.md`, and docs for new/updated material.
   - Read enough code/diffs on the 5-10 most interesting themes to understand what was
     built and why.
   - Return candidates with: working title, 2-3 sentence description, evidence
     (SHAs/PRs/paths/dates), what prompted the work (a constraint, a gap, a cost, a
     design question, or a failure), why it's reusable, and any proprietary-info risks
     to sanitize.
   - Also return a "considered and rejected" list with one-line reasons.

3. **Filter and rank** the combined candidates:
   - Drop anything already covered by an existing post (unless there's a major new angle).
   - Drop anything that can't be sanitized (full data-source lists, vendor pricing,
     business specifics, security-sensitive operational detail).
   - Prefer cross-cutting patterns (server+client stories), AI-agent workflow learnings,
     and "hard-won lesson" narratives over routine feature work. A candidate with a
     concrete prompt for the work (a constraint, a gap, a cost, a question, a failure)
     beats one that only exists as "we built X"; see `blog-post-generator`.
   - Merge overlapping candidates from different repos into one cross-platform post idea.

4. **Present the proposal**: a ranked list with working title, one-paragraph pitch,
   evidence pointers, and sanitization notes. Do NOT write posts — wait for the user to
   pick, then use `blog-post-generator`.

## Important Notes

- Nothing proprietary in proposals or posts: patterns yes, secrets/specifics no.
- Big-picture architecture, workflows, performance stories, and agent practices are the
  sweet spot — see existing posts in `_posts/` for calibration.
- A good batch is 5-10 strong candidates; quality over exhaustiveness.
