---
layout: post
title: "ELI5 as the Default Output Contract for Coding Agents"
date: 2026-08-07 09:30:00 -0600
summary: "The most valuable artifact of our agent review rounds was never the findings list - it was the plain-language digest with a recommendation. So we made that the default for every response, as a checked-in output style shared across all our repos."
tags: [ai-agents, claude-code, workflow, communication]
---

## The Problem

One item in our iOS code-review checklist carries a rule. After the adversarial review round settles, *"digest to the user as an ELI5 with a recommendation."* Not a findings dump: a plain-language explanation a smart person with zero context can follow, ending with what the agent thinks you should do. That digest turned out to be the most valuable artifact of the whole round. The findings were the work. The ELI5 was the interface.

Then we moved our [Hello Weather](https://helloweather.com) sessions to a new, more capable model tier, and it delegates more aggressively than the last one. A single session on the iOS repo will dispatch an adversarial review round to a second frontier model, fan out subagents per surface, and drain a findings queue before the human sees a word. The raw output is expert-dense: findings lists keyed by internal shorthand, invariant names invented three subagents ago, verdicts that assume you watched the whole process unfold. The one person who has to decide becomes the bottleneck, rereading output that was technically complete and practically unreadable.

So why was ELI5 a special-case rule for one checklist step, instead of the default contract for everything the agent says?

## The Solution

Claude Code has a mechanism built for this: output styles. An output style is a Markdown file that modifies the agent's system prompt. It changes how the agent *communicates* without touching what it *does*. Styles can be checked into the repo and activated by a checked-in setting, so every contributor and every model gets the same contract.

The division of labor:

- **`AGENTS.md` / `CLAUDE.md`** - project context, conventions, invariants: what the agent should *know and do*
- **Output style** - tone, structure, explanation level: how the agent should *talk to you*

The same two files went into all three Hello Weather repos (iOS, Android, and the Rails backend).

## Implementation

### The style file

`.claude/output-styles/ELI5.md`. The filename deliberately matches the style's `name:`, for a reason covered below:

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

Two lines in that file do most of the work. `keep-coding-instructions: true` keeps the harness's software-engineering instructions intact, so the style only layers communication rules on top. Without it you would be trading code quality for readability, which is the wrong trade. "Simplify the explanation, never the work" makes the same promise in the body: ELI5 is an output contract, not a capability downgrade. The code, the tests, and the review rigor are unchanged. Only the final rendering to the human changes.

### Activating it

One key in the checked-in `.claude/settings.json`. The value matches the style's `name:` field, case-sensitively:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "outputStyle": "ELI5",
  "permissions": { ... }
}
```

Because both files are committed, the default travels with the repo: every clone, every teammate, every session. Individuals can still override locally in `settings.local.json`. The style applies regardless of which model is selected, which matters here because a model change is what prompted it.

One version note: the old `/output-style` slash command was deprecated in Claude Code v2.1.73 (March 2026) and is gone from current builds. Styles themselves are alive and well. Manage them via `/config` or the settings files directly.

### Documenting the decision

A settings key with no rationale is a future mystery, so each repo's `AGENTS.md` gets a short dated section: what the setting is, where the style lives, why it exists ("the same ELI5-with-recommendation shape the adversarial review rounds digest to"), and an explicit exit condition, *drop it if it stops earning its keep*. A default you cannot cheaply reverse is a policy. This one is meant to be an experiment with a paper trail. (On 2026-08-13 the repos went tool-agnostic and split the note: `CLAUDE.md` now holds the setting and file path, and `AGENTS.md` states the contract as prose any tool can follow.)

### Attack your own config before trusting it

Before calling it done, we dispatched one more round of three parallel agents. One verified the official docs and full changelog history. One swept community field reports (GitHub issues, HN, practitioner blogs). One adversarial reviewer was instructed to refute the shipped files, and went as far as reading the style loader in the shipped Claude Code binary. The mechanism held: output styles are actively maintained, and their one deprecation (v2.0.30, late October 2025) was reversed within days on community feedback. But the round produced four one-line hardenings, all folded into the version shown above:

- **The filename matches the style's `name:` case-sensitively** (`ELI5.md`, not `eli5.md`). An open issue ([anthropics/claude-code#47482](https://github.com/anthropics/claude-code/issues/47482)) reports styles that show as active in the statusline while their body is silently never injected, with a name/filename case mismatch isolated as the trigger. Our binary read suggested current versions resolve it correctly anyway, and matching them is free insurance.
- **The unconditional "end with a recap" rule is gone.** Claude Code ships an always-on anti-verbosity section in its system prompt, and a style instruction that fights it every turn is the documented adherence-killer. The recommendation is now gated on "whenever a decision is needed."
- **Codenames get glossed on first mention only, and exact symbol names are explicitly preserved.** The original wording demanded re-explaining every internal name forever, padding that pressures the model toward vague paraphrase when the precise symbol is the point.
- **"Every user-facing response" became "every response to the user."** In our repos "user-facing" is a term of art for shipped product copy, which has its own stricter voice rules. The style must never read as license to rewrite that.

The round also set two expectations we chose to accept rather than fix. Custom styles got no per-turn adherence reminder at the time. The binary only reinforced built-in styles, despite docs saying otherwise, so we expected some drift back toward default terseness late in long sessions, and judged the experiment on early-session behavior. Claude Code v2.1.238 (2026-08-20) fixed custom styles drifting back to the default voice mid-session. And a committed `outputStyle` also applies to headless runs in the repo. The request for a per-run opt-out ([#81334](https://github.com/anthropics/claude-code/issues/81334)) was closed on 2026-08-17 as already possible: `claude -p --settings '{"outputStyle":"default"}'` overrides the committed key for one run.

One sentiment note: teams building custom styles in 2026 overwhelmingly build them to make the agent *terser*. A plain-language default swims against that current and against the server's own conciseness instructions, so it cannot afford fights it will lose.

## Results

The narrow-case rule became the general contract with nine small file changes across three repos: one style file, one settings key, and one documentation note each. Nothing is measured yet. What we expect, based on the review-round experience:

- **Decisions speed up.** Bottom line first means the human reads one sentence before deciding whether to read ten.
- **Delegated work stays legible.** The more a session fans out, the more the final digest matters, and the style makes it mandatory rather than checklist-dependent.
- **The contract survives model churn.** Styles bind to the repo, not the model.

The accepted cost was the two limits above: some drift back toward terseness late in long sessions, and no headless opt-out. Both have since been lifted, as noted there.

## Lessons Learned

- **Watch what you actually read.** The findings lists were skimmed. The ELI5 digests were read. The artifact you consistently reach for is telling you what the default output should be.
- **Put communication contracts in the repo, not in memory.** A style file and a settings key are versioned, reviewable, and portable across tools. An agent's private memory is none of those.
- **Don't write instructions that fight the system prompt.** An unconditional "always recap" mandate loses to the built-in anti-verbosity instructions eventually. A conditional one never has to win the argument.
- **Check the docs against the binary.** The docs said every style gets per-turn adherence reminders. The style loader said only built-ins do. Minutes of adversarial review caught an assumption we would otherwise have relied on, and the gap was real until v2.1.238 closed it.

---

## How This Post Was Made

**Prompt 1:** "see recent claude opus48 delegating, I think we might want to default to ELI5 output in Fable for now, too. how do we do that? we may want to update all helloweather repos, too"

**Prompt 2:** "and add a blog post about this (trevorturk.github.io repo) and also review recent work with date formatting extensions etc"

**Prompt 3:** "dispatch research / adversarial review with opus48 to determine if there's anthropic docs, user sentiment etc about this which might inform our choices, it's ok if this is all correct, but let's dispatch one more round of research, you digest and report back"

**Prompt 4:** "agreed with your recommendation, but also update the blog post pr (we're working on 4 prs in this session)"

Generated by Claude using the blog-post-generator skill, in the same session that made the config changes it describes. The mechanism was verified against the current Claude Code docs before writing; the "Attack your own config" section documents the three-agent research round that then hardened both the config and this post.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the checklist rule and the model change that prompted the work, Results states expectations as expectations plus the accepted cost, and Lessons Learned dropped the two bullets that repeated body sections. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The style file, settings key, nine-file count, and checklist quote match the repos; the style excerpt now carries the file's em dashes, the checklist rule is described as one item rather than the last one, and the exit-condition quote uses the current wording plus a note that the AGENTS.md section split into CLAUDE.md and AGENTS.md on 2026-08-13. The `/output-style` version note now cites the changelog's v2.1.73 deprecation, the styles deprecation is dated to v2.0.30 in late October 2025, and the two accepted limits are marked as since lifted: the drift fix landed in v2.1.238 and issue #81334 closed pointing to `--settings`.
