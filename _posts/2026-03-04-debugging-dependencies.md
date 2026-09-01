---
layout: post
title: "Debugging Dependencies: Research Before Workarounds"
date: 2026-03-04 09:00:00 -0600
summary: "Two patterns for working with third-party code: investigate source before guessing, and research upstream before patching."
tags: [ruby, debugging, dependencies, patterns]
---

## The Problem

Agent files were not being found, and the paths the loader built were malformed. The loader lived in a third-party gem. The tempting fix was to guess at the path manipulation and patch around it in our own code, or to monkey patch the loader.

Both build debt on top of a guess. A library that "isn't working" is usually being called wrong. Or the feature exists under a different name. Or someone already fixed it upstream, and the workaround would outlive the fix.

## Two Complementary Patterns

Two habits cover the two ways a dependency disappoints. When something does not work, read the source before guessing at what it does. When something is missing, research upstream before writing a workaround.

## Pattern 1: Investigate Source

Most package managers can locate dependency source and open it in an editor. In Ruby:

```bash
# Find where the gem is installed
bundle show gem_name
# => /path/to/gems/gem_name-1.2.3

# Open it in your editor
bundle open gem_name
```

Equivalent commands exist for other ecosystems:
- **npm/yarn:** `npm explore package-name` or check `node_modules/package-name`
- **pip:** `pip show -f package-name` to find location
- **cargo:** Check `~/.cargo/registry/src/`

### The Debugging Workflow

**Step 1:** Locate the dependency
```bash
bundle show problematic_gem
```

**Step 2:** Open and read the source
```bash
bundle open problematic_gem
# Search for the method/class causing issues
# Read the code to understand actual behavior
```

**Step 3:** Add debug statements if reading is not enough
```ruby
# Temporary changes in gem code:
puts "DEBUG: path = #{path.inspect}"
binding.break  # Stop execution here
```

**Step 4:** Run your test and watch the output
```bash
bin/rails test test/models/example_test.rb
```

**Step 5:** Restore the dependency
```bash
bundle pristine gem_name
```

### What You'll Often Find

- **Wrong usage** - The method expects different arguments than you're passing
- **Different naming** - The feature exists but is called something else
- **Configuration needed** - A setting enables the behavior you want
- **Version mismatch** - Your version doesn't have the feature yet

### Restore Before You Commit

Debug edits in a gem are temporary. Restore after every investigation, and never commit modified dependency source:
```bash
bundle pristine gem_name  # Restore single gem
bundle pristine           # Restore all gems
```

## Pattern 2: Research Before Patching

When the library lacks what you need, spend 25 minutes researching before writing a workaround. A patch written first tends to outlive the upstream fix it duplicates.

### The Research Protocol

**GitHub Issues (5 min)**
```bash
gh issue list --repo owner/gem-name --state all --limit 100 | grep -i "feature"
gh pr list --repo owner/gem-name --state all --limit 100 | grep -i "feature"
```

Look for:
- Open issues requesting the feature
- Open or merged PRs implementing it
- Discussions about the approach
- Closed issues explaining why it was rejected

**Recent Releases (5 min)**
```bash
gh release list --repo owner/gem-name --limit 10
```

Check changelogs for:
- Recent additions of the feature
- Beta or experimental support
- New configuration options

**Source Code Search (10 min)**
```bash
cd $(bundle show gem-name)
grep -r "feature_keyword" .
```

Look for:
- Existing implementation with different naming
- Partial implementation you can extend
- Configuration hooks that enable behavior

**Decision (5 min)**

Choose based on what you found, in this order:

1. **Use existing** - Feature exists, you missed it
2. **Wait for PR** - Open PR implements it, looks likely to merge
3. **Contribute upstream** - Feature missing, maintainer active
4. **Temporary workaround** - Last resort only

### If You Must Workaround

Some workarounds are unavoidable. When you write one:

- **Isolate it** - Single file, clearly marked
- **Add a kill switch** - Environment variable to disable
- **Test for removal** - Version check that fails when you upgrade
- **Link to upstream** - Comment with issue/PR URL
- **Plan removal** - Document when/how to remove it

```ruby
# config/initializers/gem_workaround.rb
#
# WORKAROUND: Fixes X behavior in gem_name < 2.0
# See: https://github.com/owner/gem_name/issues/123
# Remove when: gem_name >= 2.0 (PR #456 merged)
#
# Toggle: DISABLE_GEM_WORKAROUND=1 to disable

return if ENV["DISABLE_GEM_WORKAROUND"]
return if Gem::Version.new(GemName::VERSION) >= Gem::Version.new("2.0")

# ... minimal patch code ...
```

The two guard lines are the removal plan. The environment variable disables the patch for one run, and the version check retires it once the fixed release is installed.

## Real Example: Path Resolution Bug

This is the bug from the opening. Reading the source instead of guessing at path manipulation took six commands:

```bash
# 1. Locate gem
bundle show swarm_sdk
# => /path/to/gems/swarm_sdk-2.0.6

# 2. Open and search
bundle open swarm_sdk
# Search for file loading logic
# Found: paths resolved relative to config file directory

# 3. Add debug output
# puts "Loading agent from: #{resolved_path}"

# 4. Run test
bin/rails test test/models/workflow_test.rb
# Output reveals: Loading from wrong directory

# 5. Fix: Use correct relative path in config

# 6. Restore gem
bundle pristine swarm_sdk
```

The gem resolved paths relative to the config file's directory, so the fix was a corrected relative path in our config. Fixed in 10 minutes by reading source. No workaround needed.

## Results

- The path bug was a usage error, fixed in 10 minutes with a config change and no patch.
- The cost is a 25-minute time box before any workaround, and `bundle pristine` after every investigation.
- Fewer workarounds and monkey patches to maintain, and fixes that go upstream instead of staying local.

## Lessons Learned

- **Most "bugs" are usage errors.** Read the library's source before assuming it is broken; the fix is usually on your side of the call.
- **25 minutes of research is cheaper than any workaround.** Issues, releases, then source, then decide.
- **Contribute over patch.** An upstream fix helps everyone and never needs removing.
- **Give every workaround an exit.** A version guard that retires it once the fix ships beats a comment asking someone to remember.

---

## How This Post Was Made

**Prompt:** "let's write (one or more) posts about the skills we have in helloweather web and ios... in web we have debugging and dependency research, which I think might be a good one to share."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Combines the debugging and dependency-research skills from helloweather/web into a unified workflow.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the path-resolution bug instead of the reader's situation, the pattern overview no longer restates the section headings, the workaround code gets a note on the version guard, and Results and Lessons Learned hold only what the body does not already say. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
