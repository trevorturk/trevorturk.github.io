---
layout: prompts
title: "Just Run bin/next"
post_url: /just-run-bin-next/
post_date: 2026-09-17
model: "GPT-6"
last_edited: 2026-09-17
last_edited_by: "GPT-6"
---

**Prompt 1 (originating prompt, supplied from the previous session):** "I think we blogged about our github issues migration, which has been going really well, but I'm so happy with the 'bin/next' stuff I wonder if that deserves a post of its own? there's something about 'putting yourself on the rails' and 'not giving you any excuses' or 'reducing friction when getting started' etc that has been so valuable for me."

**Prompt 2 (follow-up, supplied from the previous session):** "I'd like the title to be a bit better, maybe naming bin/next somehow."

**Prompt 3 (current request):** "I'd like a new blog post, here's some info from a previous session where I didn't like the way it was writing the post, but there should be plenty of info to start with:"

The request included this previous-session handoff. Its research claims are reproduced as supplied, not endorsed as current facts; corrections and omissions are recorded below. Line wrapping is normalized.

```text
⏺ Here's the kickoff prompt. It gives the thesis, the verified facts, and the constraints, and leaves the prose and the final title to the next agent.

Title candidates that name the command, best first in my view:
- "bin/next: One Command, No Excuses"
- "Just Run bin/next"
- "bin/next Puts You on Rails"
- "Start Every Session with bin/next"

Write a new post for this blog (Mechanical Turk, trevorturk.github.io) about `bin/next`,
the "what's next?" command in the Hello Weather repos. Read AGENTS.md first, then follow
the blog-post-generator skill (.claude/skills/blog-post-generator/) for voice, structure,
the template, and the "How This Post Was Made" section. Use the pull-requests skill for
the PR. Branch, never push to main. Make editorial calls yourself and record them; don't
stop to ask.

## The originating prompt (record this verbatim in _prompts/ and the PR)

"I think we blogged about our github issues migration, which has been going really well,
but I'm so happy with the 'bin/next' stuff I wonder if that deserves a post of its own?
there's something about 'putting yourself on the rails' and 'not giving you any excuses'
or 'reducing friction when getting started' etc that has been so valuable for me."

Follow-up: "I'd like the title to be a bit better, maybe naming bin/next somehow."

## Thesis

The expensive part of starting work is deciding what to start. Make that decision once,
on the board, when you rank the item. Then have one zero-argument command read it back.
At session start there is nothing left to decide, so there is no excuse not to begin.
This is about the human as much as the agent: both ask the same question and get the same
answer from the same source. The title should name `bin/next`. Candidates: "bin/next: One
Command, No Excuses", "Just Run bin/next", "bin/next Puts You on Rails". Pick one or
write a better one.

## What's already published (don't repeat it, link to it)

- _posts/2026-09-03-moving-the-queue-out-of-git.md owns the mechanism: the org Project
  board, the print order, the GraphQL cost cut (1,472 points to 9), the iOS/Android
  wrappers, permissions. It is the only post that mentions bin/next, and only as plumbing.
- _posts/2026-07-29-plans-disposable-skills-durable.md owns "nothing is a valid answer"
  and documents the old prose "What's-Next State Machine" that bin/next replaced.
- _posts/2026-09-09-run-a-job-search-like-a-project.md has the closest existing sentence
  ("This removes a recurring planning task from the model"), but it is a model-side
  argument. The human-side version is unwritten.
- _posts/2026-09-09-write-the-handoff-before-you-stop.md and
  2026-09-09-keep-agent-memory-in-the-repo.md are adjacent; both note the user finds the
  workflow valuable and set that aside as unmeasurable. Same house style applies here:
  report the lived experience as a qualitative claim, with that caveat attached.
- Nothing in the corpus discusses friction, excuses, rails, or where to start.

## Research to do yourself (verify everything below before using it)

Sources: ~/Code/helloweather/{web,ios,android}. Read web/bin/next (about 463 lines of
stdlib Ruby, a NextCli module), web/test/bin/next_test.rb, the 8-line wrappers in
ios/bin/next and android/bin/next, the planning skill at
web/.agents/skills/planning/SKILL.md, and the bin/next passages in each repo's AGENTS.md
and PLANS.md. Use `git log --follow web/bin/next` and `gh pr view` for history. The
script is read-only, so running it is safe.

Findings from a first research pass, as of 2026-09-17:

Design choices that remove friction
- Zero arguments. Same command name in all three repos, and it works from git worktrees.
- Allowlisted in both Claude Code and Codex settings, so it never triggers a permission
  prompt. Mutating gh commands still prompt.
- Output order is priority order: PRs needing your attention, Waiting due, Recurring due,
  Active, then the ranked Main queue (capped at 20 roots, with a footer pointing to --all).
  The skill says to answer in that order.
- Review work prints before new work, so you can't skip reviews for something shinier.
- Empty sections print an explicit "none".
- "Active" is derived from assignees and Fixes/Refs lines in assigned PRs. There is no
  In Progress status to maintain, so it can't go stale.
- Due dates mean "next check date", so scheduled work surfaces on its own.
- Warnings for unplaced issues and undated waits replace an inbox to groom.
- It fails loudly on a partial board read, and error messages say what to run.

The rails are a written rule, not a hook
- web/AGENTS.md: "`bin/next` is the only 'what's next' — never answer it from `PLANS.md`,
  a spec, or issue prose."
- android/AGENTS.md: "Before starting substantive work here: run `bin/next`".
- No hooks exist in any of the three .claude/settings.json files. It is convention plus
  the allowlist.

One command for both ends of the day
- PR #2036, "Make bin/next the single session handoff": "the one command for both morning
  pickup and session close".
- The planning skill: "Treat 'can we close this session?' as a live closeout audit, not a
  request for reassurance", and it runs bin/next to do the audit.

Before and after
- It replaced a hand-maintained PLANS.md queue plus a four-step prose "What's Next Rule"
  that a person or agent carried out by hand.
- Since 2026-09-02: roughly 98 issues closed and 200 PRs merged across the three repos.
- Commits touching planning files: web 50% to 3%, iOS 54% to 9%.

Pain points worth including
- The taxonomy was redone on day one (tracking issue #2043: "First-day use exposed
  conflicting Bucket, Status, rank, label, verdict, and assignment meanings").
- The API cost blowup and the retired scheduled digest (#2077).
- --repo false-positive warnings for cross-repo parents (#2107).
- Fewer than half of merged PRs carry the leading Fixes/Refs line the rule asks for.

## The framing trap

Do not claim the script does everything for you. It does not pick a single task, create a
branch, assign the issue, or print the next command. It prints a ranked report and every
other source is forbidden as an answer. The honest claim is: one canonical, prompt-free,
read-only answer, and nowhere else to look. That supports "no excuses" on its own terms.

## Sanitization (hard requirements)

- Fabricate all sample output. Real issue titles expose the roadmap, a data-source list,
  vendor names, pricing items, and an open security bug. None of that may appear.
- Anonymize collaborators. No names or handles.
- Do not quote web/PLANS.md (competitors, revenue targets, vendor cost docs).
- Generalize the OWNER / PROJECT_NUMBER / REPOS constants and describe the token setup
  generically. Sanitized code excerpts from the script are fine.

## Deliverables

- _posts/YYYY-MM-DD-slug.md with a full timestamp in `date:`.
- _prompts/slug.md with the prompts above and your edit history.
- A PR with a "Prompts Used" section.
- Do not use Claude/agent memory for anything; durable notes go in repo files.

I filled two gaps without checking:
- Skill path: I gave it as .claude/skills/blog-post-generator/ based on AGENTS.md. I didn't confirm Codex can see skills there, so you may need to
  point it at the right location.
- Repo numbers: the counts and percentages come from one research pass today. The prompt tells the next agent to re-verify them, and they'll drift
  if the post is written later.
```

## How This Post Was Made

Generated by GPT-6 using the repository's blog-post-generator and pull-requests skills on September 17, 2026. Read `AGENTS.md`, the current script and its tests, both native-repo wrappers, the planning skill, the planning passages in all three repos, and their checked-in permission settings. Checked the script's Git history and the source records for the session-close change, the first-day board redesign, the API-cost correction, and the cross-repo placement fix. The web checkout was at `f9356d264` during research.

Before drafting, the plain-language explanation was: the board stores what you decided matters, and `bin/next` brings that decision back when you sit down to work. Reviews and due checks appear before new work. The person and agent still need to understand and do the task, but they can begin without assembling another list of possibilities. Closing a session includes checking that unfinished work will appear on the next run.

Judgment calls:

- Chose “Just Run bin/next” to name the action without promising that the script eliminates judgment or forces execution. Kept the human experience central and attributed reduced friction to the owner's report, with no measured productivity claim.
- Used a focused essay rather than filling out another implementation tour. Linked the migration, old planning procedure, and handoff posts. The older planning article shows a five-step rule after an update; avoided repeating the handoff's four-step count.
- Corrected the blanket permission claim: all three Claude settings allowlist the command, but only the web Codex rules explicitly list it. No checked-in Claude settings file has a hook. These settings do not guarantee execution in every sandbox or authentication state.
- Described review-first behavior as output order plus a written rule, not an inability to skip reviews. Described Active as derived from maintained signals, not impossible to become stale.
- Omitted issue/PR totals, planning-file percentages, and the PR-reference compliance percentage. They were not remeasured or used as evidence. Kept the verified reference-parser limitation instead.
- Kept the three historical corrections brief; the earlier post owns the detailed API mechanism. Did not present the historical API-point measurement as the current cost.
- Fabricated the entire sample report. Published no live titles, collaborator identities, board URL, private source links, business details, credentials, or quoted planning-index content.
- Put this process record in the prompts file, as the current skill requires. The post layout links it from the front-matter subheader.

**Editing pass (2026-09-17):** Reviewed the draft against the source and the plain-register rules. Kept the default report's limit precise (top-level items, not total rows), the PR-link parser requirement explicit, and the distinction between recorded priorities and implementation decisions. The prompt history includes the supplied handoff so readers can see which earlier claims were corrected or omitted.

Validation: `bin/verify-post --new` reported 1,306 prose words, one sentence over 25 words, and no retired phrases. `bundle exec jekyll build` passed. Checked the rendered post, prompts link, internal links, and linked heading. The source script's tests passed with the repo's bundled environment: 19 runs, 97 assertions, no failures or errors. Initial attempts outside that environment hit Ruby/Minitest version mismatches. A read-only live run succeeded with no warnings after network access was enabled; only report group names and the warning count were retained in the validation output.

**Rewrite (2026-09-17):** "this wording is so strange, wdym "the owner" here? can you write this more like a blog post from my perspective, ELI5 for the reader, try not to sound so much like an LLM!"

The first draft described the request from outside instead of explaining the experience in the author's voice. Rewrote it in first person as requested. The plain-language explanation before drafting was: I already put my tasks in order. When I sit down to work, I want to see that list, along with anything that needs attention first. This command gets it for me, and I can ask my coding agent to use it too. I still have to do the work, but I don't need another conversation about where to start.

Judgment calls: removed the story about requesting a blog post and the repeated third-person descriptions of the author. Used the first-person experience supplied in the prompts without inventing a daily routine or productivity measurements. Explained the report labels and technical terms where they appear. Kept the code blocks, section headings, date, links, and technical limits; preserved the original prompts and process notes above. Shortened the essay rather than filling it with implementation detail already covered by the earlier post. The original request's quoted phrases remain in the prompt history; the article now explains them directly.
