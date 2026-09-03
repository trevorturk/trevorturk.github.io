---
layout: post
title: "Debugging Dependencies: Research Before Workarounds"
date: 2026-03-04 09:00:00 -0600
summary: "Two habits for third-party code: read the library's source before guessing what it does, and check upstream before writing a workaround."
tags: [ruby, debugging, dependencies, patterns]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Our app couldn't find its agent files. The paths it was building were wrong, and the code that built them lived in a third-party gem, not in our app. The tempting fix was to guess what the gem was doing to the path and work around it in our own code, or to monkey patch the gem (override one of its methods from our app).

Either way we'd be building on a guess. When a library "isn't working", we're usually calling it wrong. Sometimes the feature we want exists under a different name. Sometimes the bug is already fixed upstream, and a workaround we write today would still be there long after the fix ships.

## Two Complementary Patterns

A dependency lets you down in two ways, and each gets its own habit. When something doesn't work, read the source before guessing what it does. When something is missing, check upstream before writing a workaround.

## Pattern 1: Investigate Source

Most package managers can tell you where a dependency's source lives and open it in your editor. In Ruby:

```bash
# Find where the gem is installed
bundle show gem_name
# => /path/to/gems/gem_name-1.2.3

# Open it in your editor
bundle open gem_name
```

Other ecosystems have equivalents:
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

**Step 3:** Add debug statements if reading isn't enough
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

The debug lines you added to the gem are temporary. Put the gem back after every investigation, and don't commit a gem you've edited:
```bash
bundle pristine gem_name  # Restore single gem
bundle pristine           # Restore all gems
```

## Pattern 2: Research Before Patching

When the library is missing something you need, spend 25 minutes looking before you write a workaround. A workaround written first tends to stay in place long after upstream fixes the same thing.

### The Research Protocol

**GitHub Issues (5-10 min)**
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

Pick the first of these that fits what you found:

1. **Use existing** - Feature exists, you missed it
2. **Contribute upstream** - Feature missing, maintainer active
3. **Wait for PR** - Open PR implements it, looks likely to merge; test it from a Gemfile branch
4. **Temporary workaround** - Last resort only

### If You Must Workaround

Sometimes there's no way around writing one. When that happens:

- **Isolate it** - Single initializer, clearly marked
- **Test for removal** - Version check test that fails when you upgrade
- **Link to upstream** - Comment with issue/PR URL
- **Plan removal** - Document when/how to remove it

```ruby
# config/initializers/gem_name_fixes.rb
# Temporary fix for gem_name bug
# Upstream PR: https://github.com/author/gem-name/pull/123
# Remove when gem-name >= vX.Y.Z
Rails.application.config.to_prepare do
  require "gem_name"

  module GemName
    class BuggyClass
      def buggy_method
        # Fixed implementation
      end
    end
  end
end
```

```ruby
# test/models/some_test.rb
test "gem_name version check for monkey patch" do
  current_version = Gem.loaded_specs["gem_name"]&.version&.to_s

  assert_equal "1.2.3", current_version,
    "gem_name version changed to #{current_version}. " \
    "Check if monkey patch still needed (PR #123)."
end
```

The test is how the monkey patch gets removed. It passes today and fails the first time the gem is upgraded, so whoever upgrades has to check whether the patch is still needed. Without it, the patch would keep overriding the gem's method after upstream had fixed the bug.

## Real Example: Path Resolution Bug

This is the bug from the opening. Reading the source instead of guessing took six steps:

```bash
# 1. Locate gem
bundle show swarm_sdk
# => /path/to/gems/swarm_sdk-2.0.6

# 2. Open and search
bundle open swarm_sdk
# Search for file loading logic
# Found: SwarmSDK::Swarm.load
# Found: agent_file paths resolved relative to the config file's directory

# 3. Add debug output in the gem
# puts "Loading agent from: #{resolved_path}"

# 4. Run test
bin/rails test test/models/workflow_test.rb
# Loading agent from: app/agents/workflows/app/agents/math/coordinator.md (WRONG)

# 5. Fix: agent_file "../math/coordinator.md", not "app/agents/math/coordinator.md"

# 6. Restore gem
bundle pristine swarm_sdk
```

The gem resolves `agent_file` paths relative to the directory the config file is in. We had written a path relative to the project root, so the gem put the config directory in front of it and got the doubled path in the debug output. The fix was one corrected relative path in our config. No workaround.

## Results

- The path bug was on our side. One config change fixed it, with no workaround.
- The cost is 25 minutes of research before any workaround, and a `bundle pristine` after every investigation.
- We maintain fewer workarounds and monkey patches, and fixes go upstream instead of staying in our app.

## Lessons Learned

- **Most "bugs" are usage errors.** Read the library's source before deciding it's broken. The fix is usually on your side of the call.
- **25 minutes of research is cheaper than any workaround.** Check issues, then releases, then source, then decide.
- **Contribute upstream instead of patching.** An upstream fix helps everyone, and there's nothing to remove later.
- **Give every workaround an exit.** A version check test that fails on the next upgrade works. A comment asking someone to remember doesn't.
