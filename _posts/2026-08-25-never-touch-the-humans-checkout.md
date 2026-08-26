---
layout: post
title: "Never Touch the Human's Checkout: Worktrees, DerivedData, and the QA Handoff for Agents Sharing a Repo"
date: 2026-08-25 08:30:00 -0600
summary: "One person runs several coding agents against an iOS repo that Xcode has open. The rules that make that safe: every change on a branch in a gitignored worktree, a separate DerivedData path, shared package and compile caches, and a one-way QA handoff back to the human's checkout."
tags: [ai-agents, workflow, ios, tooling]
---

## The Problem

[Hello Weather](https://helloweather.com) is a one-person product, and that person builds it in Xcode with the repository open all day. The same repository is where several coding agents do their work at the same time — drafting a fix, editing a plan, updating a skill. So the agents and the human share one working copy of one repo, and the human's copy is not idle: it has an editor holding files open, an incremental build database on disk, and a branch checked out that the human is in the middle of testing.

That sharing goes wrong in two distinct ways, and both bit us before the rules existed.

The first is the working tree. An agent that edits a file in the shared checkout, or runs `git checkout` to switch branches, changes what the human sees under their own cursor. Early on an agent switched branches mid-QA and the human's build started failing against code they had never written; another left a half-finished edit behind, so the next `git status` was a mess nobody could attribute. A shared working tree has exactly one HEAD and one set of files, and two actors writing to it race.

The second is subtler and more expensive: the build. Xcode and command-line `xcodebuild` both write to a DerivedData folder — a build database plus compiled artifacts — and they cannot share one. When an agent ran `xcodebuild` against the same DerivedData Xcode was using, the human's next build failed with "database is locked" or picked up stale artifacts, and the only reliable recovery was Clean Build Folder and a full recompile. A weather app with a watch target and a widget target is not a fast clean build, and losing it mid-afternoon to an agent's background test run is the kind of friction that makes you turn the agents off.

Neither problem is about agent output quality. Both are about coexistence: an autonomous process sharing a repo and a build system with a human's live IDE. A [sibling post](/agent-fanout-isolation-contract/) covers keeping parallel agents from colliding with *each other* when they merge; this post is the other axis: keeping every agent out of the way of the *human's* editor and build state.

## The Solution

Give every agent its own isolated checkout and its own build location, and make the isolation the default rather than a thing to remember. Concretely: every agent change, down to a one-line doc edit, happens on a branch in a git worktree created before the first edit; worktrees live under a gitignored directory inside the repo; agent builds write to a separate DerivedData path via an environment variable, while a shared package checkout and compile cache keep isolation from becoming constant re-cloning and re-compiling; and there is exactly one sanctioned path back into the human's checkout — the QA handoff — with a checklist that keeps it one-way.

### Every change goes in a worktree, created before the first edit

The reason to reach for worktrees rather than "just be careful in the shared checkout" is that careful does not survive contact with a background agent. A git worktree is a second working directory attached to the same repository, with its own branch and its own files, sharing the one object store. The human's checkout and the agent's worktree can sit on different branches with different uncommitted changes and never touch each other's files. That is the whole property we need.

The rule is absolute on purpose: no edits in the main checkout at all, including plan and documentation one-liners, because a rule with a "small changes are fine" exception is one an agent will talk itself past. It is enforced two ways in this repo — written in `AGENTS.md` and backed by a PreToolUse hook that blocks edits outside a worktree (landed 2026-08-04). The load-bearing sentence:

> All agent work happens in git worktrees — the main checkout belongs to the user and stays untouched. Each worktree is an isolated checkout of its own branch. Worktrees live under the repo's gitignored `.claude/worktrees/` directory — never as siblings of the repo.

A worktree that only edits text is free to create. A worktree that has to *build* needs the step everyone forgets: the two secret files this project keeps out of git — an API-keys xcconfig and a crash-reporter config — are not copied into a fresh worktree, because git only populates tracked files. Without them `xcodebuild` fails immediately on the missing base configuration reference. So the setup script creates the worktree, links the secrets, and sets the build-isolation variables in one go, runnable as-is against any repo with the same shape:

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

Notice that the secrets are symlinked rather than copied: they stay gitignored, and rotating a key updates every worktree at once. The three exported variables are the subject of the next section. One operational note for agents: pass **absolute paths** in every file operation inside the worktree, because a relative path resolves against whatever directory the shell last landed in — in a workspace with sibling repos, rarely the worktree you meant.

### Agent builds write to their own DerivedData, and share the caches that are safe to share

The reason for a separate DerivedData path is the "database is locked" failure above: Xcode's build database and a CLI build's cannot coexist. The fix is one flag, `-derivedDataPath`, wired to an environment variable so the human default (unset, Xcode's location untouched) and the agent path (set, private folder) are the same script. But naive isolation has a cost: if every worktree gets a *completely* private build world, each one re-clones every package and recompiles every file, and a fleet of worktrees spends all day doing that.

So the split is deliberate. The build *database* is per-worktree, because it is what conflicts. The package checkout and the content-addressed compile cache are *shared* across worktrees, because they are safe to share — the package cache is just downloaded source, and the compile cache is content-addressed, so a later worktree reuses an earlier worktree's compiles instead of redoing them (the shared-cache work landed 2026-08-07 through 08). Here are the trimmed-but-complete lines from the build script showing all three passed to `xcodebuild`:

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

The thing to notice is which knob is per-worktree and which is shared. `-derivedDataPath` is per-worktree because the build database is the resource that conflicts; `-clonedSourcePackagesDirPath` and the compile cache are shared because they are inputs, not mutable build state. One discipline the variable cannot enforce: never run two builds at once even with separate DerivedData paths, because they still contend on the shared cache in ways that corrupt it. Isolation gives each agent a safe *place* to build; serialization keeps them off the shared parts.

### Disk fills up, so cleanup is part of the workflow, not an afterthought

The reason cleanup is a first-class rule is arithmetic. Every worktree that builds spawns roughly 6 GB of DerivedData, so ten units of work in a day is 60 GB of build cache for changes that may already be merged. Left alone, the disk fills and every build fails for a reason that has nothing to do with any agent's code.

A worktree is done the moment its branch is squash-merged into main; the object store keeps the history, so its working directory and build cache are waste from then on. The cleanup script prunes stale worktree records, then deletes DerivedData for any workspace path that no longer exists — exactly the set left behind by removed worktrees. Safe to run at the end of any session:

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

Notice it never deletes by age or by guesswork. It reads each DerivedData folder's recorded `WorkspacePath` and deletes only when that path is gone — so an active worktree's cache is never touched, and a removed worktree's cache never survives. The signal to clean is structural (the workspace no longer exists), not a heuristic.

### The QA handoff is the one sanctioned way back into the human's checkout

Everything above keeps agents *out* of the human's checkout. But there is one moment the human genuinely wants a branch *in* it: QA. When the human says "let me test that branch," they mean in Xcode, in the checkout they already have open — not in a worktree they would have to go find. That is the single sanctioned exception, and it needs its own discipline because it is the one time the two worlds deliberately touch.

The failure mode it prevents is a build war: if an agent runs a CLI build against the checkout the human's IDE is building, it stomps the incremental build state and forces a clean rebuild mid-QA — the exact corruption from the first section, at the worst possible moment. So the handoff is one-way: once a branch moves to the human's checkout, the checkout owns it and agents stop building it. The checklist:

```markdown
## QA handoff (worktree -> human's checkout)

1. Push the branch. Remove the worktree (the branch persists);
   check the branch out in the human's checkout. Displacing whatever
   it was on is expected — the human QAs one thing at a time.
2. Open the PR if it isn't already, so the human reviews code on
   GitHub while building the branch locally.
3. One-way: from here the human's checkout owns this branch. Follow-up
   commits happen there, not in a recreated worktree.
4. Never run a CLI build (xcodebuild) in the human's checkout after
   the handoff — it stomps their live incremental build state.
5. Never tell the human to `git pull` for follow-up commits made in
   their own checkout — those commits are already in their working copy.
```

That last line looks trivial and is not: an agent that committed a fix in the human's checkout and then told them to `git pull` to see it caused real confusion, because the commit was already in their working copy — there was nothing to pull. After the handoff the mental model flips: the checkout is the working copy, and the agent is a guest in it.

One smaller rule rides along on any pushed branch, handoff or not: never force-push it. PR branches are additive-only, so review fixes are new commits, never a rebase that rewrites what someone may have already pulled.

## Results

- Agents and a human share one iOS repo all day without the human's editor or build state ever being disturbed. Branch switches, stray edits, and locked build databases in the shared checkout went from recurring interruptions to structurally impossible, because the worktree rule is enforced by a hook rather than by remembering.
- Build isolation costs one extra DerivedData cache (~6 GB) per concurrent worktree, not one per package: the shared package checkout and compile cache let a second worktree reuse the first's downloads and compiles.
- "Database is locked" and stale-artifact rebuilds during the human's own Xcode sessions stopped, because no agent build writes to the DerivedData Xcode is using.
- The QA handoff gives the human a tested branch in the checkout they already have open, with the PR up for review and no agent build fighting their live one.

## Lessons Learned

- **Isolate by default, and enforce it mechanically.** A "small edits in the shared checkout are fine" exception is one an agent rationalizes its way into, so the rule is zero edits outside a worktree, backed by a hook — a policy you have to remember is a policy that fails at 2 a.m.
- **Separate the build state that conflicts; share the state that is safe.** DerivedData holds a build database and must be per-worktree; the package checkout and compile cache are inputs, so sharing them turns isolation from "re-clone everything" into "reuse everything but the database."
- **Copy the secrets git won't.** A worktree only gets tracked files, so gitignored configs must be linked in before the first build or `xcodebuild` fails on a missing base configuration.
- **Make cleanup structural, not a habit.** Delete a build cache when its workspace path no longer exists, never by age or by hand, so active worktrees survive and dead ones never linger.
- **The handoff is one-way, and the human's checkout owns the branch after it.** Once a branch is in the IDE the human builds live, no agent runs a CLI build against it and no agent tells them to pull commits already in their working copy.

---

## How This Post Was Made

**Prompt 1:** "please kick off a big batch to look through all skills looking for other topics that might be interesting to blog about. we could look at git history, but I think since we've been using claude/codex for the last ~year we should have most of the interesting stuff built into the skills by now. however, you can also look at the changelog view in the iOS repo for other highlights that might be worth dispatching research about. come back to me with a list of possible topics (that haven't already been covered in the blog) …"

**Prompt 2:** "lets do 4, 20, 21, 22 -- the others I think are not worth it"

Ten Claude agents mined the iOS, web, and Android skills, the iOS changelog, and the plan indexes for uncovered topics; the owner picked four from the ranked list. This post was researched and drafted by one agent from the cited skills, plans, and code, under the why-then-how voice and self-contained-code brief settled in the previous localization batch, then reviewed before publishing.
