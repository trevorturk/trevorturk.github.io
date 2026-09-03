---
layout: post
title: "ELI5 as the Default Output Contract for Coding Agents"
date: 2026-08-07 09:30:00 -0600
summary: "The plain-language digest with a recommendation was the part of our agent review rounds we actually read, so we made it the default for every response, as a checked-in output style shared across our repos."
tags: [ai-agents, claude-code, workflow, communication]
---

## The Problem

One item in our iOS code-review checklist says that once a second model has finished reviewing the code, the agent should *"digest to the user as an ELI5 with a recommendation."* That means a plain-language explanation a smart person with no context can follow, not a list of findings. It ends with what the agent thinks we should do. The digest turned out to be the part of the review round we actually read.

Then we moved our [Hello Weather](https://helloweather.com) sessions to a newer, more capable model tier, and it delegates much more than the last one did. One session on the iOS repo will send the code to a second frontier model for review, start a subagent for each part of the app, and work through a queue of findings before we see a word. The raw output is written for experts: findings keyed by internal shorthand, invariant names a subagent invented three steps back, verdicts that assume we watched the whole thing happen. One person has to decide, and that person was rereading output that was complete but hard to read.

So we asked why ELI5 was a rule for one checklist step instead of the default for everything the agent says.

## The Solution

Claude Code has a feature for this: output styles. An output style is a Markdown file that adds to the agent's system prompt. It changes how the agent *talks* without changing what it *does*. A style can be checked into the repo and turned on by a checked-in setting, so every contributor and every model gets the same rule.

The two files split the job:

- **`AGENTS.md` / `CLAUDE.md`** - project context, conventions, invariants: what the agent should *know and do*
- **Output style** - tone, structure, explanation level: how the agent should *talk to you*

We put the same two files in all three Hello Weather repos (iOS, Android, and the Rails backend).

## Implementation

### The style file

The file is `.claude/output-styles/ELI5.md`. The filename matches the style's `name:` on purpose, for a reason covered below:

```markdown
---
name: ELI5
description: Plain-language digests — bottom line first, jargon defined, recommendation included
keep-coding-instructions: true
---

Write every response to the user as an ELI5 digest: plain language a smart reader
with zero context can follow in one pass. Simplify the explanation, never the work —
code, tests, and technical decisions stay precise and idiomatic.

- Lead with the bottom line in one plain sentence: what happened, what you found,
  or what you recommend.
- Use everyday words. When a technical term is unavoidable, define it in-line the
  first time it appears.
- Prefer a short concrete example or analogy over an abstract description.
- Never lean on internal codenames, ticket numbers, or shorthand without saying
  what they are on first mention; precise symbol and file names stay exact.
- When reporting delegated or reviewed work (agent rounds, adversarial reviews),
  digest it the same way: what was checked, what was found, and a clear
  recommendation — not a raw findings dump.
- End with a clear recommendation whenever a decision is needed.
```

Two lines in that file do most of the work. `keep-coding-instructions: true` keeps the harness's own software-engineering instructions, so the style only adds communication rules on top of them. Without it we'd be trading code quality for readability, which is the wrong trade. "Simplify the explanation, never the work" makes the same promise in the body. The code, the tests, and the review rigor don't change. Only the final write-up to the human does.

### Activating it

One key in the checked-in `.claude/settings.json`. The value matches the style's `name:` field, including case:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "outputStyle": "ELI5",
  "permissions": { ... }
}
```

Both files are committed, so the default travels with the repo to every clone, every teammate, and every session. Anyone can still override it locally in `settings.local.json`. The style applies whichever model is selected, which matters here because a model change is what prompted it.

One version note: the old `/output-style` slash command was deprecated in Claude Code v2.1.73 (March 2026) and is gone from current builds. Styles themselves still work. Manage them through `/config` or edit the settings files directly.

### Documenting the decision

A settings key with no explanation is a puzzle for whoever reads it next, so each repo's `AGENTS.md` gets a short dated section: what the setting is, where the style lives, why it exists ("the same ELI5-with-recommendation shape the adversarial review rounds digest to"), and when to remove it, *drop it if it stops earning its keep*. We wanted an experiment we could reverse cheaply, with a written record. (On 2026-08-13 the repos went tool-agnostic and split the note: `CLAUDE.md` now holds the setting and file path, and `AGENTS.md` states the rule as prose any tool can follow.)

### Attack your own config before trusting it

Before calling it done, we sent out one more round of three agents in parallel. One checked the official docs and the full changelog history. One read community reports (GitHub issues, HN, practitioner blogs). One was told to try to disprove the shipped files, and it went as far as reading the style loader in the shipped Claude Code binary. The mechanism held up: output styles are actively maintained, and the one time they were deprecated (v2.0.30, late October 2025) the change was reversed within days after community feedback. But the round produced four one-line fixes, all in the version shown above:

- **The filename matches the style's `name:`, including case** (`ELI5.md`, not `eli5.md`). An open issue ([anthropics/claude-code#47482](https://github.com/anthropics/claude-code/issues/47482)) reports styles that show as active in the statusline while their body is never injected. A case mismatch between name and filename was isolated as the trigger. Our read of the binary suggested current versions handle it correctly anyway, but matching them costs nothing.
- **The unconditional "end with a recap" rule is gone.** Claude Code's system prompt has an always-on section against verbosity, and a style instruction that fights it every turn is the documented way to lose adherence. The recommendation is now limited to "whenever a decision is needed."
- **Codenames get explained on first mention only, and exact symbol names are kept.** The original wording asked the agent to re-explain every internal name every time. That padding pushes the model toward vague paraphrase when the precise symbol is the point.
- **"Every user-facing response" became "every response to the user."** In our repos "user-facing" means shipped product copy, which has its own stricter voice rules. The style must never read as permission to rewrite that.

The round also found two limits we chose to accept rather than fix. Custom styles got no per-turn adherence reminder at the time. The binary only reinforced built-in styles, though the docs said otherwise. So we expected the agent to drift back toward its terse default late in long sessions, and we judged the experiment on early-session behavior. Claude Code v2.1.238 (2026-08-20) fixed that drift. The second limit is that a committed `outputStyle` also applies to headless runs in the repo. The request for a per-run opt-out ([#81334](https://github.com/anthropics/claude-code/issues/81334)) was closed on 2026-08-17 as already possible: `claude -p --settings '{"outputStyle":"default"}'` overrides the committed key for one run.

One note on sentiment: most teams building custom styles in 2026 build them to make the agent *terser*. A plain-language default goes the other way, and it also pushes against the server's own conciseness instructions, so it can't include rules those instructions will override.

## Results

The one-step rule became the default with nine small file changes across three repos: one style file, one settings key, and one documentation note each. We haven't measured anything yet. Based on the review rounds, we expect:

- **Decisions speed up.** The bottom line comes first, so we read one sentence before deciding whether to read ten.
- **Delegated work stays readable.** The more a session fans out, the more the final digest matters, and the style requires it instead of leaving it to a checklist.
- **The rule survives model changes.** Styles are tied to the repo, not the model.

The cost was the two limits above: some drift back toward terseness late in long sessions, and no headless opt-out. Both have since been lifted, as noted there.

## Lessons Learned

- **Watch what you actually read.** We skimmed the findings lists and read the ELI5 digests. The output you keep reaching for is the one that should be the default.
- **Put communication rules in the repo, not in memory.** A style file and a settings key are versioned, reviewable, and work across tools. An agent's private memory is none of those.
- **Don't write instructions that fight the system prompt.** An unconditional "always recap" rule loses to the built-in anti-verbosity instructions eventually. A conditional one doesn't have to win that argument.
- **Check the docs against the binary.** The docs said every style gets per-turn adherence reminders. The style loader said only built-in ones do. A few minutes of review caught an assumption we'd have relied on, and the gap was real until v2.1.238 closed it.

---

## How This Post Was Made

**Prompt 1:** "see recent claude opus48 delegating, I think we might want to default to ELI5 output in Fable for now, too. how do we do that? we may want to update all helloweather repos, too"

**Prompt 2:** "and add a blog post about this (trevorturk.github.io repo) and also review recent work with date formatting extensions etc"

**Prompt 3:** "dispatch research / adversarial review with opus48 to determine if there's anthropic docs, user sentiment etc about this which might inform our choices, it's ok if this is all correct, but let's dispatch one more round of research, you digest and report back"

**Prompt 4:** "agreed with your recommendation, but also update the blog post pr (we're working on 4 prs in this session)"

Generated by Claude using the blog-post-generator skill, in the same session that made the config changes it describes. The mechanism was verified against the current Claude Code docs before writing; the "Attack your own config" section documents the three-agent research round that then hardened both the config and this post.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the checklist rule and the model change that prompted the work, Results states expectations as expectations plus the accepted cost, and Lessons Learned dropped the two bullets that repeated body sections. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The style file, settings key, nine-file count, and checklist quote match the repos; the style excerpt now carries the file's em dashes, the checklist rule is described as one item rather than the last one, and the exit-condition quote uses the current wording plus a note that the AGENTS.md section split into CLAUDE.md and AGENTS.md on 2026-08-13. The `/output-style` version note now cites the changelog's v2.1.73 deprecation, the styles deprecation is dated to v2.0.30 in late October 2025, and the two accepted limits are marked as since lifted: the drift fix landed in v2.1.238 and issue #81334 closed pointing to `--settings`.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 1, run after the pilot (#67) merged. This post's rewrite keeps the style file, settings key, and every version number and issue link, and swaps the third-person, contract-heavy prose for a person explaining what we did and why. Judgment calls: "adversarial review" became "a second model's review" everywhere except the quoted AGENTS.md rationale, which must stay verbatim; "the contract" became "the rule" in the body while the title keeps "Output Contract" because the skill links the post by that name; the aphorisms ("The findings were the work. The ELI5 was the interface.", "A default you cannot cheaply reverse is a policy.", "swims against that current") were replaced with the plain fact each stood for. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"
