---
name: pull-requests
description: GitHub pull request patterns. Use when creating PRs or updating PR descriptions.
---

# Pull Requests

Patterns for creating clear, transparent pull requests.

## When to Use

- Creating a new pull request
- Updating a PR description
- Reviewing PR conventions

## PR Template

```markdown
## Summary
- Bullet points describing what changed

## Prompts Used

**Prompt 1:** "[First prompt that initiated this work]"

**Prompt 2:** "[Follow-up prompts, if any]"

## Test plan
- [ ] Verification steps

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Requirements

1. **Always include prompts** - Every PR must include a "Prompts Used" section with the prompts that led to the changes
2. **Use bullet summaries** - Keep the summary concise with bullet points
3. **Add test plan** - Include verification steps as a checklist

## Why Include Prompts?

Following the "Post Your Prompts" philosophy:
- Transparency over optimization
- The raw prompt is often more valuable than polished output
- Helps others understand intent and reproduce similar work
- Documents the human-AI collaboration

## Example

```markdown
## Summary
- Add CI workflow for Jekyll build verification
- Add Gemfile.lock for reproducible builds

## Prompts Used

**Prompt 1:** "can you add some basic tests and/or a ci check? nothing too crazy, but enough to verify that jekyll will be able to build"

**Prompt 2:** "CI failed, investigate and fix"

## Test plan
- [ ] CI passes on this PR
- [ ] Local build works with `bundle exec jekyll build`

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

## Important Notes

- Capture prompts as you go - don't wait until PR creation
- Include follow-up prompts that led to additional commits
- Keep prompt text verbatim when possible
