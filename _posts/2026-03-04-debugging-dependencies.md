---
layout: post
title: "Debugging Dependencies: Research Before Workarounds"
date: 2026-03-04 09:00:00 -0600
summary: "Two patterns for working with third-party code: investigate source before guessing, and research upstream before patching."
tags: [ruby, debugging, dependencies, patterns]
---

## The Problem

You're using a third-party library. Something isn't working. You have two tempting options:

1. **Guess and patch** - Add a workaround based on what you think the library does
2. **Monkey patch** - Override the broken behavior in your codebase

Both create technical debt. The library probably works correctly - you're just using it wrong. Or the feature you need already exists under a different name. Or someone already fixed this upstream.

## Two Complementary Patterns

### Pattern 1: Investigate Source First

When something doesn't work, read the source code before guessing.

### Pattern 2: Research Upstream First

Before implementing a workaround, check if someone already solved your problem.

## Pattern 1: Investigate Source

Most package managers let you locate and read dependency source code. In Ruby:

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

**Step 3:** Add debug statements if needed
```ruby
# Temporary changes in gem code:
puts "DEBUG: path = #{path.inspect}"
binding.break  # Stop execution here
```

**Step 4:** Run your test to observe
```bash
bin/rails test test/models/example_test.rb
```

**Step 5:** Restore the dependency to original state
```bash
bundle pristine gem_name
```

### What You'll Often Find

- **Wrong usage** - The method expects different arguments than you're passing
- **Different naming** - The feature exists but is called something else
- **Configuration needed** - A setting enables the behavior you want
- **Version mismatch** - Your version doesn't have the feature yet

### Critical Rule

Always restore dependencies after debugging:
```bash
bundle pristine gem_name  # Restore single gem
bundle pristine           # Restore all gems
```

Never commit modified dependency source code.

## Pattern 2: Research Before Patching

Before implementing a workaround, spend 25 minutes researching. This prevents technical debt.

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

Choose the best approach based on what you found.

### Decision Priority

1. **Use existing** - Feature exists, you missed it
2. **Wait for PR** - Open PR implements it, looks likely to merge
3. **Contribute upstream** - Feature missing, maintainer active
4. **Temporary workaround** - Last resort only

### If You Must Workaround

Sometimes workarounds are unavoidable. When you implement one:

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

## Real Example: Path Resolution Bug

**Problem:** Agent files not found, paths malformed.

**Wrong approach:** Guess at path manipulation, add workarounds.

**Right approach:**

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

**Result:** Fixed in 10 minutes by reading source. No workaround needed.

## Results

With these patterns:
- Fewer workarounds and monkey patches
- Better understanding of dependencies
- Contributions to upstream projects
- Less technical debt to maintain

## Lessons Learned

- **Read source first** - Most "bugs" are usage errors
- **25 minutes saves hours** - Research prevents bad workarounds
- **Contribute over patch** - Upstream fixes benefit everyone
- **Restore always** - `bundle pristine` after every investigation
- **Document workarounds** - Future you will thank present you

---

## How This Post Was Made

**Prompt:** "let's write (one or more) posts about the skills we have in helloweather web and ios... in web we have debugging and dependency research, which I think might be a good one to share."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Combines the debugging and dependency-research skills from helloweather/web into a unified workflow.
