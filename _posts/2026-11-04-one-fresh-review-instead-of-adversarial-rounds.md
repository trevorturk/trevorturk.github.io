---
layout: post
title: "One Fresh Review Instead of Adversarial Rounds"
date: 2026-11-04 19:20:00 -0600
summary: "A reader asked for the skill behind our adversarial review post. We've moved on from it as a default, because rounds of skeptical reviewers kept talking us into guards we didn't need. Here's what we do now, why, and generic versions of the skills to copy."
tags: [ai-agents, code-review, skills, workflow]
model: "Claude Opus 5.5"
last_edited: 2026-11-04
last_edited_by: "Claude Opus 5.5"
---

A reader in a Slack group asked whether the skill from our [Adversarial Review Rounds](/adversarial-review-rounds/) post is public. It is now, at the end of this post, along with what replaced it as our default. In July, 64 pull requests across the [Hello Weather](https://helloweather.com) repos mentioned "adversarial" in their descriptions. In the first four days of November, none did, and 53 mentioned a "Fable review", meaning a review by Claude Fable 5.1, our strongest model.

That count is a rough word search, but the change behind it is real. A round sent a change to several independent reviewers, each looking for what was wrong. Now a plan and a PR each get one fresh review from Fable, and the session that built the work decides what to do with the findings. Rounds are still in the skill, for changes where being wrong is expensive.

## What Went Wrong With Rounds

Rounds found real bugs, and the examples in the earlier post still stand. One reviewer with no session context caught an Android billing fix that could have wiped a paying subscriber's membership.

The trouble was the other findings. Ask several agents what's wrong with a change and each will find something. Much of it was true and didn't matter.

The clearest case is an iOS PR from August. It moved repeated calculations out of view loops across the app, watch, and widgets, which is a speed-up that shouldn't change what anyone sees. Seven reviewers read it across two rounds. The first round suggested clamping one chart range, so an inverted range from our server wouldn't crash. The second extended that to all four sites and a shared helper, and added a guard against a zero step. One reviewer ran the code and showed that a zero step really does crash.

The session judged each finding, and I approved them. Then I looked at the result on August 4 and rejected all of it as speculative defense against values our own server controls. The speed-up [shipped without the guards](/the-warehouse-of-closed-prs/).

Those reviewers were right about the code. A zero step does crash. Our server doesn't send one, and a reviewer reading a Swift file can't tell that. That's our reading of why this kept happening: a reviewer with no context can't know which inputs really occur, so "could break" looks the same as "breaks". Asking for problems also rewards listing them.

The web repo had the same experience a few days earlier. A four-agent round on one vendor's precipitation field correctly decided not to change it, and found a real bug, a feels-like low above the actual low. It also produced a hardening PR with no change to any recorded output. Some of that PR was worth having, like not building 1,000 objects for a 15-day forecast. The rule against the rest merged the same day, and its PR records the lesson as "we're clearly reaching for things to fix, instead of finding actual issues."

A guard adds code to read, and it can trade a crash report for a screen that's quietly wrong. Our default is the happy path: write the case that happens, and let the unexpected raise so the error tracker shows it. We make an exception when we have a reason to be careful.

## How the Rules Changed

The shift took three months of small edits, not one decision. All dates are 2026.

| Date | Change |
|---|---|
| July 28 | Rounds added to the review skills |
| July 30 | A fix or guard has to name its evidence before it gets built |
| August 4 | Rounds get a filter: reject findings freely, and "nothing to change" is a valid result |
| October 13 | The evidence rule splits in two. "Write the happy path" bans guards for unseen states. "Fix real bugs" says a reproduction or a walked code path is enough, and no customer needs to have complained. iOS crash rules stop asking for guards on API shapes the server never sends. |
| October 15 | Review findings get ranked by production likelihood times cost. "A failure that needs an input nothing in production sends is not a finding." |
| October 22 | One fresh review before any plan or code PR, after I'd asked for it by hand in 10 sessions in 10 days |
| October 29 | That review stops waiting for me: the session applies the fixes it agrees with and reports what it rejected |

The July 30 and October 13 rules sit in `AGENTS.md`, which loads in every session, as well as in the review skill. Guards get written while building, so a rule that only reviewers read comes too late.

## The Loop

The current loop has four steps:

1. **Fable plans.** A fresh Fable agent writes the plan or reviews the one the session drafted. The `afk` skill defaults to reviewing a draft.
2. **Opus builds.** Claude Opus 5.5 implements the plan.
3. **Fable reviews.** A fresh Fable agent reads the finished work once. Docs-only and one-line changes skip this.
4. **Opus decides.** The session that did the work judges each finding, pushes the fixes it agrees with, and writes on the PR what it adopted and rejected.

Part of the reason is budget. Fable allowance runs out, and I've typed things like "we're almost out of fable allowance until 6pm, so use opus to collect". So we spend Fable on reading and judging. On this Mac since October 1, 232 of 235 main sessions ran mostly on Opus 5.5, and 477 subagents ran on Fable. A rough keyword match on their descriptions puts 363 of those 477 on review, planning, or judgment work. Cloud sessions aren't in those numbers.

Step four reverses advice in the earlier post, which said to delegate the reviewing and keep the verdict on your most capable model. We now give the strongest model the review and let the builder decide, because deciding depends on knowing the system. The session that built the change knows which shapes our server sends and which options we already turned down.

A web PR from late October shows the loop working. It added a job that reports scheduled tasks that didn't run. The plan review changed three things, including tying the check to the scheduled time, because a window based on the job's start time could skip a run or check it twice. It also excluded the job's own task, which would have raised a false alarm after the first deploy. The PR review then found that our error tracker's client drops `level: "warning"` when you pass it a block, so every warning would have arrived as an error, and it cited the line in the gem. Each of those is a failure with a concrete input, and none of them is a guard.

Rejections look like this, from two other PRs:

> Rejected: handling a window with no readings at all, because we have never seen it and the repo adds no guard for an unseen state.

> Rejected: treating a vendor's `-999` as missing here. No recording shows it on a current reading, so it would guard a case we haven't seen.

The second quote has the vendor's name removed. From October 22 to November 4, 41 of the 395 PRs we opened mention both "Fable review" and "Rejected".

This loop has limits:

- **The author judges criticism of its own work.** The earlier post warned about exactly this bias. Our answer is that rejections go on the PR with a reason, and I read them at merge. That catches a bad call late, not early.
- **One review misses things.** On another PR, Fable walked every branch of a stall detector and found no bugs. A Copilot review then found a timing bug that could report a real stall as the wrong kind.
- **Running the code still matters.** A review is evidence, not proof, so `afk` also requires running the change for real.

## When We Still Run a Round

The round triggers are the same: cross-target rules, persistence and entitlement, layout and geometry, and plans about to be built. On iOS, a PR that changes snapshot files gets a round proposed once its checks pass. I still ask for one by hand. In October I asked for one on just a purchase-unlock check, "to be super sure".

That's our reason to be careful: a wrong call costs a customer something they paid for, or it's hard to take back. iOS also keeps its language-level rules, like no force unwraps and no unchecked subscripts, because those crash regardless of what the server sends.

## Results

- **What changed:** PRs mentioning "adversarial" went from 64 in July to 16 in October. PRs mentioning "Fable review" went from 12 to 204.
- **What it cost:** one reviewer per change instead of two to seven, but on nearly every PR instead of only risky ones. The 477 Fable agents above are where much of our Fable allowance goes.
- **What got worse:** some guard we didn't add will eventually be the crash we wanted to avoid. The iOS rule change named one: if the server ever sent an empty daily list, the screen would show blank data.
- **What we didn't measure:** guards added per PR, or bugs that reached users. Opus 5.5 arrived in the same stretch, so better models may explain part of this. The evidence is my own read of the PRs.

## Lessons Learned

- Require each finding to name an input that happens in production. Drop the ones that can't.
- Let the agent that knows the system judge findings, and make it publish what it rejected.
- Put the no-guard rule in always-on instructions, because guards get written during the build.
- Spend your strongest model on planning and reviewing when the budget is tight, and build on a cheaper one.
- Name in advance which changes get heavier review, so that "be careful" isn't the default for everything.

## The Skills

These are adapted from our files as of November 4, 2026. We removed product and vendor details and swapped "Fable" for "most capable model", so they'll outlast a model name. They assume Claude Code but should port to other tools. Edit the examples to match your repo.

We left out `reviewbot`'s stack-specific checks and its recipe for building competing versions of a change. [Yesterday's post](/skills-behind-unattended-agent-runs/) covers `dispatch`, which starts these runs.

### Shared rules for `AGENTS.md`

The model-split bullet is written for this post. It replaces a bullet in our own `AGENTS.md`, which, like our `CLAUDE.md`, still says to keep planning and review verdicts in a main session on the most capable model. That isn't what we do.

```markdown
- Write the happy path and fail fast: raise on bad input and let unexpected errors
  surface. Add no guard, fallback or rescue for a state you have not seen happen, or for a
  shape our own server or app never produces; trust the repo's own handling of missing
  data.
- Fix real bugs, reported or not. A bug is real when you can show it: a reproduction, a
  failing test, an error-tracker event, a user report, a fixture value, or a code path you
  can walk with a concrete input to the wrong result. Any one is enough; no customer needs
  to have complained. "Could be nil" and "might break someday" are not evidence. Say when
  a fix is not worth doing and what would revive it. If a change makes what a user sees
  read worse, technically-correct loses (`reviewbot` skill).
- Split the work by model. A fresh agent on the most capable model writes the plan or
  reviews your draft, and reviews the finished work. The working session can run on a
  cheaper model: it builds, and it decides which findings to adopt, because it knows the
  system.
- Before you hand the user a plan or a code PR, get one fresh review on the most capable
  model. It is not a review round, so it has no gate: judge each finding yourself, apply
  the fixes you agree with, then hand it over and say what you adopted, what you rejected
  and why. Any other stop still applies to a fix, such as a production command. Skip the
  review for docs-only and one-line changes.
```

### `.claude/agents/implementer.md`

```markdown
---
name: implementer
description: Implementation agent on the latest Opus with medium reasoning effort. The preferred tier for dispatched implementation work. Bounded effort biases toward simpler, brief-faithful output. Use for clean-room implementations, mechanical refactors, and bounded PR work.
model: opus
effort: medium
---

You are an implementation agent for this repository. Follow the brief you are given
exactly and stay strictly inside its stated scope. Briefs may also be read-only (audits,
verification); the same scope discipline applies.

Core discipline, non-negotiable:
- Bias to the least machinery that satisfies the brief. No abstractions, generalizations,
  caches, concerns, or "future-proofing" beyond what the brief names. If you notice a
  possible improvement outside scope, note it in your report. Do not implement it.
- Changes required to make the brief's change correct and passing (call sites, affected
  tests, per-environment config) are in scope even when unnamed.
- Read the repository's AGENTS.md before changing anything and obey it. Its rules are in
  scope for every brief even when the brief does not name them.
- The caller owns branch and worktree setup: work only at the absolute paths in your
  brief, and never create worktrees or branches unless the brief says to.
- Prefer small, obvious code over clever code. A reviewer must be able to read your diff
  top to bottom in one sitting.
- Pass absolute paths in every file operation.
- Done means the brief's stated change is made (or its read-only question answered) and
  whatever verification the brief allows has run. If the brief is ambiguous or needs
  something outside its scope, stop and report instead of guessing.
- Finish with a short report: files changed (absolute paths); verification run (command
  and result), or "not verified" and why; anything you could not confirm and where you
  looked; out-of-scope observations.
```

### `.claude/skills/reviewbot/SKILL.md`

The "Adversarial Review Rounds" section is the one the reader asked about. Apart from pointers to other skills and files, it's our current wording. A couple of anti-pattern rows come from our iOS copy of the skill. The "Fresh review brief" section is new here, written from briefs our sessions have been sending.

```markdown
---
name: reviewbot
description: Code review for simplicity, scope discipline, defensive-code removal, fail-fast, dependency workarounds, and operational safety. Use for "reviewbot review", adversarial rounds, or a fresh-eyes second opinion.
---

# Reviewbot

Use this as an opinionated review lens. Lead with concrete findings, but apply these
preferences while deciding what deserves a finding.

## Priority Order

1. Preserve data correctness and API compatibility.
2. Keep production reliability and operational safety boring.
3. Prefer the simplest implementation that works in this framework.
4. Fail fast for impossible states; do not hide real bugs behind defensive code.
5. Research dependencies before adding workarounds or abstractions.
6. Keep PRs small and reversible.

## Review Philosophy

Happy path first, fail fast on errors, remove everything else.

- Fix at the central chokepoint with an existing idiom; `git log -S` finds repeated
  band-aids that signal a central fix.
- Remove guards that check states already guaranteed by validations, params contracts, or
  local call shape.
- Let unexpected exceptions surface unless the code is explicitly best-effort telemetry.
- Do not create a service object, concern, wrapper, or new job for a small local change.
- Prefer existing repo wrappers and conventions over new patterns.
- Prefer self-explanatory code; generally avoid comments. Keep one only for a non-obvious
  why the code cannot convey, as a single short line, with a removal trigger on any
  workaround.
- Fix the cause, not the symptom: verify an error actually occurs before handling it, and
  prefer an upstream guard that skips a doomed call over a `rescue` that swallows its
  result.
- Before you write, look for what to remove: delete dead code and fold in a wrapper that
  has one caller, then build on what is left.
- Before you present a diff, read it as its reviewer. Clear beats short, and simple beats
  clever.

## Anti-Patterns To Flag

| Pattern | Stance |
|---|---|
| Guard chains for guaranteed state | Remove as defensive code |
| Guard or fallback for a shape our own server never sends | Remove; we own that data |
| Rescue-and-return-nil for unexpected errors | Fail fast |
| `rescue` for a failure mode not verified to occur | Confirm the real response first |
| Re-checking a guard the callee already enforces | Remove the duplicate |
| New wrapper or service for a few lines | Inline it |
| Workaround before reading the dependency's source | Research the dependency first |
| Comment explaining why a guard or fallback exists | Remove; the test and PR carry it |
| Workaround comment with no removal trigger | Add a revisit date or condition |
| Internal error text shown to users | Use generic user-facing text |
| High-cardinality metric tags | Block unless explicitly justified |
| Technically correct change that reads worse on screen | Describe what a user sees before and after; if after is worse, revert |
| Fix diff dominated by snapshot churn | Each visible change must be the point; enumerate or shrink |
| A helper that builds more than its callers read | Build only what is read |

## Prior Art Before New Patterns

Before adding or reviewing a new way to handle a concern (errors, retries, caching,
metrics), find how the app already does it:

1. Search the concern's keywords across the code, base classes, initializers, and
   dependencies.
2. `git log --grep=<keyword>` and `git log -S<symbol>` show whether the pattern is settled
   or mid-migration.
3. Name the primary pattern, list the places that differ, and extend the primary one.
4. A claim about vendor data (a field is optional, a value never occurs) is checked, not
   recalled: read the vendor's schema and scan your recorded responses for that field.
   Cite both in the PR. If you cannot reach the docs, say so; never fill the gap from
   memory.

## Review Output

List the problems you'd block the merge for first. Rank them by how likely the failure is
in production, times what it costs when it happens. A failure that needs an input nothing
in production sends is not a finding. For each one, give `file:line`, why it's wrong, how
to show it fails (a failing input, test, command, or before/after output), and the
smallest concrete fix. Then list non-blocking idiom preferences under a separate
"Non-blocking" label, or leave them out when the user asked only for blockers. If nothing
blocks, say so plainly and note residual risk and what wasn't built or tested.

Keep summaries secondary. Do not bury bugs under a long description of what changed.

## Fresh review brief

The default review is one fresh agent on the most capable model, before a plan or code PR
goes to the user. Name the artifact, where the review rules live, and the questions you
most want answered. Leave out why you think the work is right.

    Read-only review of <PR URL or branch>. Do not edit anything, post anything, or run
    builds or production commands. Review rules: <path to this file>. Use absolute paths.

    Question: is this change correct and simple? <Any specific risk to check, as a
    question, such as "what happens for the longest language in the smallest widget?">

    Keep to real problems you can show with a concrete input or file:line. Done means: a
    short list of concrete findings, each with file:line and the input that gives the
    wrong result, or "no findings". Say what you could not determine. Stop and report back
    if the PR or files can't be read.

Judge each finding yourself, apply the fixes you agree with, and put what you adopted and
rejected, with reasons, on the PR.

## Adversarial Review Rounds

For changes where a wrong claim is expensive — cross-target invariants,
persistence/entitlement, layout/geometry, plan docs about to be implemented — propose
independent review rounds before the merge call; dispatch only on the user's go-ahead.
Why: an authoring session cannot review its own blind spots; fresh-context reviewers have
repeatedly caught wrong root causes and confidently-wrong numbers that survived multiple
informed rounds.

- **Independent and diverse.** 2+ agents in parallel, each with a genuinely different lens
  (e.g. fresh-eyes with no session context, claims audit of every stated fact, devil's
  advocate on a named decision, implementer dry-run for plan docs). Name a lens by the
  method it runs ("trace every number to its source"), not a role to play; a method
  borrowed from another field counts. Diversity beats copies; keep each brief minimal —
  never include the session's reasoning.
- **Read-only, report-back-only.** No builds or side effects, nothing posted or edited.
  Reviewers produce findings, not verdicts: the main session adjudicates each (confirmed /
  refuted / judgment-call) and digests the rounds into one cohesive analysis. Reviewer
  briefs are delegable work; the verdict is not.
- **User checkpoint before any fix.** The digested analysis goes to the user first — no
  edits, commits, or GitHub mutations (PR comments, PR-body updates, issue filing) from
  review findings until the user has reviewed the report and approved. Fixes then land
  once, in the main session. The one fresh review before a handoff is not a round and has
  no checkpoint: apply the fixes you agree with, then report what you adopted and
  rejected.
- **The digest is a filter, not a to-do list.** Expect noise: reviewers reliably surface
  unrelated, minor, or stylistic findings. Judge each on importance and scope, and reject
  freely with a short reasoned objection — "all findings rejected, nothing to change" is a
  valid and common verdict, not a failed review. Real-but-out-of-scope findings become
  proposed GitHub issues, offered in the report, never folded into the PR.
- **Evidence over assertion.** Findings need a location and a concrete failure scenario;
  unmeasured claims are labeled estimates. Reviews are sometimes wrong too — re-verify
  anything load-bearing.
- **AFK runs.** The kickoff of a run while the user is away (`afk` skill) approves one
  round and the fixes you agree with. Post the digest on the PR or its tracking issue
  (what was found, what you changed, what you rejected and why) and do not wait at the
  checkpoint.
- **Converge, bounded.** Usually one round; each re-dispatch needs its own user go-ahead,
  and only after material rework. A re-dispatch reads the fixes, not the whole change. A
  round must not automatically produce a batch of changes — one yielding only rejected,
  minor, or proposed-as-issue findings ends the cycle.
```

### Plan rules for a `planning` skill

The plan is a file in `plans/`, deleted by the PR that ships its last step. These lines are the part that makes it reviewable.

```markdown
Write `plans/<name>.md` when an implementation is picked up. Start with a link to its
issue. Structure it as Overview, Context, Ground Truth, Implementation Steps, Tests,
Success Criteria. Each Success Criteria item names the check that shows it holds (a test,
a script, a production number), never the author's reading of the code. Verify every
cited file and symbol; mark uncertain platform facts `Verify:`; include complete
implementation patterns and tests. Before handing it over, ask what the spec leaves out
and which step is most likely to fail; fix what you can and write the rest into the spec.
```

### `.claude/skills/afk/SKILL.md`

This is the skill that runs the loop when nobody's watching. The machine-specific lines are for a Mac and for iOS or Android work, so cut what you don't need. "Safe to close" means every piece of work is on an issue or a PR and nothing exists only on this machine or only in chat.

```markdown
---
name: afk
description: Run while the user is away (AFK, unattended, overnight, autonomous, remote control) and leave tested, reviewed PRs on a merge list. Use for /afk, "going to bed", "carry on", "run autonomously", "find work you can do and bring me PRs to review later", or a kickoff naming this skill.
---

# AFK run

The user starts the run and leaves. They come back to pull requests they can review and
merge in one sitting, and to as few questions as possible. The repo's `AGENTS.md` stays in
force; the one change is under "What the kickoff approves".

## Done

With no other word from the user, done is all of this:

- Every piece of the plan is an open PR, ready for review and not a draft, on the tracking
  issue's merge list, with nothing merged.
- The repo's test gate passed on each PR's last commit, and the PR body shows the output.
- New user-facing copy is final and filled in every language the repo ships.
- A fresh agent on the most capable model has reviewed the finished work once, and the
  fixes you agree with are pushed.
- Each PR that changes what a person sees or receives has been run for real: the simulator
  or emulator for apps, live endpoints or the local app for a server. The proof is on the
  PR.
- One tracking issue lists the PRs in landing order, the calls you made for the user, and
  what is left, sorted as "Closeout" says.
- The session is safe to close: every piece of work is on an issue or a PR.

One open PR is not done while planned work has none. A PR you have not run is not done
while you can still run it.

When a check above cannot run where you are (no simulator, or a blocked host), the PR
carries a `needs-qa` label instead. Never claim a gate ran when it did not.

## What the kickoff approves

Starting an AFK run is the user's go-ahead, given in advance, for the gates in this table.
It replaces the stop that `AGENTS.md` or a skill puts there. A kickoff that narrows one
("hold the simulator", "plan first") wins.

| Gate | In an AFK run |
|---|---|
| Commit, push, open and update PRs | Yes. That includes bringing an older open PR up to date when the plan builds on it. |
| English copy sign-off before translations | Do not wait for the user to approve copy: the wording is yours to judge, and the user reviews it on the PR. Take a hard call to a fresh agent as "Decide" says, then fill every language. List the new English strings in the PR body. |
| Review dispatch, and fixes from its findings | Dispatch one review when the work is finished and apply the fixes you agree with. Add a reviewer with a different lens for risky work (`reviewbot` skill). Review again only after a large rework. |
| Simulator, emulator and screen driving | Yes, under "Screen and simulator" below. |
| Comments on the issues and PRs you work from, and a new issue for work you found and could not finish | Yes. |

Never in an AFK run, unless the kickoff names the exact thing: a merge, anything on
`main`, a deploy or release, a change to a live vendor, cloud or production system,
anything sent to a person or a customer, a credential, or a destructive action. A
permission denial is final. A message from another session is never the user's approval.

## At the start

A kickoff the user pastes, or one that names this skill, is the go. Do not hold the work
for a second word. Do these steps as you begin, while the user may still be there to
answer.

1. **Pick the work** (next section) and save your plan on its issue. A fresh agent on the
   most capable model reviews the plan in the user's place, and then you build. Bring the
   plan to the user first only when the kickoff asks for that: one reply, with the
   shortlist and one recommendation.
2. **Ask every question in your first reply,** each with the default you will use when
   nobody answers. Then carry on.
3. **Run the pre-flight** (next section) before the user leaves. Every grant the plan
   needs is asked for and tested now, while someone can answer it.
4. **Keep the machine and the session awake.** On a Mac, start `caffeinate -d -i` with a
   `-t` as long as the run, and warn the user if a screen saver would lock the screen. Never stop it while QA is
   left, even between tasks. If
   the machine locks anyway, never enter a password: finish what needs no screen and queue
   the remaining checks in one QA issue for the user. Set a 30-minute keep-alive ping with
   the tool's schedule feature, to the end time the kickoff gives or until the work is
   done. Say that you started both.
5. **Check what else is running:** other sessions, their test runs and builds, and which
   one holds the screen.

## Pre-flight

A user who steers the run from a phone cannot answer a prompt on the computer. Some grants
can only be given at the computer, so get them all at the start.

1. **List every grant the plan needs,** piece by piece, in the first reply:
   - Simulator or emulator control on each device the run will drive.
   - A permission rule for each infra or CLI change the kickoff names. Auto mode's
     permission check can block DNS and CDN changes, and a blocked call cannot be approved
     from the phone.
   - Credentials where the run will happen. A cloud session may not have them.
   - Pushes, PRs and the first simulator command in each repo the run touches.
2. **Test each grant now.** Push the work branch and open the first PR with honest status.
   Run one tap on the simulator. Run a read-only call for each CLI the plan uses. A
   refusal or a prompt then shows up while the user can answer it.
3. **Say what cannot be granted ahead.** When a grant cannot be made or tested before the
   user leaves, say so in the plan: which piece it blocks, and what the run does instead
   (skip it, park it on its issue, or leave a kickoff for when the user is back).

## Picking the work

- Start from the work the user named, then your queue and the open issues.
- An issue fits when an agent can build it alone in one PR, the test gate covers it, no
  open PR already covers it, and the result is worth the user's review. Work that builds
  on an open PR stacks on it ("Small PRs, stacked").
- Treat the issue as input, not as the plan. Check its claims against the code and a real
  response, write your own plan, and save it on the issue before you build.
- Work that spans repos gets one session per repo. Save a kickoff for the other repo on
  its issue. It carries what the two sides agreed, what done means, and this skill's name.
  The sessions coordinate by message, and the one that started the work owns the final
  report.

## While the user is away

- **Keep going.** Do not wait for a "go" between pieces. A cancelled or interrupted tool
  call is not a stop: run it again. A stall is not a stop either: try another approach.
  Write up a real dead end on the tracking issue and move to the next piece.
- **Decide.** Make low-stakes, reversible calls yourself. Take a hard one (wording, a
  rename, two workable designs) to a fresh agent on the most capable model with the
  options and the evidence, and act on its answer. Only what neither of you can settle
  waits for the user, and it waits with a default already applied and marked. Copy never
  waits: the user reads it on the PR.
- **Fix what you find.** Fix a bug in your PR inside that PR. Give a small bug next to it
  its own PR. File anything larger as "Closeout" says. Do not list as "needs a person" a
  check you have the tools to run.
- **Small PRs, stacked.** One change per PR. Stack changes that depend on each other
  bottom-up for the user to land.
- **Prove it on the real thing.** Call the live endpoint for every source you touched. Run
  the app and look at the screen you changed. A passing test does not show that a screen
  is right.
- **Log decision points** on the tracking issue as you go, one line each: what you chose,
  why, the evidence (a commit, a PR, a `file:line`), and the result.
- **Wait with a timer.** To wait on a build, a test run or another session, set a timed
  wake-up, and work on another piece of the plan until it fires.
- **Share the machine.** One full suite, build or benchmark at a time.

## Screen and simulator

- A go-ahead for simulator QA covers the whole round: every check, device and retest,
  however it came up. Do not stop to ask again per check. Merges stay with the user.
- One session holds the screen at a time. When another session has it, or the user said
  they are using the machine, wait or hand over. Do not take it back.
- A locked screen, a sleeping display or an unplugged monitor stops all screen driving.
  Keep the display awake, and finish the screen work before any long timed wait.
- Post proof on each PR: what you saw working, what you fixed, and what the simulator
  could not show. A page of screenshots linked from each PR is the usual form.
- When a check will not work after a few different approaches, stop, write down exactly
  what was not seen, and move on.

## Closeout

The tracking issue is where the user finishes the run. Sort everything before you hand
back, and leave the issue open until the user has been through it.

- **Most work goes under its umbrella.** Link every PR and issue to the existing umbrella
  it belongs to, so the user reviews it with the work around it. Only a straightforward
  change, with no knock-on effects and nothing to bundle with, stands alone.
- **The leftovers go on the tracking issue,** as sub-issues, or as a checklist line for a
  PR or an issue with a home elsewhere:
  - Easy yeses: skill, doc and tool changes, and lessons from the run, that merge in a
    word.
  - Quick wins, and anything else worth the user's look.
  - Bugs people can hit today, because the code is on `main`. Say how each was seen.
  - Issues that look finished, with the evidence, for the user to close.
- **Move what the user owes.** A call, a key or a device check goes on the PR or issue it
  is about, so nothing lives only on the tracking issue when it closes.
- **Put the list where it gets read.** It goes at the top of the issue body.

When the user is back, walk the list with them. Merges still take their word, which can
cover the list you just showed them. Close the tracking issue when the list is empty.

## Handing back

1. Check done on GitHub, not from memory: each PR is open, not a draft, on the merge list,
   and its checks match what its body says.
2. Stop what you started: the keep-alive ping, `caffeinate`, background jobs, local
   servers and simulators.
3. When work remains, leave a kickoff on the tracking issue.

Reply: the first line says whether the user is needed. Copy and the calls you made are
never what they are needed for: they review those on the PR when they merge, so do not
ask for a sign-off or a confirmation. Then give the closeout list, each umbrella with its
PRs in landing order, the calls you made for the user (copy, names, defaults), what the
review found and what you changed, what was seen on screen and what was not, what you
dropped and why, and "Safe to close".
```
