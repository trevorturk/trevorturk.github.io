---
layout: post
title: "Moving the Queue Out of Git"
date: 2026-09-03 16:20:00 -0600
summary: "For six months our plans lived as markdown in git, and half of all commits touched them. We moved the queue to one GitHub Project across three repos, kept in-flight specs and dated reference docs in git, and made bin/next the only answer to 'what's next'. The planning skill shrank to policy, the script does the reading, and the first design got simplified on day two."
tags: [ai-agents, planning, workflow, github, skills]
model: "Claude Fable 5.1"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Since March, half of the commits in our web and iOS repositories touched planning files. Most of those commits changed nothing but planning text. A status flip, a re-rank, or a new "next due" date was a branch, a PR, a review, and a merge, because the queue lived in git.

| Repo | Commits, March to September 1 | Touching `plans/` or `PLANS.md` |
|---|---|---|
| web | 627 | 313 |
| iOS | 652 | 353 |

[Hello Weather](https://helloweather.com) is built across three repositories by coding agents. Two earlier posts describe the markdown system those agents used: [Planning Skill: Living Documents, Not Project Management](/planning-skill/) and [Plans Are Disposable, Skills Are Durable](/plans-disposable-skills-durable/). An index file ranked the work. A `plans/` folder held one document per piece of work, and the PR that shipped a plan deleted it. It was a good memory for agents. It was a poor place for a human to look, and the owner is about to have much less time to look. That made the change urgent. The board had to become the only planning page a person opens, and everything off it had to be invisible to agents by default.

## What the Markdown System Got Right

Some of it was worth keeping, and we kept it.

A plan written to the handoff standard is a spec an agent can execute cold. Overview, Context, Ground Truth with verified file paths, Implementation Steps, Tests, Success Criteria. Nothing in an issue tracker holds that as well as a file next to the code it describes.

Deleting the plan in the PR that ships it kept the folder honest. A plan that outlived its work was a bug, and we had a rule and a monthly sweep for it.

Reference material that never changes state, like a cost model or a merge runbook, was fine as a dated file. The trouble was only that it shared a folder and an index with the queue.

And git gave us diffs. When an agent rewrote a plan's ground truth, the review saw exactly what changed. That is the one thing we gave up.

## Where It Broke

The index was supposed to be pointers. By the end it was 257 lines under 22 headings: a current pick, a force-ranked table, a triggered-work table, a recurring-work table, and ten "reference groups" of links. The `plans/` folder in web held 95 files and close to 2 MB, with the largest at 80 KB. The planning skill that taught agents to read all of that had grown to 298 lines.

Dates hid in prose. We had a rule that every dated action had to be a row in the index, and a sweep for dates that only existed inside a plan. Both were needed because the format had no date field.

There was no way to assign anything. The designer needed to see the work waiting on them without reading git, and "assigned to" was a word inside a paragraph.

Cross-repo links rotted. The web index linked to `../ios/plans/*.md` and `../android/plans/*.md`. Every rename on one side broke a link on another.

Issues had been demoted to a scratch buffer. The old rule was that the steady state is zero open issues across the org, and an open issue was a capture waiting to be drained into a plan. So the tool GitHub already gives you for tracking work was the one thing we were not tracking work in.

## The Move

The rule after the move is one fact, one home:

| Fact | Home |
|---|---|
| Priority, ownership, waits, and dates | one org Project across the three repos |
| Problem, evidence, and decisions | a GitHub issue |
| In-flight implementation spec | `plans/<name>.md` |
| Permanent research or runbook | a dated file in `docs/` |
| Durable rule | `AGENTS.md` or a skill |

Every plan was pre-sorted in comments on one tracking issue, and the owner approved the sort before anything was converted. Then, in web: 22 files moved to a new `docs/` folder with the status sections removed and a provenance note added. Two in-flight specs stayed in `plans/`. The other 71 became issues and were deleted. Git history is the archive. Each converted issue links a permalink to the file at the last commit before the migration, so `git show <sha>:plans/<file>.md` still retrieves the full document.

We chose not to paste plan bodies into issues. An issue body caps at 64 KB, has no diff history, and stale ground truth in an issue reads as current. Instead each converted issue follows a short template:

```markdown
Standing: owner-requested, decided | agent-suggested, unconfirmed | speculative

**What:** one paragraph.
**Why / evidence:** one or two lines.
**Decisions already made:** bullets, if any.
**Links:** related issues, PRs, docs, or a live spec.
**Reference PR:** #N, if banked.
Snapshot of `plans/<file>.md` at <sha> — <permalink>. Ground truth not re-verified since; re-verify before building.
```

The migration day created 246 issues across the three repositories. The web PR was +487 and −33,494 lines; iOS was +521 and −49,952; Android was +145 and −4,388. The web index went from 257 lines to under 50: a ranking thesis, a pointer to the board, and a catalog of the two specs and the reference docs. No queue, no status, no dates.

The "banked PR" idea from [The Warehouse of Closed PRs](/the-warehouse-of-closed-prs/) survived. An issue for parked work carries a `Reference PR:` line, and a monthly job runs `git merge-tree` of each reference PR's head against main and toggles a `reference:conflicts` label. On the first dry run all eight of the iOS reference PRs already conflicted, which is the expected state. The value is the shape of the change, not mergeability.

## The Skill Points, the Script Reads

The neat part is how this combined with the pattern from [Skills and Scripts](/skills-and-scripts/). That post paired a short skill that teaches the model a workflow with a `bin/` script that does the work. The old planning skill was 298 lines because it had to teach an agent a procedure. Read the index. Check the recurring table for due rows and the triggered table for fired triggers. Walk the ranked table, then update the current pick. Now the procedure is a script and the skill is policy.

`bin/next` in the web repo makes one `gh project item-list` call for the board. Then it runs one GraphQL query per repository for open issues with their parent, assignees, and linked PRs, and one `gh pr list --assignee @me` per repository. Then it prints, in order: assigned open PRs, Waiting and Recurring rows due today or earlier, Active work, and the ranked Main queue. The skill says to answer "what's next" in that order and never to reconstruct the queue from prose.

The grouping logic is the part with rules in it, trimmed here to the decisions:

```ruby
def group(items, issues, pull_requests = [], all:)
  today = Date.today
  by_key = items.to_h { |i| [i[:key], i] }
  by_issue = issues.to_h { |i| [i[:key], i] }
  # join board items to live issues; drop items whose issue closed
  items = items.select { |i| i[:open] }

  queue     = items.select { |i| i[:status] == "Queue" }
  waiting   = items.select { |i| i[:status] == "Waiting" }
  recurring = items.select { |i| i[:status] == "Recurring" }
  fired         = waiting.select { |i| due?(i, today) }.sort_by { |i| i[:next_due] }
  recurring_due = recurring.select { |i| due?(i, today) }.sort_by { |i| i[:next_due] }

  # a top-level initiative is active when it, or any descendant,
  # is assigned or has an open PR
  roots = tree_roots(queue)
  active_roots, main_roots = roots.partition { |root| active_in_subtree?(root, queue) }
  active = all ? trees_for(queue, active_roots) : active_paths(queue)
  main_all = trees_for(queue, main_roots)
  main = all ? main_all : limit_trees(main_all, MAIN_LIMIT)

  warnings = []
  warnings += (waiting + recurring).reject { |i| i[:next_due] }.map do |i|
    "#{i[:status]} item #{i[:repo]}##{i[:number]} has no Next due date"
  end
  unplaced = issues.reject { |issue| by_key.key?(issue[:key]) || descendant_of_project_item?(issue, by_issue, by_key) }
  warnings += unplaced.group_by { |i| i[:repo] }.map { |repo, list| "Unplaced open issues in #{repo}: #{list.size}" }

  { "warnings" => warnings, "assigned_pull_requests" => pull_requests,
    "active" => active, "main" => main,
    "fired_triggers" => fired, "recurring_due" => recurring_due }
end
```

Notice that "active" is derived, not stored. There is no In Progress status. An initiative is active when someone is assigned to it or to any sub-issue under it, or when a sub-issue has an open PR. Assignment is the only signal, so the board can't disagree with itself about what is in flight. The two warnings are the two rules we used to enforce by sweep. One is a wait with no date. The other is an issue that is neither on the board nor under something that is.

The script is a module with `exit NextCli.run if $PROGRAM_NAME == __FILE__` at the bottom. A Minitest file can `load` it and test the grouping with hand-built hashes and no network. Twelve tests cover active derivation, hierarchy nesting, the rank cutoff, and the warnings.

The board is cross-repo, so there is one implementation. iOS and Android each ship a wrapper that finds the web checkout beside the repo's main checkout and hands off:

```bash
#!/usr/bin/env bash
# "What's next?" — delegates to the web repo's bin/next (one board, three repos).
set -euo pipefail
main="$(cd "$(git -C "$(dirname "$0")" rev-parse --path-format=absolute --git-common-dir)/.." && pwd)"
web="${HW_WEB:-$(cd "$main/.." && pwd)/web}"
[ -x "$web/bin/next" ] || { echo "bin/next: no web checkout at $web — set HW_WEB." >&2; exit 1; }
exec "$web/bin/next" "$@"
```

The `git-common-dir` call matters because agents work in worktrees, and a worktree's `.git` is a file pointing at the main checkout. Resolving through it means the wrapper works from any worktree.

The permissions follow the earlier post's rule too. The web and Android settings allowlist `bin/next`, `bin/plans-reconcile`, and the read-only `gh project` commands. Board edits, parenting, and closes still prompt.

## Guardrails

Three small jobs replace the monthly sweeps we used to run by hand.

`bin/plans-reconcile` runs in CI on every web PR. Every file in `plans/` must link an open issue that is on the board, with an `**Issue:** #N` line near the top. Every actively owned top-level Queue issue must have a spec or say "spec pending" in its body. A spec held ahead of active ownership only warns. This is the old "close the plan in the PR that ships it" rule, enforced by a build instead of a reviewer.

The monthly rot check is described above.

A Monday workflow runs `bin/next --all` and posts the output as a comment on a pinned "Planning digest (weekly)" issue. Subscribing to that issue delivers the digest by email, with no Slack bot to build. One thing to know: the default Actions token can't read an org Project or touch the other two repositories. The digest needs a fine-grained token with Projects read on the org and Issues read and write on each repo.

## Day Two: One Queue Instead of Five Buckets

The first design did not survive its first day of use, and the correction is worth recording.

The original board had a custom Bucket field with Now, Next, Later, Waiting, and Design, on top of GitHub's built-in Status field. Rank restarted inside each bucket. A `backlog` label marked issues that were deliberately off the board. A `needs-design` label marked design work, which was also a bucket, which was also a view. There was a work-in-progress cap of two Now items per repository, and a Now item had to have an open PR.

By the next morning the contradictions were visible. Design was both a bucket and a filtered view. Status was present but ignored. Deferred work still had assignees from the import, which broke the ownership rule. And 169 imported backlog issues sat off the board where nothing would ever list them.

The second design, shipped the same afternoon, has one Status field with Queue, Waiting, Recurring, and Done. Main is the one force-ranked list: every top-level Queue initiative has a unique global rank, and sub-issues inherit their parent's position. Design is a view of Queue items assigned to the designer, not a state. There is no backlog lane and no persistent inbox. Every open issue is either a Project item or a real sub-issue of one. Work that is not credible enough to parent or rank is closed as not planned.

The WIP cap went too. We normally carry one top-level initiative at a time, but the script now shows overlap under Active without a numeric warning. A cap the tool enforces gets gamed by reassigning; a list that shows the overlap gets read.

Migrating live meant two steps. The script first learned to read both models, mapping the old buckets onto Queue, so the board could be reconfigured while `bin/next` kept working. Then a follow-up PR removed the compatibility code, deleted the migration spec, and closed its issue with `Fixes`. The spec for the planning system was itself the first spec to go through the new lifecycle.

## Results

- Web's planning index dropped from 257 lines to under 50. The planning skill went from 298 lines to 104 and the issues skill from 113 to 85, because the reading procedure moved into a script with tests.
- The board holds 152 items today: 116 in Queue, 12 Waiting, 14 Recurring, and 10 Done. `bin/next --all` runs clean with no warnings after the hierarchy audit.
- Two guardrails that were monthly manual sweeps, orphaned specs and undated waits, are now a CI job and a warning line.
- We gave up diff history on plan content that became issues. Comments are the decision trail, and the permalink recovers the old document. The acceptance test is set for one release cycle out: the share of commits touching planning files should fall from about half to under 20 percent. That has not been measured yet.

## Lessons Learned

- If changing a status costs a PR, count how many of your commits are status changes. Half is a sign the state is in the wrong place.
- Keep the spec next to the code and the queue in the tracker. A file is the right home for content an agent executes; a field is the right home for a date or a rank.
- Derive "in progress" from assignment and open PRs instead of storing it. One signal can't contradict itself.
- Convert a procedure the skill teaches into a script the skill points at, and put the rules in tests. The skill gets shorter and the rules get checked.
- Expect to redesign the board after a day of real use. Ship the first version with the tool able to read both models so the live switch is safe.
