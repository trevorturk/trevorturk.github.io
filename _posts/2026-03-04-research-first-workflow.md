---
layout: post
title: "Research First: Using Claude Web Before Claude Code"
date: 2026-03-04 10:00:00 -0600
summary: "A workflow for better decisions: deep research on Claude.ai, then implementation handoff to Claude Code with full codebase context."
tags: [claude, workflow, patterns]
---

## The Problem

Claude Code is eager. Give it a task like "add authentication to my Rails app" and it will immediately start implementing something. That's great for well-defined tasks, but for decisions that require research - which gem to use, which architecture pattern, which third-party service - eagerness becomes a liability.

You end up with an implementation of *a* solution before you've evaluated whether it's the *right* solution.

## The Insight

Claude's web interface and Claude Code have different strengths:

| Claude.ai (Web) | Claude Code |
|-----------------|-------------|
| Deep research and analysis | Codebase-aware implementation |
| Extended thinking for complex reasoning | Fast iteration on changes |
| Compare multiple options | File editing and testing |
| Phone notifications when done | Real-time interaction |
| Non-blocking - do other work | Blocking - watching it work |

The web interface excels at the "what should we do?" phase. Claude Code excels at the "now do it" phase.

## The Workflow

### Step 1: Research on Claude.ai

Open [claude.ai](https://claude.ai) and enable extended thinking for complex decisions. Ask research questions:

> "I need to add authentication to a Rails 8 app. Compare the major options (Devise, Rodauth, Clearance, roll-your-own with has_secure_password). Consider: maintenance activity, Rails 8 compatibility, flexibility, learning curve, and whether I need OAuth support later. Recommend an approach."

Let it think. This takes time - that's the point. Extended thinking produces more thorough analysis than quick responses.

### Step 2: Don't Block

While research runs, do other work. Claude.ai sends push notifications to your phone when the response is ready. This is async research, not a synchronous conversation.

For big decisions (architecture, major dependencies, technology choices), this matters. You want comprehensive analysis, not the first answer that comes to mind.

### Step 3: Review the Report

The web response gives you a full analysis: pros/cons, trade-offs, recommendations with reasoning. Read it. Disagree with parts? Ask follow-up questions. Refine the recommendation until you're confident in the direction.

### Step 4: Request Implementation Instructions

Once you've decided on an approach, ask for a handoff document:

> "I've decided to use Rodauth. Create two things:
> 1. A full report I can reference later (architecture decisions, why Rodauth, key configuration choices)
> 2. A copy/paste-able markdown prompt for Claude Code with step-by-step implementation instructions. Assume Claude Code has full access to my codebase but doesn't know anything about this research conversation."

The second artifact is key - it bridges the research phase to the implementation phase.

### Step 5: Hand Off to Claude Code

Open Claude Code in your project directory. Paste the implementation instructions. Now Claude Code does what it's good at: reading your existing code, understanding your patterns, and implementing the solution in a way that fits your codebase.

The decision was made thoughtfully. The implementation is codebase-aware.

## Why This Works

### Research Quality

Extended thinking on the web produces better research than quick Claude Code responses. The model takes time to:
- Consider multiple options
- Weigh trade-offs systematically
- Check for edge cases and gotchas
- Provide nuanced recommendations

### Context Separation

Research doesn't need your codebase. "Which auth gem should I use?" is independent of your specific file structure. Keeping research separate prevents the model from anchoring on implementation details before the decision is made.

### Non-Blocking Work

Research takes minutes. Watching Claude think isn't productive time. Phone notifications let you context-switch to other work and come back when the analysis is ready.

### Clean Handoff

The implementation prompt captures the decision and reasoning. Claude Code gets clear instructions without needing to re-derive the research. Your codebase context combines with pre-researched decisions.

## Example: Choosing an Authentication Gem

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

When requesting handoff documents, ask for two things:

### 1. Reference Report
Full analysis you can revisit later. Include:
- Options considered and why each was rejected/selected
- Key configuration decisions and reasoning
- Known limitations and future considerations
- Links to documentation

### 2. Implementation Prompt
Copy/paste instructions for Claude Code. Include:
- Clear statement of what to implement
- Specific technical choices already made
- Step-by-step implementation order
- Explicit instructions (don't let Claude Code re-research)

## Results

With this workflow:
- Better decisions from deeper research
- No blocking on research time
- Clean handoff between phases
- Implementation that fits your codebase
- Documented reasoning for future reference

## Lessons Learned

- **Eagerness isn't always good** - Claude Code's speed is a liability for decisions that need thought
- **Extended thinking matters** - Complex research benefits from the web interface's deeper analysis
- **Async research scales** - Phone notifications let you run multiple research threads
- **Handoff documents work** - The implementation prompt bridges research to action
- **Context separation helps** - Research without codebase anchoring produces better options analysis

---

## How This Post Was Made

**Prompt:** "I'd like another post explaining the benefits of using claude's website for research. Use Claude on the web for research and then pass the results into Claude code. Claude code is too eager. Research on the web does a much more comprehensive job. you can enable extended thinking and research on the website. you can get notifications on your phone when it's done, and not block other work. this works well for researching stuff like which rubygem to use for auth etc etc. claude code is too eager to dive in sometimes. you can also ask claude.ai to provide a full artifact/report, and also another copy/paste-able markdown report with instructions for claude code to use (in tandem with having full access to your codebase etc) to procced with implementation. create a pr off latest main."

Generated by Claude (Opus 4.5) using the blog-post-generator skill.
