---
layout: post
title: "AI Agent Permissions: Trust but Verify"
date: 2026-03-03 08:00:00 -0600
summary: "A three-tier allow, ask, deny policy for Claude Code and Codex: local work runs unprompted, pushes get a glance, and main and the deploy remotes are blocked outright."
tags: [patterns, claude, codex, security]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Click "allow" on every command a coding agent runs and within a day you are approving without reading. Run `--dangerously-skip-permissions` instead and the agent does whatever it decides to, including whatever a file, issue, or PR comment told it to. The first mode is safe until the safety wears off. The second is fast and, outside a sandbox, one misread instruction from an accident.

Our web repo is worked on by both Claude Code and Codex. The pass that produced the config below loosened an earlier, stricter version: file edits and local git now run without a prompt, and skip-permissions stays off the table.

## The Solution: Tiered Permissions

Three tiers, matching oversight to risk:

| Tier | Risk Level | Examples |
|------|------------|----------|
| **Allow** | Low | `git status`, `gh pr view`, file edits |
| **Ask** | Medium | `git push`, `gh pr create`, dependency changes |
| **Deny** | High | Push to main, deploy to production |

The sorting criterion is whether an action is local and reversible. Most development work is, so the agent runs freely there. The gate sits on anything that touches shared systems or cannot be undone. The block sits on the few targets where even an approval is the wrong answer.

## Implementation

Claude Code reads a JSON settings file with `allow`, `ask`, and `deny` lists. Codex reads prefix rules with a decision of `allow`, `prompt`, or `forbidden`. Both files below are trimmed to the entries that show the shape; the policy by category is in the next section.

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
      "Bash(git push* origin *:main*)",
      "Bash(git push* staging*)",
      "Bash(git push* production*)"
    ]
  }
}
```

The push rules are what to notice. `git push*` is in `ask` and the protected targets are in `deny` (main under both spellings, plus the two deploy remotes), so a push prompts by default and a push to main or a deploy remote is refused, approval or not.

### Codex: `.codex/rules/default.rules`

```
# Mapping: allow -> decision="allow", ask -> decision="prompt", deny -> decision="forbidden"

# forbid - protect main and deployment targets; guard rules carry
# match/not_match self-tests that execpolicy checks every time it loads the file
prefix_rule(
    pattern=["git", "push", "origin", "main"],
    decision="forbidden",
    match=["git push origin main", "git push origin main --tags"],
    not_match=["git push origin feature-branch"],
)
prefix_rule(pattern=["git", "push", "origin", "HEAD:main"], decision="forbidden")
prefix_rule(pattern=["git", "push", "staging"], decision="forbidden")
prefix_rule(pattern=["git", "push", "production"], decision="forbidden")

# prompt - actions that affect shared state
prefix_rule(pattern=["git", "push"], decision="prompt")
prefix_rule(pattern=["gh", "pr", "create"], decision="prompt")
prefix_rule(pattern=["gh", "pr", "merge"], decision="prompt")

# allow - local development operations
prefix_rule(pattern=["git", "status"], decision="allow")
prefix_rule(pattern=["git", "commit"], decision="allow")
prefix_rule(pattern=["gh", "pr", "view"], decision="allow")
```

Same shape, different vocabulary: forbidden push targets first, then prompts on shared-state commands, then allows for local work. The self-tests on the guard rules arrived in July 2026; a rule that stops matching its own examples fails to load, so the test matrix in the skill stays executable.

## The Policy

### Allow (no prompt needed)

- **File operations:** `Edit`, `Write`, `Read`
- **Local git:** `add`, `commit`, `checkout`, `branch`, `diff`, `log`, `status`
- **Read-only GitHub:** `pr view`, `issue view`, `run view`
- **Tests:** `bundle exec rails test`

Local, reversible, and nothing here changes anything outside the working copy.

### Ask (requires approval)

- **Git pushes:** `push` (except denied patterns)
- **Git rewrites:** `merge`, `rebase`
- **GitHub writes:** `pr create`, `pr close`, `pr merge`
- **Dependencies:** `bundle install`, `bundle update`
- **GitHub API:** `gh api` is left unlisted, so it prompts (it was allowed in GET-only form until July 2026)

These touch shared state. A glance at the confirmation is worth the friction.

### Deny (blocked completely)

- **Main branch:** `git push origin main`
- **Deployments:** `git push staging`, `git push production`
- **Dangerous opens:** `bundle open` (can modify gems)
- **Mutating API calls:** `gh api` with `POST`, `PUT`, `PATCH`, or `DELETE`; since July 2026, `curl` with `-X`, `-d`, or `-F` as well

These never happen autonomously. The config refuses them, so approval is not on offer.

## Keeping Configs in Sync

Two formats, one policy. Without a written procedure the files drift, so a skill documents the order of operations (the middle steps, one per specific guard, are omitted here):

```markdown
## Sync Workflow

1. Update `.claude/settings.json` first.
2. Mirror behavior into `.codex/rules/default.rules`.
3. Keep explicit forbidden rules for protected push targets even if `git push` is generally prompt.
...
9. Validate both configs:
   - `jq empty .claude/settings.json`
   - `codex execpolicy check --pretty --rules .codex/rules/default.rules -- git push origin main`
   - `codex execpolicy check --pretty --rules .codex/rules/default.rules -- git push origin feature-branch`
```

The mapping:
- Claude `allow` → Codex `decision="allow"`
- Claude `ask` → Codex `decision="prompt"`
- Claude `deny` → Codex `decision="forbidden"`

## Why Not YOLO?

Running `--dangerously-skip-permissions` in a container with no production keys is defensible. For daily development on your main machine, three things argue against it.

**Prompt injection risk:** Malicious content in files, issues, or PRs could instruct the agent to take harmful actions.

**Accident risk:** The agent might misunderstand intent and push to the wrong branch, merge prematurely, or modify dependencies unexpectedly.

**Audit trail:** Approving an action means reading what is about to happen, and that reading catches mistakes. The `ask` tier keeps this without paying for it on every local command.

## Results

- Most commands run without a prompt. Approval fatigue is gone.
- A push still takes a glance, which is where wrong-branch mistakes get caught.
- Main and the deploy remotes are unreachable from the agent, with or without approval.
- Claude Code and Codex enforce the same policy. The cost is two files to update, in order, on every change.

## Lessons Learned

- **Block at the agent level even where the host protects the branch.** Branch protection is the second layer, not the only one.
- **Deny the specific target, ask on the general command.** `git push*` prompts; `git push* origin main*` is refused. The general case stays usable.
- **Sort by local and reversible, not by how the command sounds.** Read-only and local git are safe to allow broadly.
- **A prompt on something safe teaches you to stop reading.** Reserve `ask` for actions worth a glance.
- **Two formats need a written mapping and a fixed update order.** Name which file is edited first.
