---
layout: post
title: "ELI5 as the Default Output Contract for Coding Agents"
date: 2026-08-07 09:30:00 -0600
summary: "The plain-language digest with a recommendation was the part of our agent review rounds we actually read, so we made it the default for every response, as a checked-in output style shared across our repos."
tags: [ai-agents, claude-code, workflow, communication]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
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
