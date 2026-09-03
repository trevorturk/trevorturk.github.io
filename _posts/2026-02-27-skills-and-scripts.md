---
layout: post
title: "Skills and Scripts: A Pattern for LLM Workflows"
date: 2026-02-27 09:00:00 -0600
summary: "We pair a SKILL.md file that teaches the model a workflow with bin/ scripts that do the work, and we keep the data in a database the scripts query instead of in the skill."
tags: [patterns, claude, automation]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

The first skill we paired with a script managed App Store pricing across 175 territories. Each territory has its own price points and currency rules. A new conversation with the model starts out knowing none of that, so every time we open one, something has to teach it again.

There are two common ways to do that:

1. **Documentation** - Write it all down and hope the model reads it
2. **Automation** - Write scripts that do everything, with no model in the loop

Docs on their own can't do anything. Scripts on their own can't adapt. We wanted the model to understand the job *and* be able to act on it.

## The Solution: Skills + Scripts

We pair two things:

- **Skills** = `SKILL.md` files that teach Claude a workflow
- **Scripts** = commands in `bin/` that do the work

The skill tells the model what to do. The scripts give it a safe way to do it.

## Implementation

### Skills: Teaching the Model

A skill file lives at `.claude/skills/[name]/SKILL.md`. We've since moved ours to `.agents/skills/`, which Codex reads on its own, and left `.claude/skills` as a symlink so both tools find the same files. Here's the shape of one:

```markdown
---
name: skill-name
description: One-line description for discovery
---

# Skill Name

## When to Use
Trigger phrases and scenarios

## Workflow
Step-by-step process

## Commands Available
What scripts/tools exist

## Important Notes
Guardrails and warnings
```

Claude Code finds these files on disk. You can call one by name with `/skill-name`, but most of the time the model loads it on its own because a task matched the description line. That line is the one to get right.

### Scripts: Safe Automation

Scripts live in `bin/` and do the work:

```bash
#!/bin/bash
# bin/do-something

set -e  # Exit on error

# Validate inputs
if [ -z "$1" ]; then
  echo "Usage: bin/do-something <arg>"
  exit 1
fi

# Do the work
# ...
```

A script should be:

- **Self-documenting** - It prints a clear usage message when called wrong
- **Safe by default** - It previews anything destructive, and only runs it with an explicit `--apply` flag or after a `--dry-run` pass
- **Idempotent** - Running it twice gives the same result as running it once

### The Pairing

The skill lists the scripts and says which ones are safe:

```markdown
## Commands Available

- `bin/check-status` - View current state (safe, read-only)
- `bin/apply-changes` - Make modifications (preview by default; `--apply` executes)
```

Then it says when to run each one:

```markdown
## Workflow

1. Run `bin/check-status` to see current state
2. Analyze the output
3. If changes needed, run `bin/apply-changes`, then `bin/apply-changes --apply`
4. Verify with `bin/check-status` again
```

## Why This Works

The skill tells the model what it can do and where the limits are. The scripts handle the fiddly details, so we get help without pasting the whole background into every conversation. Both live in version control, so the workflow is documented and runnable in the same place, and a new team member, human or AI, can learn it from the files. Each new skill builds on the ones before it.

## Taming Context Exhaustion

We learned this one the hard way with the pricing skill. A model can only hold so much text in one conversation (its context window). Fill that with raw data and everything gets worse: answers slow down, costs go up, and the model loses track of what was said earlier.

At first we kept the price tiers for all 175 territories in YAML files the skill pointed at. That was about 700 lines per product, plus more files for purchasing-power (PPP) ratios and exchange rates. Every conversation read thousands of lines of pricing data before we could ask our first question.

Eleven days later we moved the data into a SQLite file and rebuilt the scripts on ActiveRecord models that query it. Simplified:

```ruby
# Instead of loading every territory's tiers into context...
class PricePoint < AppstoreRecord # ActiveRecord::Base over db/appstore.sqlite3
  scope :for_territory, ->(territory) { where(territory: territory).order(:price) }
end

# ...scripts return just what's needed
bin/appstore ppp list
bin/appstore validate --verbose
```

Now the model never sees the raw data. It calls a script, and the script returns only the rows the question needs, so the conversation stays small and answers stay fast. Put data in a database, not in the skill.

## Example: Capacity Monitoring

Here's a real skill, for watching our Heroku dyno count (the server processes we pay for), trimmed to its shape:

```markdown
---
name: Heroku Capacity
description: Heroku dyno capacity workflow for traffic spikes using bin/heroku status, capture, and recommend. Use when checking scale after traffic changes, APNS spike windows, or Heroku error/latency regressions.
---

# Heroku Capacity Skill

## Scope
- validating whether current dyno count is sufficient
- investigating traffic spike behavior (especially APNS cycles)
- interpreting `HOLD`, `SCALE_UP`, `PROBE_DOWN`, and manual probe decisions

## Primary Commands

- `bin/heroku status` - Quick status
- `bin/heroku check --json` - One-command operational check
- `bin/heroku capture --window spike_30 --json` then `bin/heroku recommend --json` - Manual capture flow

## Operational Guardrails

- Do not change dyno formation from a single short capture.
- Keep `formation.yml` current after scaling changes.
- Store and reference capture JSON paths in decisions/PRs.
- Treat wrapper recommendations as decision support.
```

Every command here is read-only. Scaling is still a person running the Heroku CLI. The model knows when to use the skill, which commands exist, and where it has to stop.

## Getting Started

1. **Pick a workflow** you repeat
2. **Write the skill** - What does the model need to know?
3. **Build the scripts** - Which actions should be automated?
4. **Connect them** - The skill lists the scripts and says when to run each

Start with one skill and one or two scripts, and add more as you learn what works.

## Lessons Learned

- **A skill is cheap** - A few paragraphs of context is usually enough
- **The model decides, the script checks** - The model picks what to run; the script validates inputs and guards anything destructive
- **Start read-only** - Add writes once the read-only version has earned some trust
