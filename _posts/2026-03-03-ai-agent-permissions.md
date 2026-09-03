---
layout: post
title: "AI Agent Permissions: Trust but Verify"
date: 2026-03-03 08:00:00 -0600
summary: "An allow, ask, deny policy for Claude Code and Codex. Local work runs without a prompt, a push asks first, and pushes to main or the deploy remotes are refused outright."
tags: [patterns, claude, codex, security]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

If you click "allow" on every command a coding agent runs, within a day you're approving without reading. If you run with `--dangerously-skip-permissions` instead, the agent does whatever it decides to, including whatever a file, issue, or PR comment told it to. The first mode stops being safe once you stop reading. The second is fast, but outside a sandbox one misread instruction can do real damage.

Both Claude Code and Codex work in our web repo. The config below came out of a pass that loosened an earlier, stricter version. File edits and local git now run without a prompt, and we still don't use skip-permissions.

## The Solution: Tiered Permissions

We sort every action into one of three tiers, with more oversight as the risk goes up:

| Tier | Risk Level | Examples |
|------|------------|----------|
| **Allow** | Low | `git status`, `gh pr view`, file edits |
| **Ask** | Medium | `git push`, `gh pr create`, dependency changes |
| **Deny** | High | Push to main, deploy to production |

The question for each action is whether it's local and reversible. Most development work is, so the agent runs it without asking. Anything that touches shared systems or can't be undone gets a prompt. A few targets are blocked outright, because there an approval would be the wrong answer too.

## Implementation

Claude Code reads a JSON settings file with `allow`, `ask`, and `deny` lists. Codex reads prefix rules, each with a decision of `allow`, `prompt`, or `forbidden`. We've trimmed both files below to the entries that show the shape. The full policy, by category, is in the next section.

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

Look at the push rules. `git push*` is in `ask`, and the protected targets are in `deny`: main under both spellings, plus the two deploy remotes. So a push prompts by default, and a push to main or a deploy remote is refused even if you'd have approved it.

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

Same shape, different words. The forbidden push targets come first, then prompts on commands that touch shared state, then allows for local work. The `match` and `not_match` self-tests on the guard rules arrived in July 2026. A rule that stops matching its own examples fails to load, so the test matrix in the skill gets run every time Codex loads the file, not just read.

## The Policy

### Allow (no prompt needed)

- **File operations:** `Edit`, `Write`, `Read`
- **Local git:** `add`, `commit`, `checkout`, `branch`, `diff`, `log`, `status`
- **Read-only GitHub:** `pr view`, `issue view`, `run view`
- **Tests:** `bundle exec rails test`

None of this changes anything outside the working copy, and all of it can be undone.

### Ask (requires approval)

- **Git pushes:** `push` (except denied patterns)
- **Git rewrites:** `merge`, `rebase`
- **GitHub writes:** `pr create`, `pr close`, `pr merge`
- **Dependencies:** `bundle install`, `bundle update`
- **GitHub API:** `gh api` is left unlisted, so it prompts (it was allowed in GET-only form until July 2026)

These touch shared state. A quick look at the confirmation is worth the interruption.

### Deny (blocked completely)

- **Main branch:** `git push origin main`
- **Deployments:** `git push staging`, `git push production`
- **Dangerous opens:** `bundle open` (can modify gems)
- **Mutating API calls:** `gh api` with `POST`, `PUT`, `PATCH`, or `DELETE`; since July 2026, `curl` with `-X`, `-d`, or `-F` as well

The config refuses these outright, so the agent never gets to ask.

## Keeping Configs in Sync

Two formats carry one policy. Without a written procedure the files drift apart, so a skill spells out the order of operations. We've left out the middle steps here, one per specific guard:

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

Running `--dangerously-skip-permissions` in a container with no production keys is a reasonable choice. For daily development on your main machine, we see three reasons not to.

- **Prompt injection:** something in a file, issue, or PR could tell the agent to do harm, and the agent might do it.
- **Accidents:** the agent might misread what you meant and push to the wrong branch, merge too early, or change dependencies you didn't ask about.
- **Audit trail:** to approve an action you have to read what's about to happen, and that's when you catch mistakes. The `ask` tier keeps that step without putting it in front of every local command.

## Results

- Most commands run without a prompt, so the approval fatigue is gone.
- A push still asks, and that's where we catch wrong-branch mistakes.
- The agent can't push to main or the deploy remotes, with or without approval.
- Claude Code and Codex enforce the same policy. The cost is two files to update, in a fixed order, every time the policy changes.

## Lessons Learned

- **Block at the agent level even if the host already protects the branch.** Branch protection is a second layer, not the only one.
- **Deny the specific target, ask on the general command.** `git push*` prompts and `git push* origin main*` is refused, so the general case stays usable.
- **Sort by whether an action is local and reversible, not by how the command sounds.** Read-only commands and local git are safe to allow broadly.
- **A prompt on something safe teaches you to stop reading prompts.** Save `ask` for actions worth a look.
- **Two formats need a written mapping and a fixed update order.** Name which file gets edited first.
