---
layout: post
title: "'Nothing to Change' Is a Valid Review Verdict"
date: 2026-08-05 08:20:00 -0600
summary: "Fresh-eyes review rounds worked. Acting on every finding automatically didn't. The fix was a rule that a person sees the sorted findings before anything changes, and that rejecting all of them is a normal outcome."
tags: [ai-agents, code-review, workflow]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Failure Mode We Didn't Predict

Every review round was ending the same way. The reviewers sent back their findings. The main session then fixed all of them straight from the report, left PR comments about the fixes, updated the PR body, and sometimes filed issues for the rest. The fixes were new code, so the session proposed another round to review them. That round found more things, which meant more fixes. The commit that finally changed the policy describes it plainly: the rule was written "after repeated sessions patched straight from review reports and looped review→changes→review ad nauseam."

The review rounds themselves were fine. Last week's post on [adversarial review rounds](/adversarial-review-rounds/) described the practice. Before merging a risky change, we start two or more reviewer agents with fresh context and read-only access, give each a different angle, and sort their findings in the main session. Weeks of doing this across the three [Hello Weather](https://helloweather.com) repos kept catching real problems: wrong root causes, confident but wrong numbers, and a revenue bug.

The step after the catch was the problem. The original skill said the main session "adjudicates each (confirmed / refuted / judgment-call) and applies fixes once." An agent reads that as a pipeline. Findings go in, fixes come out. Each reviewer did its job, and each edit on its own was defensible. The process as a whole was the problem. A review round had turned into a batch of code changes with a report in front of it. Asking "should we get fresh eyes on this?" had become "should we let agents with no context make an unlimited set of changes?"

## The Amendment

On 2026-08-04 the project owner shipped the fix to all three repos, iOS, web, and Android. It went in the same PR that picked a mid-tier model for delegated work, which is its own post: [Delegate to the Middle Model](/delegate-to-the-middle-model/). The change to the "Adversarial Review Rounds" section of each repo's code-review skill (`reviewbot` on the web repo) is small. Two bullets were edited and two are new. The only thing we cut from the quote is an internal attribution:

> - **Read-only, report-back-only.** No builds or side effects, nothing posted or edited. Reviewers produce findings, not verdicts: the main session adjudicates each (confirmed / refuted / judgment-call) and digests the rounds into one cohesive analysis. Reviewer briefs are delegable work under AGENTS.md's model-selection rule; the verdict is not.
> - **User checkpoint before any fix.** The digested analysis goes to the user first — no edits, commits, or `gh` mutations (PR comments, PR-body updates, issue filing) from review findings until the user has reviewed the report and approved (recurring failure mode). Fixes then land once, in the main session.
> - **The digest is a filter, not a to-do list.** Expect noise: reviewers reliably surface unrelated, minor, or stylistic findings. Judge each on importance and scope, and reject freely with a short reasoned objection — "all findings rejected, nothing to change" is a valid and common verdict, not a failed review. Real-but-out-of-scope findings become proposed GitHub issues (`gh-issues` skill), offered in the report, never folded into the PR.
> - **Converge, bounded.** Usually one round; each re-dispatch needs its own user go-ahead, and only after material rework. A round must not automatically produce a batch of changes — one yielding only rejected, minor, or proposed-as-issue findings ends the cycle.

The pull-requests skill, at the point where it talks about the merge decision, gained one sentence:

> Rounds are read-only, and the digested findings go to the user for approval before any fixes are applied.

Three parts of that text matter.

### Read-only now covers the aftermath

The original rule said reviewers can't edit. The new rule says the session that reads their reports can't either: no edits, no commits, and no `gh` commands that change anything. A PR comment, a PR body update, and a filed issue all count as changes, because an agent that can't edit code will happily "just leave a comment" instead. Nothing moves until a person has read the digest, the sorted summary of findings.

### Each re-dispatch needs its own go-ahead

The old stopping rule was "stop when a round returns only nits." The agent applied that rule to itself, and applied it loosely. Now a person approves every new round. A round that returns only rejected, minor, or issue-worthy findings ends the process.

### Rejection is named as success

This is the sentence the post is named after. Without it, an agent treats a review report like a task list, because agents are trained to finish task lists. Marking a finding "rejected" feels like leaving work undone. The skill now says in plain words that it isn't. A digest that rejects all thirty findings, each with a short reason, means the process worked.

## The Digest Is a Filter, Not a To-Do List

That sentence resolves a tension in the original design. We run several reviewers because we want different angles: fresh eyes, a claims audit, a devil's advocate, a dry run by an implementer. Those angles look where the author wasn't looking. So most of what they report will be unrelated to the change under review. A claims auditor flags an imprecise comment three functions away. A devil's advocate reopens an architecture decision from March. A fresh-eyes reviewer, with no idea what is in scope, reports everything.

Reviewer noise isn't a defect to prompt away. It's the cost of different angles. A reviewer that only returned in-scope, must-fix findings would be looking where you already looked.

So sorting is the step that matters. The session with full context does the sorting, and a person confirms it. Skip that checkpoint and the noise gets implemented instead of filtered. Every style nit becomes a diff, every out-of-scope remark becomes scope creep, and the change ends up buried in churn.

The sorting table is short:

| Finding is... | Disposition |
|---|---|
| Real and in scope | Proposed as a fix in the report; landed once, after approval |
| Real but out of scope | Offered as a *proposed* GitHub issue in the report - never folded into the PR |
| Minor, stylistic, or unrelated | Rejected with a short reasoned objection |
| Wrong | Rejected, with *why* written down |

Two rows need a note. Out-of-scope findings become issues proposed in the report, not filed. Filing is a `gh` change, and the person may know the finding is a trade-off we made on purpose. Every rejection gets one sentence of reason. Writing the reason keeps the sorting honest, and sometimes it shows that the code needs a comment more than a change.

## The Filter, Live

The new rule got tested right away. The PR that introduced it went through a three-reviewer round of its own: fresh eyes, a claims audit of the pinned agent definition, and a devil's advocate. The round produced about thirty findings. Six were accepted and landed as two commits before merge. Among those six were restoring a deleted delegation mechanism and, fittingly, adding the `gh` ban and the loop terminator to the bullets quoted above. The other two dozen were rejected with reasons. That's a 20% acceptance rate on a change we wanted to get right, from reviewers doing what we asked.

Three more cases from the same weeks on the web repo show the other verdicts.

### A headline objection, refuted by experiment

A small fix raised an HTTP server's request-line limit to stop a recurring class of platform errors. Scanner probes with huge query strings were causing them. The fix went through five separate reviews, and the main objection was strong. The hosting platform's documentation says its router caps request lines at 8KB, so oversized requests should never reach the app, and the fix should do nothing. Instead of arguing, the session tested it in production. It sent probe requests at 10KB, 20KB, 40KB, and 70KB against the live app. The documented cap was wrong for the current router generation. The 10KB and 20KB probes were forwarded and produced the same errors we had seen. The test also found the router's real forwarding limit, which gave the new limit its safety margin. The objection was rejected with the evidence attached. The finding still improved the PR, because "why 64KB?" now has a measured answer.

### A known-misleading metric, deliberately kept

An audit of a daily subscription-metrics digest flagged that one count can double-count a user in a rare same-day case. Real finding, correctly identified. Verdict: rejected as negligible at current volume. The condition for revisiting it is written in the PR ("if this number ever exceeds that one, revisit"). Sometimes the judgment is: we know, it's fine, and here is when it stops being fine.

### Real bugs in a trusted report

The same audit checked every line of a digest we had read for two months against its sources. It found real reporting bugs. A headline count silently left out an entire cohort. A funnel percentage was computed against a different base than the one shown next to it. A categorization had no "other" bucket, so anything that matched no pattern vanished from the breakdown while still counting in the totals. All accepted, all fixed. The filter doesn't mean "reject everything." When the findings are real, they go through.

Four different outcomes, and a person saw each decision before anything moved.

## Lessons Learned

- **Watch what a good practice does on its own.** Fresh-context review was working. Acting on it automatically was the bug. When you adopt an agent workflow, ask what it does by itself once the reviews come back.
- **Expect a low acceptance rate.** Reviewers with different angles mostly report things you don't need. That shows the angles differ, not that the reviewers are bad. Ours was 6 of ~30.
- **Out-of-scope findings get proposed, never folded in.** The PR stays about one thing. Real but unrelated findings become proposed issues the person can accept or decline.
- **Refute with evidence when the stakes justify it.** A production test beat a documentation citation and turned a blocker into the fix's best justification.
- **Every loop needs a stop that a person holds.** "Stop when it's just nits," enforced by the agent, becomes "one more round" forever. Each new round costs one explicit go-ahead, and a noise-only round ends the cycle.
