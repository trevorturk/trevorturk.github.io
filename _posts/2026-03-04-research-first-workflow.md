---
layout: post
title: "Research First: Using Claude Web Before Claude Code"
date: 2026-03-04 10:00:00 -0600
summary: "A workflow for better decisions: deep research on Claude.ai, then implementation handoff to Claude Code with full codebase context."
tags: [claude, workflow, patterns]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Ask Claude Code to "add authentication to my Rails app" and it starts implementing Devise before anyone has asked whether Devise is the right gem. For a well-defined task, that speed is the point. For a decision that needs research, which gem, which architecture pattern, which third-party service, it means an implementation of *a* solution exists before anyone has evaluated whether it is the *right* one.

## The Insight

Claude's web interface and Claude Code have different strengths:

| Claude.ai (Web) | Claude Code |
|-----------------|-------------|
| Deep research and analysis | Codebase-aware implementation |
| Extended thinking for complex reasoning | Fast iteration on changes |
| Compare multiple options | File editing and testing |
| Phone notifications when done | Real-time interaction |
| Non-blocking - do other work | Blocking - watching it work |

The web interface is for the "what should we do?" phase. Claude Code is for the "now do it" phase.

## The Workflow

### Step 1: Research on Claude.ai

Open [claude.ai](https://claude.ai), enable extended thinking, and ask the research question with the criteria spelled out:

> "I need to add authentication to a Rails 8 app. Compare the major options (Devise, Rodauth, Clearance, roll-your-own with has_secure_password). Consider: maintenance activity, Rails 8 compatibility, flexibility, learning curve, and whether I need OAuth support later. Recommend an approach."

### Step 2: Don't Block

Go do other work. Claude.ai sends a push notification to your phone when the response is ready.

### Step 3: Review the Report

The response lays out pros and cons, trade-offs, and a recommendation with reasoning. Disagree with parts? Ask follow-up questions until you are confident in the direction.

### Step 4: Request Implementation Instructions

Once you have decided on an approach, ask for a handoff document:

> "I've decided to use Rodauth. Create two things:
> 1. A full report I can reference later (architecture decisions, why Rodauth, key configuration choices)
> 2. A copy/paste-able markdown prompt for Claude Code with step-by-step implementation instructions. Assume Claude Code has full access to my codebase but doesn't know anything about this research conversation."

The second item is the bridge from the research phase to the implementation phase.

### Step 5: Hand Off to Claude Code

Open Claude Code in your project directory and paste the implementation instructions. Claude Code reads your existing code, picks up your patterns, and implements the chosen approach in a way that fits.

## Why This Works

### Research Quality

Extended thinking spends minutes where Claude Code spends seconds. That time goes into considering several options, weighing trade-offs systematically, and checking for edge cases and gotchas before recommending anything.

### Context Separation

"Which auth gem should I use?" does not depend on your file structure. Keeping the research away from the codebase stops the model from anchoring on implementation details before the decision is made.

### Non-Blocking Work

Watching a model think is not productive time. The phone notification turns a minutes-long wait into a context switch, and it lets several research threads run at once.

### Clean Handoff

The implementation prompt carries the decision and the reasoning behind it. Claude Code does not re-derive the research. It combines the decisions already made with the codebase context the web session never had.

## Example: Choosing an Authentication Gem

The example is hypothetical. The Rails app behind this blog never ran this comparison: its accounts use Rails' built-in `has_secure_password`, chosen in 2022, and it has never used Devise or Rodauth. The gem names stand in for any decision with the same shape.

**Bad workflow:**
```
[Claude Code]
> Add user authentication to this Rails app

*Claude Code immediately starts implementing Devise*
*You wonder if Devise was the right choice*
*Too late, it's already half-implemented*
```

**Good workflow:**
```
[Claude.ai with extended thinking]
> Compare Rails authentication options for a new Rails 8 app...

*10 minutes later, phone notification*
*Read analysis: Rodauth recommended for flexibility, active maintenance*
*Ask follow-up about OAuth support*
*Confirm decision*

> Create implementation instructions for Claude Code...

[Claude Code]
> *paste implementation prompt*

*Claude Code implements Rodauth following the researched approach*
*Fits your existing patterns because it can see your code*
```

## When to Use This Workflow

**Use research-first for:**
- Major dependency decisions (auth, payments, background jobs)
- Architecture choices (monolith vs services, database design)
- Technology selection (which API, which service, which pattern)
- Anything where "undo" is expensive

**Skip research for:**
- Well-defined implementation tasks
- Bug fixes with clear solutions
- Features where the approach is obvious
- Small, reversible changes

## The Artifacts

Ask for two handoff documents, because they serve different readers.

### 1. Reference Report

The full analysis, for you to revisit later. Include:
- Options considered and why each was rejected/selected
- Key configuration decisions and reasoning
- Known limitations and future considerations
- Links to documentation

### 2. Implementation Prompt

Copy/paste instructions, for Claude Code. Include:
- Clear statement of what to implement
- Specific technical choices already made
- Step-by-step implementation order
- Explicit instructions (don't let Claude Code re-research)

## Results

Nothing here was measured. What changed is where the decision gets made: on Claude.ai, with the reasoning written down, before Claude Code opens a file. The cost is the wait, minutes per question instead of seconds, plus one extra prompt to produce the two handoff documents. The reference report is the lasting artifact, because the reasoning survives after the implementation is done.

## Lessons Learned

- **Match eagerness to reversibility.** Let Claude Code dive in when undo is cheap. When undo is expensive, decide first, somewhere it cannot start typing.
- **Write the handoff for an implementer who knows nothing about the research.** State what was chosen and why, so it does not reopen the question.
- **Ask for the reference report in the same breath as the prompt.** The prompt is consumed once. The report is what you reread when the decision is questioned later.
