---
layout: post
title: "ELI5 as the Default Output Contract for Coding Agents"
date: 2026-08-07 09:30:00 -0600
summary: "The most valuable artifact of our agent review rounds was never the findings list - it was the plain-language digest with a recommendation. So we made that the default for every response, as a checked-in output style shared across all our repos."
tags: [ai-agents, claude-code, workflow, communication]
---

## The Problem

As coding agents get stronger, they delegate more. A single session on the [Hello Weather](https://helloweather.com) iOS repo will routinely dispatch an adversarial review round to a second frontier model, fan out subagents per surface, and drain a findings queue - all before the human sees a word.

The raw output of that work is expert-dense by nature: findings lists keyed by internal shorthand, invariant names invented three subagents ago, verdicts that assume you watched the whole process unfold. The human maintainer - the one person who has to actually decide - becomes the bottleneck, rereading output that was technically complete but practically unreadable.

We had already solved this in one narrow place. Our iOS code-review checklist ends with a rule: after the adversarial review round settles, *"digest to the user as an ELI5 with a recommendation."* Not a findings dump - a plain-language explanation a smart person with zero context can follow, ending with what the agent thinks you should do.

That digest turned out to be the most valuable artifact of the entire round. The findings were the work; the ELI5 was the interface.

So when we moved our sessions to a new, more capable model tier - which delegates even more aggressively - the question was obvious: why is ELI5 a special-case rule for one checklist step, instead of the default contract for everything the agent says?

## The Solution

Claude Code has a mechanism built for exactly this: **output styles**. An output style is a Markdown file that modifies the agent's system prompt - it changes how the agent *communicates* without touching what it *does*. Critically for a team, styles can be checked into the repo and activated by a checked-in setting, so every contributor (and every model) gets the same contract.

The division of labor that makes this clean:

- **`AGENTS.md` / `CLAUDE.md`** - project context, conventions, invariants: what the agent should *know and do*
- **Output style** - tone, structure, explanation level: how the agent should *talk to you*

We added the same two files to all three Hello Weather repos (iOS, Android, and the Rails backend).

## Implementation

### The style file

`.claude/output-styles/eli5.md`:

```markdown
---
name: ELI5
description: Plain-language digests - bottom line first, jargon defined, recommendation included
keep-coding-instructions: true
---

Write every user-facing response as an ELI5 digest: plain language a smart reader
with zero context can follow in one pass. Simplify the explanation, never the work -
code, tests, and technical decisions stay precise and idiomatic.

- Lead with the bottom line in one plain sentence: what happened, what you found,
  or what you recommend.
- Use everyday words. When a technical term is unavoidable, define it in-line the
  first time it appears.
- Prefer a short concrete example or analogy over an abstract description.
- Never lean on internal codenames, ticket numbers, or shorthand without saying
  what they are in place.
- When reporting delegated or reviewed work (agent rounds, adversarial reviews),
  digest it the same way: what was checked, what was found, and a clear
  recommendation - not a raw findings dump.
- End multi-step work with a short "what this means" recap, and a recommendation
  whenever a decision is needed.
```

Two details matter here:

**`keep-coding-instructions: true`** keeps the harness's software-engineering instructions intact. The style only layers communication rules on top. Without it you'd be trading code quality for readability, which is the wrong trade.

**"Simplify the explanation, never the work"** is the load-bearing line. ELI5 is an output contract, not a capability downgrade. The code, the tests, and the review rigor are unchanged - only the final rendering to the human changes.

### Activating it

One key in the checked-in `.claude/settings.json` (the value matches the style's `name:` field):

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "outputStyle": "ELI5",
  "permissions": { ... }
}
```

Because both files are committed, the default travels with the repo: every clone, every teammate, every session. Individuals can still override locally in `settings.local.json`, and the style applies regardless of which model is selected - which is the point, since the reason we did this was a model change.

(One version note: the old `/output-style` slash command was removed in Claude Code v2.1.91. Styles themselves are alive and well - manage them via `/config` or the settings files directly.)

### Documenting the decision

A settings key with no rationale is a future mystery, so each repo's `AGENTS.md` gets a short dated section: what the setting is, where the style lives, why it exists ("the same ELI5-with-recommendation shape the adversarial review rounds digest to"), and an explicit exit condition - *drop the setting if it stops earning its keep*. A default you can't cheaply reverse is a policy; this is meant to be an experiment with a paper trail.

## Results

The narrow-case rule became the general contract with nine small file changes across three repos: one style file, one settings key, and one documentation note each.

What we expect from it, based on the review-round experience:

- **Decisions speed up.** The bottom-line-first rule means the human reads one sentence before deciding whether to read ten.
- **Delegated work stays legible.** The more a session fans out, the more the final digest matters - the style makes the digest mandatory rather than checklist-dependent.
- **The contract survives model churn.** Styles bind to the repo, not the model. Whatever tier runs next quarter inherits the same interface.

## Lessons Learned

- **Watch what you actually read.** The findings lists were skimmed; the ELI5 digests were read. The artifact you consistently reach for is telling you what the default output should be.
- **Put communication contracts in the repo, not in memory.** A style file and a settings key are versioned, reviewable, shared across the team, and portable across tools. An agent's private memory is none of those things.
- **Separate the work from the rendering.** `keep-coding-instructions: true` plus "never simplify the work" keeps the contract honest - you're changing the interface, not the engineering.
- **Date the decision and name the exit.** "Adopted 2026-08-07; drop it if it stops earning its keep" turns a default into a reversible experiment.

---

## How This Post Was Made

**Prompt 1:** "see recent claude opus48 delegating, I think we might want to default to ELI5 output in Fable for now, too. how do we do that? we may want to update all helloweather repos, too"

**Prompt 2:** "and add a blog post about this (trevorturk.github.io repo) and also review recent work with date formatting extensions etc"

Generated by Claude using the blog-post-generator skill, in the same session that made the config changes it describes - the mechanism was verified against the current Claude Code docs before writing.
