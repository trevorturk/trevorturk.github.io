---
layout: post
title: "Just Run bin/next"
date: 2026-09-17 13:55:00 -0500
summary: "I already decided what matters. When it's time to work, bin/next saves me from deciding all over again."
tags: [workflow, ai-agents, planning]
model: "GPT-6"
last_edited: 2026-09-17
last_edited_by: "GPT-6"
---

I've been really happy with a little command called `bin/next`. We added it when we moved our work queue into GitHub two weeks ago, and it's made getting started easier for me.

I run it, and it shows me what needs attention. I can ask my coding agent to run it and get the same list. There's no need to start with a conversation about what we could work on.

Here's the command, from any of the [Hello Weather](https://helloweather.com) repos:

```sh
bin/next
```

It reads our GitHub project board, issues, and pull requests, then prints a list. It doesn't change anything. I still have to choose a task and do the work, but I have somewhere to start.

## Decide Priority Before the Session Starts

We keep our tasks in order on a project board. Putting a task near the top means deciding that it's more important than the tasks below it. That takes some thought, and I'd like to use that decision when it's time to work.

Without a clear starting point, asking an agent “what should we do next?” can turn into another planning session. It suggests things. I consider them. We could spend our time discussing priorities we'd already settled.

With `bin/next`, I can just ask it to read the list. If our priorities have changed, we update the board. If they haven't, we can get on with the work.

Previously, we had [a written procedure](/plans-disposable-skills-durable/#the-whats-next-state-machine) for figuring this out. It meant reading planning files, checking dates, and looking through issues. The command does that checking now. I've already [written about the migration](/moving-the-queue-out-of-git/) and how the script works, but the part I appreciate most is having less to figure out before I begin.

## Read From the Top

The list starts with work that needs attention before we take on something new:

- **PRs needing your attention:** proposed code changes waiting for my review or assigned to me.
- **Waiting due:** something we were waiting on that it's time to check again.
- **Recurring due:** a regular task that's due, such as checking documentation links.
- **Active:** work someone has already picked up.
- **Main:** the remaining tasks, in priority order.

Here's a shortened example with made-up tasks and numbers:

```text
## PRs needing your attention
  web#41  Clarify the empty-state message  [review requested]

## Waiting due
  none

## Recurring due
  web#52  Check documentation links  due 2026-09-17

## Active
  none

## Main
  ios#63  Improve keyboard navigation  [rank 1]
  android#74  Explain an unavailable result  [rank 2]
```

In this example, someone needs a review before I start improving keyboard navigation. That's easy to miss if I go straight to browsing new tasks. Our instructions tell the agent to read the report in order and handle those PRs first. I can still skip ahead, of course. The command puts the reminder in front of me.

I also like the explicit “none.” It tells me the command checked, so I don't need to wonder whether another list is hiding somewhere.

The default view includes the first 20 top-level Main tasks and their included subtasks. If there's more, a note points to `bin/next --all`. I don't need the whole backlog just to get started.

The dates are dates to check something, not predictions about when it will be finished. If we're waiting for a dependency, we can set a date to look again. The command brings it back to our attention then.

## Make the First Step Easy to Repeat

The command has the same name in our web, iOS, and Android repos. The actual script lives in the web repo; the other two call it. That also works from worktrees, the separate checkouts we use for different tasks.

Our agent instructions say to use `bin/next` when answering “what's next?” They also say not to piece together another queue from planning files or issue descriptions. I don't want a second opinion on the order every time I open a session.

There's no hook running it automatically. It's a written instruction we expect the agent to follow.

We've allowed it in all three repos' Claude permission settings and in the web repo's Codex rules. The iOS and Android Codex rules still lack that entry, so the setup isn't consistent everywhere. The command also needs a working GitHub login with access to the board.

Because it only reads, running it doesn't commit me to anything. It won't assign a task, create a branch, or start writing code. I can look at the list before deciding to begin.

## Leave a Queue You Can Return To

We use `bin/next` when closing a session too. The agent checks which changes actually made it into the main branch and records the unfinished work. Then it runs the command to check that the list matches what we agreed to do next.

An open PR should still show up. A task we've put off should have a place in the queue, or a date to check it again. That makes it easier to leave the work and come back later.

A [handoff prompt](/write-the-handoff-before-you-stop/) can carry details about a tricky problem. It doesn't need its own copy of the priority list.

There's still some bookkeeping. To recognize Active work, the script checks issue assignments and references in assigned open PRs. Those references need a line beginning with `Fixes` or `Refs`; a passing mention doesn't count. We removed the separate In Progress status, but we still need to keep assignments and links accurate.

## What Changed, and What Still Costs Work

I haven't timed how quickly I start or measured how much more I get done. I just find this easier to use. I can sit down, run one command, and see the work we'd already decided mattered.

It took a few fixes to get here. Our first board had too many overlapping categories. The script initially made expensive API requests. Filtering the report to one repo also produced false warnings when a task's parent lived in another repo.

We fixed those problems and added tests for the report order, empty sections, dates, list size, and cross-repo checks. A starting command needs to be dependable. Otherwise, running it gives me another problem to investigate.

I still have to decide how to do the task, and sometimes the priorities really do need another look. But on an ordinary pickup, I don't need to ask an agent for a fresh set of suggestions. We can read the list and start there.

## Lessons Learned

- Put tasks in order when you add them, while you remember why they matter.
- Show reviews and other work people are waiting on before offering new tasks.
- Before stopping, check that the next session will be able to find the unfinished work.
- Test missing data and tasks that span repos, so the first command of the day is one you can trust.
