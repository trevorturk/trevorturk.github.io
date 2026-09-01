---
layout: post
title: "Skills and Scripts: A Pattern for LLM Workflows"
date: 2026-02-27 09:00:00 -0600
summary: "How pairing SKILL.md files with bin/ scripts creates powerful, reusable LLM workflows, and why the data belongs in a database the scripts query rather than in the skill."
tags: [patterns, claude, automation]
---

## The Problem

Our first skill paired with a script managed App Store pricing across 175 territories, each with its own price points, currency rules, and constraints. A fresh conversation knows none of that. Every time you start one, something has to supply the knowledge again.

Two common approaches:

1. **Documentation** - Write detailed docs and hope the LLM reads them
2. **Automation** - Write scripts that handle everything, no LLM needed

Documentation alone lacks actionability. Full automation lacks flexibility. Neither covers the case where you want the LLM to understand *and* be able to act.

## The Solution: Skills + Scripts

The pattern pairs two things:

- **Skills** = `SKILL.md` files that teach Claude specific workflows
- **Scripts** = `bin/` commands that automate operations

Skills provide context and guidance. Scripts provide capability and safety. Together, the LLM knows what to do and has the tools to do it.

## Implementation

### Skills: Teaching the LLM

A skill file lives in `.claude/skills/[name]/SKILL.md` (we have since moved ours to `.agents/skills/`, which Codex reads natively, with `.claude/skills` left as a symlink so both tools find the same files):

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

Claude Code discovers these from the filesystem. You can invoke one by name (`/skill-name`), but most of the time the model loads it on its own when a task matches the description line, so that line does the work of discovery.

### Scripts: Safe Automation

Scripts live in `bin/` and do the actual work:

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
- **Safe by default** - Preview destructive actions, and only execute them behind an explicit `--apply` flag or after a `--dry-run` pass
- **Idempotent** - Running twice doesn't break things

### The Pairing

The skill names the scripts and says which ones are safe:

```markdown
## Commands Available

- `bin/check-status` - View current state (safe, read-only)
- `bin/apply-changes` - Make modifications (preview by default; `--apply` executes)
```

Then it says when to use each:

```markdown
## Workflow

1. Run `bin/check-status` to see current state
2. Analyze the output
3. If changes needed, run `bin/apply-changes`, then `bin/apply-changes --apply`
4. Verify with `bin/check-status` again
```

## Why This Works

The skill tells the LLM what is possible and where the guardrails are. The scripts handle the fiddly details, so the human gets help without a full context dump every conversation. The team gets workflows that are documented *and* executable, captured in version-controlled files, so a new team member, human or AI, can onboard from them. Patterns compound as you build more skills.

## Taming Context Exhaustion

The pricing skill taught us this the hard way. LLMs have limited context windows, and filling them with raw data makes everything worse: slower responses, higher costs, and a model that loses track of what was said earlier.

We first stored the price tiers for all 175 territories in YAML files the skill referenced, one file of about 700 lines per product, plus more for PPP ratios and exchange rates. Every conversation choked on thousands of lines of pricing data before we could ask a question.

The fix, eleven days later, was moving the reference data into a SQLite file and building the scripts on ActiveRecord models that query it (simplified):

```ruby
# Instead of loading every territory's tiers into context...
class PricePoint < AppstoreRecord # ActiveRecord::Base over db/appstore.sqlite3
  scope :for_territory, ->(territory) { where(territory: territory).order(:price) }
end

# ...scripts return just what's needed
bin/appstore ppp list
bin/appstore validate --verbose
```

Now the LLM never sees the raw data. It calls scripts that return structured, filtered results, so the context stays lean and responses stay fast. Put data in databases, not documentation. Let scripts do the heavy lifting, and return only what the current question needs.

## Example: Capacity Monitoring

A real skill, for monitoring Heroku dyno capacity, trimmed to its shape:

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

Every command is read-only; scaling itself is still a human running the Heroku CLI. The LLM knows when to use the skill, what commands exist, and where the safety boundaries are.

## Getting Started

1. **Identify a workflow** you do repeatedly
2. **Write the skill** - What does the LLM need to know?
3. **Build the scripts** - What actions should be automated?
4. **Document the pairing** - How do they work together?

Start simple. One skill, one or two scripts. Iterate as you learn what works.

## Lessons Learned

- **Skills are cheaper than you think** - A few paragraphs of context go a long way
- **Let the LLM orchestrate and the scripts validate** - The model decides what to run; the script checks inputs and guards destructive actions
- **Start with read-only** - Build trust before enabling writes

---

## How This Post Was Made

**Prompt 1:** "Create a 'Skills and Scripts' post explaining the pattern we've developed. Skills = SKILL.md files that teach Claude workflows, Scripts = bin/ commands that automate operations. Explain why pairing them works well."

**Prompt 2:** "Add to the skills and scripts post the idea that the scripts (CLIs) help reduce context exhaustion. We learned this the hard way with our first skill+script which was about appstore pricing. Initially, we had the 175 app store territories with each of their price points in yaml. You could hardly talk to claude because the context window would explode. So we moved the data into sqlite files and created scripts with active record models to query the data in structured ways."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Initial draft from first prompt; "Taming Context Exhaustion" section added via follow-up prompt based on real experience with the App Store pricing skill.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the pricing skill that prompted it, Why This Works is one paragraph instead of three lists, the Getting Started steps are kept as they were, and Lessons Learned drops the two bullets that restated the Solution. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The pricing skill was the first skill paired with a script, not the first skill (skills predate it by three months), so the opening says so. Skills now live in `.agents/skills/` with `.claude/skills` as a symlink, and Claude Code loads a skill when a task matches its description rather than only through a `/skill-name` command; both are reworded. The real scripts guard destructive actions with preview-by-default and `--apply` or `--dry-run` flags, not confirmation prompts, so the safety bullets and pairing example now say that. The SQLite excerpt showed a `Territory.where(currency:)` query that never existed; it is replaced with the real `PricePoint.for_territory` scope over an ActiveRecord base class and real `bin/appstore` commands, and the YAML-to-SQLite move is dated (eleven days after the skill landed). The capacity example was invented; it is replaced with the real `heroku-capacity` skill's description, scope, commands, and guardrails, whose commands are all read-only.
