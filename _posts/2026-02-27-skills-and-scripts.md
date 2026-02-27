---
layout: post
title: "Skills and Scripts: A Pattern for LLM Workflows"
date: 2026-02-27
summary: "How pairing SKILL.md files with bin/ scripts creates powerful, reusable LLM workflows."
tags: [patterns, claude, automation]
image: /images/skills-and-scripts.png
---

## The Problem

LLMs are powerful, but they need context. Every time you start a new conversation, you're starting from scratch. How do you give an LLM the knowledge it needs to help with your specific workflows?

Two common approaches:

1. **Documentation** - Write detailed docs and hope the LLM reads them
2. **Automation** - Write scripts that handle everything, no LLM needed

Both have limitations. Documentation alone lacks actionability. Full automation lacks flexibility. What if you want the LLM to understand *and* be able to act?

## The Solution: Skills + Scripts

The pattern we've developed pairs two things:

- **Skills** = `SKILL.md` files that teach Claude specific workflows
- **Scripts** = `bin/` commands that automate operations

Skills provide context and guidance. Scripts provide capability and safety. Together, they create workflows where the LLM understands what to do *and* has the tools to do it.

## Implementation

### Skills: Teaching the LLM

A skill file lives in `.claude/skills/[name]/SKILL.md`. It contains:

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

Claude Code automatically loads these skills, making them available as `/skill-name` commands.

### Scripts: Safe Automation

Scripts live in `bin/` and handle the actual operations:

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

Scripts should be:

- **Self-documenting** - Clear usage messages
- **Safe by default** - Require confirmation for destructive actions
- **Idempotent** - Running twice doesn't break things

### The Pairing

The skill references the scripts:

```markdown
## Commands Available

- `bin/check-status` - View current state (safe, read-only)
- `bin/apply-changes` - Make modifications (requires confirmation)
```

And provides guidance on when to use each:

```markdown
## Workflow

1. Run `bin/check-status` to see current state
2. Analyze the output
3. If changes needed, run `bin/apply-changes`
4. Verify with `bin/check-status` again
```

## Why This Works

### For the LLM

- Clear context about what's possible
- Guardrails built into the workflow
- Scripts handle the fiddly details

### For the Human

- LLM can help without needing full context dump every time
- Scripts provide safety rails
- Workflows are documented *and* executable

### For the Team

- Knowledge captured in version-controlled files
- New team members (human or AI) can onboard quickly
- Patterns compound as you build more skills

## Taming Context Exhaustion

Here's something we learned the hard way: LLMs have limited context windows, and bloating them with raw data makes everything worse. Slower responses, higher costs, and eventually the model just... forgets things.

Our first skill was for managing App Store pricing across 175 territories. Each territory has its own price points, currency rules, and constraints. Initially, we stored all this in YAML files that the skill would reference. The result? Context window explosion. Every conversation would choke on thousands of lines of pricing data before we could even ask a question.

The fix was moving the data into SQLite and building scripts with proper data models to query it:

```ruby
# Instead of loading 175 territories into context...
Territory.where(currency: "EUR").map(&:price_points)

# Scripts return just what's needed
bin/pricing territories --currency EUR
```

Now the LLM never sees the raw data. It calls scripts that return structured, filtered results. The context stays lean, responses stay fast, and the model can actually focus on the task.

**The pattern:** Put data in databases, not documentation. Let scripts do the heavy lifting. Return only what's needed for the current question.

## Example: Capacity Monitoring

Here's a real example - a skill for monitoring infrastructure capacity:

```markdown
---
name: capacity-check
description: Monitor and adjust infrastructure capacity
---

# Capacity Check

## When to Use
- "Check current capacity"
- "Are we ready for the traffic spike?"
- "Scale up for the event"

## Commands

- `bin/capacity status` - Current state (read-only)
- `bin/capacity scale [n]` - Adjust capacity (requires confirmation)

## Guardrails

- Never scale below minimum (defined in config)
- Always verify before scaling down
- Log all changes for audit
```

The LLM knows when to use it, what commands exist, and what the safety boundaries are.

## Getting Started

1. **Identify a workflow** you do repeatedly
2. **Write the skill** - What does the LLM need to know?
3. **Build the scripts** - What actions should be automated?
4. **Document the pairing** - How do they work together?

Start simple. One skill, one or two scripts. Iterate as you learn what works.

## Lessons Learned

- **Skills are cheaper than you think** - A few paragraphs of context go a long way
- **Scripts provide safety** - Let the LLM orchestrate, scripts validate
- **The pairing is key** - Neither alone is as powerful as both together
- **Start with read-only** - Build trust before enabling writes
