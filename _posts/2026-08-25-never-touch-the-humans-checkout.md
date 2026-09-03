---
layout: post
title: "Never Touch the Human's Checkout"
date: 2026-08-25 08:30:00 -0600
summary: "Several coding agents share an iOS repo that a human has open in Xcode. The rules that keep them apart: every agent change in a gitignored worktree, a private DerivedData path with shared package and compile caches, cleanup triggered by structure, and a one-way QA handoff back to the human's checkout."
tags: [ai-agents, workflow, ios, tooling]
model: "Claude"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

An agent switched branches in the middle of the human's QA session, and the human's next build failed against code they had never written. A different agent left a half-finished edit behind, and the next `git status` was a mess nobody could attribute. A third ran `xcodebuild` while Xcode was building, and the human's build died with "database is locked." The only recovery was Clean Build Folder and a full recompile of an app with a watch target and a widget target.

All three happened in the same setup. [Hello Weather](https://helloweather.com) is a one-person product, and that person builds it in Xcode with the repository open all day. Several coding agents work in the same repository at the same time, drafting fixes, editing plans, updating skills. The human's copy is never idle. It has an editor holding files open, an incremental build database on disk, and a branch checked out mid-test.

The failures sort into two kinds. The working tree has one HEAD and one set of files, so two actors writing to it race. The build has one DerivedData folder, holding a build database plus compiled artifacts, and Xcode and command-line `xcodebuild` cannot share it. Neither problem is about agent output quality. Both are about an autonomous process coexisting with a human's live IDE. A [sibling post](/agent-fanout-isolation-contract/) covers keeping parallel agents from colliding with *each other* when they merge. This post is about keeping every agent out of the *human's* editor and build state.

## The Solution

Give every agent its own checkout and its own build location, and make that the default rather than something to remember. Four rules:

- Every agent change, down to a one-line doc edit, happens on a branch in a git worktree created before the first edit. Worktrees live under a gitignored directory inside the repo.
- Agent builds write to a private DerivedData path. The package checkout and compile cache stay shared across worktrees.
- Cleanup is triggered by structure, not habit: a build cache is deleted when its worktree no longer exists.
- Exactly one path leads back into the human's checkout, the QA handoff, and it is one-way.

### Every change goes in a worktree

"Be careful in the shared checkout" does not survive contact with a background agent. A git worktree is a second working directory attached to the same repository, with its own branch and its own files, sharing the one object store. The human's checkout and the agent's worktree can sit on different branches with different uncommitted changes and never touch each other's files.

The rule is absolute on purpose. No edits in the main checkout at all, including plan and documentation one-liners, because a rule with a "small changes are fine" exception is one an agent will talk itself past. It is written in `AGENTS.md`, which every session loads (landed 2026-08-04, replacing wording that had allowed branches in the main checkout):

> The main checkout is the user's (Xcode has it open). Never work in it — no edits, no branches, no uncommitted state left behind — unless the user explicitly authorizes it in the current session. All work, including one-line plan/doc edits, happens on a branch in a worktree under `.claude/worktrees/<name>` (gitignored; never a sibling of the repo in `~/Code/helloweather`), created before the first change.

The same PR first tried a PreToolUse hook that blocked edits outside a worktree. Review dropped it, so the rule is enforced by instructions, not tooling. What makes it hold is that it has no exceptions and is read at the start of every session.

A worktree that only edits text is free to create. A worktree that has to *build* needs one more step. Git only populates tracked files, so the two secret files this project keeps out of git, an API-keys xcconfig and a crash-reporter config, are missing from a fresh worktree, and `xcodebuild` fails immediately on the missing base configuration. Written as one script, the setup creates the worktree, links the secrets, and sets the build-isolation variables in one go:

```bash
#!/bin/bash
set -euo pipefail

REPO="$HOME/Code/app/ios"          # the human's checkout (Xcode has this open)
NAME="fix-widget-crash"            # short name for this unit of work
WORKTREE="$REPO/.claude/worktrees/$NAME"

# Gitignored secrets git will NOT copy into a fresh worktree. Link, don't copy,
# so a rotated key updates everywhere at once. Names are illustrative.
SECRETS=(
  "app/Resources/Secrets.xcconfig"
  ".crashrc"
)

git -C "$REPO" fetch origin main
git -C "$REPO" worktree add -b "feature/$NAME" "$WORKTREE" origin/main

for rel in "${SECRETS[@]}"; do
  mkdir -p "$WORKTREE/$(dirname "$rel")"
  ln -sf "$REPO/$rel" "$WORKTREE/$rel"
done

# Build isolation: a private DerivedData so this never shares a build database
# with Xcode. Shared package + compile caches so isolation isn't re-cloning.
export HW_DERIVED_DATA="$HOME/Library/Developer/Xcode/DerivedData/app-agent"
export HW_SPM_CACHE="$HOME/Library/Caches/app-SPM"
export HW_CAS_PATH="$HOME/Library/Caches/app-CAS"

echo "Worktree ready: $WORKTREE"
```

The secrets are symlinked rather than copied, so they stay gitignored and a rotated key updates every worktree at once. The three exported variables are the subject of the next section. One more habit for agents working inside a worktree: use absolute paths in every file operation. A relative path resolves against whatever directory the shell last landed in, and in a workspace with sibling repos that is rarely the worktree you meant.

### Isolate the build database, share the caches

The "database is locked" failure has a one-flag fix: `-derivedDataPath`, wired to an environment variable. Unset, the human's default leaves Xcode's location untouched. Set, the agent builds into a private folder.

But a fully private build world per worktree is expensive. Each one would re-clone every package and recompile every file, and a fleet of worktrees would spend all day doing that. So the split follows what actually conflicts. The build *database* is private: the agent path when the variable is set, Xcode's own per-workspace location otherwise. The package checkout is shared, because it is just downloaded source. The compile cache is shared, because it is content-addressed, so a later worktree reuses an earlier worktree's compiles instead of redoing them (landed 2026-08-07 through 08). The build script passes all three to `xcodebuild`; this is the real script trimmed to the flags that matter, one scheme instead of two:

```bash
#!/bin/bash
set -euo pipefail

# Private DerivedData when set (agents/CI); unset keeps Xcode's shared location.
DERIVED_DATA=()
if [ -n "${HW_DERIVED_DATA:-}" ]; then
  DERIVED_DATA=(-derivedDataPath "$HW_DERIVED_DATA")
fi

# Shared across worktrees: downloaded packages, and the content-addressed
# compile cache (safe to share; it is not the build database).
SPM_CACHE="${HW_SPM_CACHE:-$HOME/Library/Caches/app-SPM}"
CAS_PATH="${HW_CAS_PATH:-$HOME/Library/Caches/app-CAS}"

xcodebuild build \
  -workspace App.xcworkspace \
  -scheme AppWidgets \
  -destination 'generic/platform=iOS Simulator' \
  "${DERIVED_DATA[@]}" \
  -clonedSourcePackagesDirPath "$SPM_CACHE" \
  -quiet \
  COMPILATION_CACHE_ENABLE_CACHING=YES \
  COMPILATION_CACHE_CAS_PATH="$CAS_PATH"
```

One rule the variables cannot enforce: never run two builds at once, even with a private DerivedData path. They still share the package checkout and the compile cache, and the rule is simply never to test what happens when two builds write to those together. Isolation gives each agent a safe place to build. Serialization keeps them off the shared parts.

### Cleanup is structural

Every worktree that builds spawns roughly 6 GB of DerivedData, and a full private agent cache runs about 8 GB. Ten units of work in a day is 60 GB or more of build cache for changes that may already be merged. Left alone, the disk fills and every build fails for a reason unrelated to any agent's code.

A worktree is done the moment its branch is squash-merged into main. The object store keeps the history, so the working directory and its build cache are waste from then on. The cleanup script prunes stale worktree records, then deletes DerivedData for any workspace path that no longer exists. That is exactly the set left behind by removed worktrees, so it is safe to run at the end of any session. Trimmed here; the real one also drops a simulator tool's caches and unavailable simulators:

```bash
#!/bin/bash
set -euo pipefail

REPO="$HOME/Code/app/ios"

echo "Pruning stale worktree records..."
git -C "$REPO" worktree prune

echo "Deleting DerivedData for workspaces that no longer exist..."
for dir in "$HOME/Library/Developer/Xcode/DerivedData/"app-*/; do
  [ -d "$dir" ] || continue
  workspace="$(/usr/libexec/PlistBuddy -c "Print WorkspacePath" "$dir/info.plist" 2>/dev/null || true)"
  if [ -n "$workspace" ] && [ ! -e "$workspace" ]; then
    echo "  removing $(basename "$dir") ($workspace)"
    rm -rf "$dir"
  fi
done

df -h / | tail -1 | awk '{print "Free space: " $4}'
```

It never deletes by age or by guesswork. It reads each DerivedData folder's recorded `WorkspacePath` and deletes only when that path is gone. An active worktree's cache is never touched, and a removed worktree's cache never survives.

### The QA handoff is one-way

Everything above keeps agents *out* of the human's checkout. There is one moment the human wants a branch *in* it. When they say "let me test that branch," they mean in Xcode, in the checkout they already have open, not in a worktree they would have to go find. That is the single sanctioned exception, and it is the one time the two worlds deliberately touch.

The handoff has to be one-way. Once the branch is in the human's checkout, the checkout owns it. Follow-up commits happen there, and no agent runs a CLI build against it, because that would stomp the live incremental build state mid-QA. The checklist, condensed from the pull-requests skill:

```markdown
## QA handoff (worktree -> human's checkout)

Confirm the handoff with the human first, then:

1. Push the branch. Remove the worktree (the branch persists);
   check the branch out in the human's checkout. Displacing whatever
   it was on is expected — the human QAs one thing at a time.
2. Open the PR if it isn't already, so the human reviews code on
   GitHub while building the branch locally.
3. One-way: from here the human's checkout owns this branch. Follow-up
   commits happen there, not in a recreated worktree.
4. Never tell the human to `git pull` for follow-up commits made in
   their own checkout — those commits are already in their working copy.
5. Iterate conversationally: propose each follow-up as a readable diff
   in chat and get agreement before committing. Approved changes still
   push promptly so the PR tracks the conversation.
6. Never run a CLI build (xcodebuild) in the human's checkout after
   the handoff — it stomps their live incremental build state. Their
   next Xcode build verifies the follow-up; for anything riskier than
   a trivial change, say so, or verify in the worktree before handoff.
```

The last item looks trivial and is not. An agent once committed a fix in the human's checkout and then told them to `git pull` to see it. The commit was already in their working copy, so there was nothing to pull, and the instruction caused real confusion. After the handoff the mental model flips: the checkout is the working copy, and the agent is a guest in it.

One smaller rule rides along on any pushed branch, handoff or not: never force-push it. PR branches are additive-only. Review fixes are new commits, never a rebase that rewrites what someone may have already pulled.

## Results

- Branch switches, stray edits, and locked build databases in the human's checkout went from recurring interruptions to, as of August 2026, not recurring. The worktree rule is loaded in every session rather than remembered.
- The cost is one extra DerivedData cache, 6 to 8 GB, for each worktree that builds. Package downloads and compiles are reused across worktrees instead of repeated.
- The human gets a tested branch in the checkout they already have open, with the PR up for review and no agent build fighting their live one.

## Lessons Learned

- **Split state by whether it conflicts, not by what it is.** The build database conflicts and must be private. The package checkout and compile cache are inputs and can be shared. Treating all build state the same would have made isolation unaffordable.
- **Isolation makes concurrency possible, not safe.** A private DerivedData path still shares the package and compile caches, so builds are serialized. A rule the tooling cannot enforce belongs in the agent instructions.
- **A rule with a small-change exception is not a rule.** The worktree policy holds because it covers one-line doc edits too, and because it is in the file every session reads rather than in memory. The mechanical hook was tried and dropped; the wording did the work.
- **The handoff flips the agent's mental model, and the agent does not notice.** The phantom `git pull` was the symptom. After a handoff, the agent has to treat the human's working copy as the source of truth, not the branch it pushed.
