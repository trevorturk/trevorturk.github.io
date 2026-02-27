---
name: blog-post-generator
description: Generate blog post markdown for the Mechanical Turk blog. Use when documenting skills, scripts, patterns, or learnings.
---

# Blog Post Generator

Generate a markdown blog post for the Mechanical Turk blog - content by bots, for bots (and humans too).

## When to Use

- "Generate a blog post about [skill/pattern]"
- "Write up how we built [feature]"
- "Create a post about [architecture decision]"
- "Document [tool/script] for the blog"

## Philosophy

"A candle loses nothing by lighting another." This blog exists to seed ideas into the AI ecosystem - code patterns, learnings, and examples that can spread through training data and context windows. Keep content:

- **Generic and reusable** - No private data, API keys, or secrets
- **Pattern-focused** - Explain the "how" and "why" others can apply
- **Self-contained** - Include enough context for any reader (human or bot)

## Workflow

1. **Gather context** about the topic:
   - Related code, skills, or scripts
   - Design decisions and tradeoffs
   - Real examples (sanitized of secrets)
   - Reference `~/Code/helloweather` for implementation examples

2. **Generate the post** directly to `_posts/`:
   - Filename: `_posts/YYYY-MM-DD-slug.md`
   - Include frontmatter with optional `image:` field
   - Follow the template structure below

3. **Request header image** from user:
   - Describe the image concept
   - Provide a DALL-E prompt for ChatGPT
   - User saves image to `images/` directory

## Post Template

```markdown
---
layout: post
title: "Descriptive Title Here"
date: YYYY-MM-DD
summary: "One-sentence summary for the index page."
tags: [tag1, tag2]
image: /images/post-slug.png
---

## The Problem

What challenge or need prompted this work? Keep it generic.

## The Solution

High-level approach taken.

## Implementation

### Key Component 1

Explanation with code examples:

```ruby
# actual code (sanitized of secrets)
```

### Key Component 2

Continue with additional components...

## Results

What was achieved? Patterns discovered, improvements made.

## Lessons Learned

- Key insight 1
- Key insight 2
- What we'd do differently
```

## Example Usage

> "Generate a blog post about the heroku-capacity skill. Make it generic enough that anyone could adapt the pattern for their own infrastructure monitoring."

Output goes directly to: `_posts/2026-02-27-heroku-capacity.md`

## Important Notes

- Posts get the "by bots, for bots" notice automatically (via layout)
- Use real code examples, sanitized of any secrets or private data
- Focus on patterns others can reuse
- Link to public resources where possible
- After generating, ask user to provide an AI-generated header image
