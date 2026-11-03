---
layout: post
title: "The Skills Behind Our Unattended Agent Runs"
date: 2026-11-03 16:10:00 -0600
summary: "Excerpts from the skills we use to start agent builds, define done, prove the work on a real screen, and turn repeated corrections into setup changes, plus what the approach costs us."
tags: [ai-agents, skills, workflow]
model: "Claude Opus 5.5"
last_edited: 2026-11-03
last_edited_by: "Claude Opus 5.5"
---

In the month to October 30, we merged 568 PRs across the three [Hello Weather](https://helloweather.com) repos. The median merged PR was 21 to 32 lines, depending on the repo. Another 135 PRs were ready for review and waiting.

A few developers I talk to are building similar setups. John Nunemaker published [a write-up](https://www.johnnunemaker.com/dont-make-a-human-do-what-ai-is-amazing-at/) about an agent that opens small PRs and never merges them. [Justin Searls](https://justin.searls.co) has agents record video before and after each fix, because an agent's own report that something worked isn't enough. The questions that keep coming up are what starts the work, how you trust it, and how it gets better.

Our answers are mostly in skills. We have 86 skill folders across the three repos, and most cover one system, like App Store pricing or CDN caching. The ones below are about how the work gets run and checked, so they should transfer. The excerpts are trimmed and sanitized.

## `dispatch`: What Starts the Work

Builds don't run on a timer. A few times a day, I check my plan usage and ask what an agent can build. `dispatch` lists the issues labeled `agent-ready`, which means an agent can build the work alone in one PR and the test gate covers it. It gives each issue a rough cost. Then it asks whether to start one or as many as capacity allows.

```markdown
- **S**: web, one area, no new copy.
- **M**: web across several files, android, or new user-facing copy (translations).
- **L**: ios work that changes a screen (simulator), or anything cross-repo.

The first act of every build: run `bin/next --agent-ready` again and confirm its issue
is still unclaimed. Then push the branch and open a draft PR. That draft PR is the claim.
```

The draft PR keeps two builds from taking the same issue. An agent can't read the usage meter, so a run stops when the list is empty, when it gets a rate-limit refusal, or when I say stop.

This has a real cost. On October 30, the waiting PRs in web and iOS had a median age of 13 days, and they were about ten times the size of the ones that merged. Nunemaker caps open PRs at five per repo. We looked at that and decided against it. We tried showing the count before each dispatch, then took that out three days later. Hello Weather is a side project and I build when there are tokens to use, so the skill now says "Never hold a build for the pile". The open count has gone from 135 to 175 in the four days since October 30, 121 of them on iOS. I'm betting that I can catch up, and I haven't yet.

## `afk`: What Done Means

I start an `afk` run and leave. The goal is to come back to PRs I can review and merge in one sitting, so the skill defines done as a state you can check on GitHub.

```markdown
- Every piece of the plan is an open PR, ready for review and not a draft, assigned to
  the user, with nothing merged.
- The repo's test gate passed on each PR's last commit, and the PR body shows the output.
- A fresh agent on the most capable model has reviewed the finished work once, and the
  fixes you agree with are pushed.
- Each PR that changes what a person sees or receives has been run for real. The proof
  is on the PR.
- One tracking issue lists the PRs in landing order, the calls you made for the user,
  and what is left.

Never in an AFK run, unless the kickoff names the exact thing: a merge, anything on
`main`, a deploy or release, a change to a live vendor, cloud or production system,
anything sent to a person or a customer, a credential, or a destructive action.
```

Two other rules apply before I leave. The pre-flight tests every permission the plan needs, with one simulator tap and one read-only call per CLI. I may be steering from my phone later, and some grants can only be approved at the computer. The first reply also has to ask every question at once, each with the default the agent will use if nobody answers.

The closeout makes a long run readable afterward. Most PRs get linked to the larger initiative they belong to. The leftovers go at the top of the tracking issue, sorted into groups like "Easy yeses" for docs and tool changes that merge in a word, and "Bugs people can hit today", which say how each bug was seen.

## `emulator-qa` and `simulator-qa`: Proof on a Real Screen

A green test suite doesn't show that a screen is right, so `emulator-qa` has a long-run recipe with a checklist on the tracking issue. The rows cover free and member accounts, light and dark, the largest font, rain and snow, offline, permission denied, and a second language.

```markdown
**Prove each row.** Take a screenshot and read it. A row passes only when you saw the
right screen with the right data, and "it didn't crash" is not a pass. Mark each row
passed, failed with its issue or PR, or untested with the reason.

**Stop when** every row has a result, or usage runs out. Keep the checklist current as
you go, so that a run cut off at any point leaves its state on the issue.
```

This works better on Android than iOS, for mechanical reasons. The Android emulator runs with no window, so it doesn't touch the Mac's screen and it keeps running when the screen locks. On iOS, driving the simulator can take window focus, one session holds it at a time, and a locked Mac stops everything until I unlock it. `dispatch` also builds iOS and Android work in cloud sessions, where iOS can't compile or run its tests at all. Those PRs get a `needs-mac-qa` label, and a Mac session runs the owed checks later. The rule is "Never claim a gate ran when it did not."

## `retro`: Repeated Corrections Become Setup Changes

Every two weeks, `retro` reads what people typed in recent sessions and looks for repeated corrections. A script cuts a month of transcripts from about 200 MB to under 1 MB by keeping only the human messages and a snippet of each reply. Parallel readers go through the slices. A pattern survives only if it shows up in two or more slices, or in three or more sessions of one slice. Then the skill picks where the fix goes:

```markdown
Pick the lightest fix that will hold, in this order:

1. A check, when a mechanical mistake repeats after it was written down: a permission
   rule, a `bin/` check, a lint rule, or a test. Prose did not hold.
2. A `reviewbot` rule, for a judgment call about code.
3. A line in the skill that owns the task.
4. A line in the shared section of `AGENTS.md`, only when every session needs it.
5. A cut. Does the line change what an agent would do by default?
```

On October 30, we added PRs closed without merging to what the retro reads. That idea came from Nunemaker, who keeps a rule file built from rejected PRs. We kept our existing skills as the home for those rules and only added the input. The November 1 run brought five proposals with quoted evidence, and I approved all five. The retro is adapted from two open-source skills.

## `investigate`: Answers With Labels

An agent's summary reads the same whether it checked something or guessed. `investigate` makes it mark the difference on every claim:

```markdown
- Found: something written says so. Cite it.
- Supported: several indirect signs agree. Cite them.
- Inferred: your own reading. Say so.
- Unknown: say what would settle it.

Keep the labels when you summarise. A summary that drops them turns a guess into a fact.
```

For "is this safe", it ranks the proof from "you said so" up to "you reproduced it in the running app", and the agent has to say where it stopped. Its bug recipe starts by naming the evidence that the bug is real. With no evidence, the agent writes a won't-fix with the trigger that would revive it, instead of building a guard that "might help".

## `performance-optimization` and `deploy-check`: Numbers That Reach Production

In October we removed `Rack::Deflater` so Cloudflare could do the compressing, and our benchmark showed a 2% CPU saving. Nothing changed in production. Falcon was compressing every response itself, and our probes call `Rails.application.call`, which skips Falcon's middleware. A week later we turned off Falcon's gzip layer too, and that change delivered the saving. Our benchmark puts it at about 0.85 ms per forecast response, and we estimated Heroku would send Cloudflare about 14 times more bytes. The skill now requires each run of performance work to prove its benchmark can see the cost before the first change. It also makes the agent question its result before reporting it:

```markdown
- Why not double? Name what limits the result, from a profile.
- Were both arms run alike (same Ruby, YJIT, env, warm caches)?
- Is it within limits? A piece that takes 1% of a request can save at most 1%.
- Does it reproduce across interleaved rounds? Report the median and the range.
- Did the changed code run inside the timed region?

Call the result inconclusive when you cannot name the limiter.
```

The same skill keeps a "do not re-attempt" list, so an agent doesn't redo failed work. Swapping JSON libraries is on it: encoding a prebuilt hash takes about 0.06 ms, while building our output takes about 2.68 ms.

`deploy-check` is read-only and new as of November 1, so it doesn't have much of a record yet. It compares each change with the hours just before the deploy and the same hours on an earlier day, and it blames the deploy only if the number moved at the deploy. The verdict is healthy, regression, or inconclusive. Anything it couldn't read gets named rather than counted as passed.

## Lessons Learned

- Write "done" as a state another session can check, not as a task list.
- Keep merge, deploy, and customer contact with a person, then pre-approve everything else at kickoff.
- Test permissions before you leave, while someone can still answer a prompt.
- When a mistake repeats after you wrote the rule down, add a check instead of more prose.
- Decide up front what happens when PRs arrive faster than you can review them, and measure the queue.
