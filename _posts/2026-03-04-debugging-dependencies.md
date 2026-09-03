---
layout: post
title: "Debugging Dependencies: Research Before Workarounds"
date: 2026-03-04 09:00:00 -0600
summary: "Two patterns for working with third-party code: investigate source before guessing, and research upstream before patching."
tags: [ruby, debugging, dependencies, patterns]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
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

Choose based on what you found, in this order:

1. **Use existing** - Feature exists, you missed it
2. **Contribute upstream** - Feature missing, maintainer active
3. **Wait for PR** - Open PR implements it, looks likely to merge; test it from a Gemfile branch
4. **Temporary workaround** - Last resort only

### If You Must Workaround

Some workarounds are unavoidable. When you write one:

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

The test is the removal plan. It passes today and fails on the first gem upgrade, so the patch cannot be forgotten and cannot silently override the upstream fix.

## Real Example: Path Resolution Bug

This is the bug from the opening. Reading the source instead of guessing at path manipulation took six steps:

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

The gem resolved `agent_file` paths relative to the config file's directory, so a project-relative path got the config directory prepended. The fix was a corrected relative path in our config. No workaround needed.

## Results

- The path bug was a usage error, fixed with a config change and no patch.
- The cost is a 25-minute time box before any workaround, and `bundle pristine` after every investigation.
- Fewer workarounds and monkey patches to maintain, and fixes that go upstream instead of staying local.

## Lessons Learned

- **Most "bugs" are usage errors.** Read the library's source before assuming it is broken; the fix is usually on your side of the call.
- **25 minutes of research is cheaper than any workaround.** Issues, releases, then source, then decide.
- **Contribute over patch.** An upstream fix helps everyone and never needs removing.
- **Give every workaround an exit.** A version check test that fails on the next upgrade beats a comment asking someone to remember.
