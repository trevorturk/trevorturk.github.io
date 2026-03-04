---
layout: post
title: "AI Agent Permissions: Trust but Verify"
date: 2026-03-03 08:00:00 -0600
summary: "A tiered permission system for Claude Code and Codex that balances autonomy with safety."
tags: [patterns, claude, codex, security]
---

## The Problem

AI coding agents are powerful, but how much autonomy should you give them? The community seems split:

**YOLO mode:** Run `--dangerously-skip-permissions` and let the agent do whatever it wants. Fast, but risky without sandboxing.

**Approval fatigue:** Click "allow" on every single command. Safe, but tedious enough that you stop paying attention.

Neither extreme works well. Full autonomy risks accidents. Constant prompting trains you to approve without reading.

## The Solution: Tiered Permissions

We use a three-tier system that matches risk to oversight:

| Tier | Risk Level | Examples |
|------|------------|----------|
| **Allow** | Low | `git status`, `gh pr view`, file edits |
| **Ask** | Medium | `git push`, `gh pr create`, dependency changes |
| **Deny** | High | Push to main, deploy to production |

The key insight: **most development work is local and reversible**. Let the agent work freely there. Gate the actions that touch shared systems or can't be undone.

## Implementation

### Claude Code: `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Edit",
      "Write",
      "Bash(git add*)",
      "Bash(git commit*)",
      "Bash(git checkout*)",
      "Bash(git diff*)",
      "Bash(git status*)",
      "Bash(gh pr view*)",
      "Bash(gh issue view*)",
      "Bash(bundle exec rails test*)"
    ],
    "ask": [
      "Bash(git push*)",
      "Bash(git merge*)",
      "Bash(gh pr create*)",
      "Bash(gh pr merge*)",
      "Bash(bundle install*)"
    ],
    "deny": [
      "Bash(git push* origin main*)",
      "Bash(git push* staging*)",
      "Bash(git push* production*)"
    ]
  }
}
```

### Codex: `.codex/rules/default.rules`

```
# Mapping: allow -> decision="allow", ask -> decision="prompt", deny -> decision="forbidden"

# Forbidden - protect main and deployment targets
prefix_rule(pattern=["git", "push", "origin", "main"], decision="forbidden")
prefix_rule(pattern=["git", "push", "staging"], decision="forbidden")
prefix_rule(pattern=["git", "push", "production"], decision="forbidden")

# Prompt - actions that affect shared state
prefix_rule(pattern=["git", "push"], decision="prompt")
prefix_rule(pattern=["gh", "pr", "create"], decision="prompt")
prefix_rule(pattern=["gh", "pr", "merge"], decision="prompt")

# Allow - local development operations
prefix_rule(pattern=["git", "status"], decision="allow")
prefix_rule(pattern=["git", "commit"], decision="allow")
prefix_rule(pattern=["gh", "pr", "view"], decision="allow")
```

## The Policy

### Allow (no prompt needed)

- **File operations:** `Edit`, `Write`, `Read`
- **Local git:** `add`, `commit`, `checkout`, `branch`, `diff`, `log`, `status`
- **Read-only GitHub:** `pr view`, `issue view`, `run view`, `api:GET`
- **Tests:** `bundle exec rails test`

These are safe. They're local, reversible, and part of normal development flow.

### Ask (requires approval)

- **Git pushes:** `push` (except denied patterns)
- **Git rewrites:** `merge`, `rebase`
- **GitHub writes:** `pr create`, `pr close`, `pr merge`
- **Dependencies:** `bundle install`, `bundle update`

These touch shared state. A quick glance at the confirmation is worth the friction.

### Deny (blocked completely)

- **Main branch:** `git push origin main`
- **Deployments:** `git push staging`, `git push production`
- **Dangerous opens:** `bundle open` (can modify gems)

These should never happen autonomously. Even with approval, they're blocked at the config level.

## Keeping Configs in Sync

Claude Code and Codex use different formats but should enforce the same policy. We created a skill to document the sync process:

```markdown
## Sync Workflow

1. Update `.claude/settings.json` first
2. Mirror behavior into `.codex/rules/default.rules`
3. Keep explicit forbidden rules for protected push targets
4. Validate both configs with test commands
```

The mapping:
- Claude `allow` → Codex `decision="allow"`
- Claude `ask` → Codex `decision="prompt"`
- Claude `deny` → Codex `decision="forbidden"`

## Why Not YOLO?

Running `--dangerously-skip-permissions` in a container with no production keys is defensible. But for daily development on your main machine?

**Prompt injection risk:** Malicious content in files, issues, or PRs could instruct the agent to take harmful actions.

**Accident risk:** The agent might misunderstand intent and push to the wrong branch, merge prematurely, or modify dependencies unexpectedly.

**Audit trail:** When you approve actions, you're reviewing what's about to happen. That review catches mistakes.

The tiered approach gives you autonomy where it matters (local work) and oversight where it matters (shared systems).

## Results

After implementing this:

- **No more approval fatigue** - Most commands just run
- **Pushes still require a glance** - Catches wrong-branch mistakes
- **Main is protected** - Can't accidentally push to main even with approval
- **Both tools aligned** - Claude and Codex enforce the same policy

## Lessons Learned

- **Deny rules beat GitHub protection** - Don't rely solely on branch protection; block at the agent level too
- **Allow patterns are generous** - Read-only operations and local git are safe to allow broadly
- **Ask is the middle ground** - Reserve it for "check before doing" actions
- **Keep both configs in sync** - Different tools, same policy

---

## How This Post Was Made

**Prompt:** "I'm seeing a bunch of chatter lately about claude code permissions. we recently did a bunch of work in helloweather/web to make permissions a bit more flexible and to allow a bit more editing files etc, but tried to keep things a bit more strict and defintiely avoiding dangerously-skip-permissions. see some chatter here: [discussion about YOLO mode, containers, and permission approaches] -- don't reference names, but I'd like you to use the blog post skill and add a post to our blog about how we're doing permissions for claude and codex, using a new permissions skill we added. see recent prs in helloweather/web. create a pr with a blog post about how we're approaching permissions for sharing with others. make sure to include this prompt as the 'how this was made' part as well."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Context gathered from commit 77c236d5 ("Streamline Claude Code permissions v2") and the permissions sync skill in helloweather/web.
