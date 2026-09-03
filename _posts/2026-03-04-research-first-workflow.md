---
layout: post
title: "Research First: Using Claude Web Before Claude Code"
date: 2026-03-04 10:00:00 -0600
summary: "Decide on Claude.ai first, with extended thinking, then paste a handoff prompt into Claude Code, which can see the codebase."
tags: [claude, workflow, patterns]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Ask Claude Code to "add authentication to my Rails app" and it starts installing Devise before anyone has asked whether Devise is the right gem. That speed is what you want for a well-defined task. It's a problem when the real task is a decision, like which gem to use. By the time you've weighed the options, half of one is already built.

## The Insight

Claude's web interface and Claude Code are good at different things:

| Claude.ai (Web) | Claude Code |
|-----------------|-------------|
| Deep research and analysis | Codebase-aware implementation |
| Extended thinking for complex reasoning | Fast iteration on changes |
| Compare multiple options | File editing and testing |
| Phone notifications when done | Real-time interaction |
| Non-blocking - do other work | Blocking - watching it work |

Use the web interface for the "what should we do?" phase. Use Claude Code for the "now do it" phase.

## The Workflow

### Step 1: Research on Claude.ai

Open [claude.ai](https://claude.ai), turn on extended thinking, and ask the question with your criteria spelled out:

> "I need to add authentication to a Rails 8 app. Compare the major options (Devise, Rodauth, Clearance, roll-your-own with has_secure_password). Consider: maintenance activity, Rails 8 compatibility, flexibility, learning curve, and whether I need OAuth support later. Recommend an approach."

### Step 2: Don't Block

Go do other work. Claude.ai sends a push notification to your phone when the answer is ready.

### Step 3: Review the Report

The answer lays out the trade-offs and recommends one option, with reasons. If you disagree with part of it, ask follow-up questions until you're sure about the direction.

### Step 4: Request Implementation Instructions

Once you've decided, ask for a handoff:

> "I've decided to use Rodauth. Create two things:
> 1. A full report I can reference later (architecture decisions, why Rodauth, key configuration choices)
> 2. A copy/paste-able markdown prompt for Claude Code with step-by-step implementation instructions. Assume Claude Code has full access to my codebase but doesn't know anything about this research conversation."

You'll paste the second one into Claude Code.

### Step 5: Hand Off to Claude Code

Open Claude Code in your project and paste the implementation prompt. It reads your existing code and builds the chosen approach to match how the rest of the app is written.

## Why This Works

### Research Quality

Extended thinking spends minutes where Claude Code spends seconds. It uses that time to compare several options and look for problems with each before it recommends one.

### Context Separation

"Which auth gem should I use?" doesn't depend on how your files are laid out. If the research can't see the codebase, the code that's already there can't pull the decision one way before you've made it.

### Non-Blocking Work

Watching a model think isn't productive time. With the phone notification you can go do something else, and you can have several research questions running at once.

### Clean Handoff

The implementation prompt carries the decision and the reasons for it, so Claude Code doesn't redo the research. It takes the decision as given and applies it to your code, which the web session never saw.

## Example: Choosing an Authentication Gem

This example is made up. The Rails app behind this blog never ran this comparison. Its accounts use Rails' built-in `has_secure_password`, chosen in 2022, and it has never used Devise or Rodauth. The gem names are just an example; any decision with several options works the same way.

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
- Big dependency decisions (auth, payments, background jobs)
- Architecture choices (monolith vs services, database design)
- Picking a technology (which API, which service, which pattern)
- Anything where "undo" is expensive

**Skip research for:**
- Well-defined implementation tasks
- Bug fixes with clear solutions
- Features where the approach is obvious
- Small, reversible changes

## The Artifacts

Ask for both documents, because they have different readers. The report is for you. The prompt is for Claude Code.

### 1. Reference Report

The full analysis, for you to reread later. Include:
- The options considered, and why each was picked or dropped
- The main configuration choices and the reasons for them
- Known limits and things to revisit later
- Links to documentation

### 2. Implementation Prompt

Copy/paste instructions, for Claude Code. Include:
- What to build
- The technical choices already made
- The order to build it in
- A line telling Claude Code not to reopen the research

## Results

We didn't measure anything. The decision now gets made on Claude.ai, with the reasons written down, before Claude Code opens a file. The cost is the wait, minutes per question instead of seconds, plus one extra prompt for the two documents. The reference report is the piece that lasts, because you can reread the reasoning after the implementation is done.

## Lessons Learned

- **Match eagerness to how easy undo is.** Let Claude Code dive in when undo is cheap. When it's expensive, decide first, somewhere it can't start typing.
- **Write the implementation prompt for a reader who knows nothing about the research.** Say what was chosen and why, so it doesn't reopen the question.
- **Ask for the reference report and the prompt together.** The prompt gets used once. You'll reread the report when someone asks why you chose this.
