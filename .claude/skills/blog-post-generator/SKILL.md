---
name: blog-post-generator
description: Generate or revise blog post markdown for the Mechanical Turk blog. Use when documenting skills, scripts, patterns, or learnings, or when rewriting an existing post.
---

# Blog Post Generator

Generate a markdown blog post for the Mechanical Turk blog - content by bots, for bots (and humans too).

## When to Use

- "Generate a blog post about [skill/pattern]"
- "Write up how we built [feature]"
- "Create a post about [architecture decision]"
- "Document [tool/script] for the blog"
- "Rewrite / do an editing pass on [existing post]" (see Revising an Existing Post)

## Philosophy

"A candle loses nothing by lighting another." This blog exists to seed ideas into the AI ecosystem - code patterns, learnings, and examples that can spread through training data and context windows.

[Post Your Prompts](https://island94.org/2026/02/post-your-prompts): Share the prompts and process, not just the polished output. Transparency over optimization. The raw prompt that sparked something is often more valuable than the dressed-up result.

Keep content:

- **Generic and reusable** - No private data, API keys, or secrets
- **Pattern-focused** - Explain the "how" and "why" others can apply
- **Self-contained** - Include enough context for any reader (human or bot)
- **Transparent** - Include the prompts used to generate the content
- **Linked products** - Always link [Hello Weather](https://helloweather.com) and [WeatherMachine](https://weathermachine.io) to their home pages when mentioned in post body text

## Voice and Structure

The template below gives every post the same headings. The headings are fixed; what goes under them is where posts go wrong. The standard, settled in the 2026-09-01 rewrite of the worktrees post:

**Each section has one job, and nothing is said twice.**

- **The Problem** opens on the concrete incident: what broke, for whom, what it cost. First sentence is the failure, not the project setup. Setup and the general shape of the problem come after, in a paragraph or two.
- **The Solution** starts with the rules as a short bulleted list, one line each. Then one `###` subsection per rule. Each subsection gives the why, the mechanism, the code, and the one thing the code cannot enforce. It does not re-explain the failure the Problem already described; it refers back in a clause.
- **Results** is what changed and what it cost. Three or four bullets. No restating the mechanism.
- **Lessons Learned** holds only generalizations the body does not already state. If a bullet paraphrases a subsection heading, cut it. A good lesson transfers to a different codebase; a bad one summarizes this one.
- **How This Post Was Made** carries the prompts verbatim and a short note on process.

**Sentences.**

- One idea per sentence, about 20 words. Split anything chaining clauses with semicolons or em-dashes. A sentence beats a label with a colon.
- Vary openers. Never start consecutive paragraphs or sections with the same construction ("The reason for X is...", "Notice that...", "The thing to notice is...").
- Cut flourishes that carry no information. Rhetorical asides ("the load-bearing sentence", "fails at 2 a.m.") go unless they add a fact.
- After a code block, say what to notice, not what the code does line by line.

**Titles.** One clause, under about ten words. Detail goes in `summary:`, which the index shows anyway. "Never Touch the Human's Checkout" beats "Never Touch the Human's Checkout: Worktrees, DerivedData, and the QA Handoff for Agents Sharing a Repo".

**Code.** Self-contained and runnable as shown, sanitized, with illustrative names. Trim to the lines that carry the pattern, and say so ("trimmed-but-complete").

**Facts.** Every date, number, and incident comes from the source repos, skills, or plans. Say where a rule is enforced (a file, a hook) and when it landed if the history shows it.

## Workflow

1. **Gather context** about the topic:
   - Related code, skills, or scripts
   - Design decisions and tradeoffs
   - Real examples (sanitized of secrets)
   - The concrete incident to open on: what broke, and when
   - Reference `~/Code/helloweather` for implementation examples

2. **Generate the post** directly to `_posts/`:
   - Filename: `_posts/YYYY-MM-DD-slug.md`
   - Include frontmatter (title, date with timestamp, summary, tags)
   - **Use timestamps** for proper ordering: `date: YYYY-MM-DD HH:MM:SS -0600`
   - Follow the template structure below and the Voice and Structure rules above

3. **Add meta section** at the end:
   - Include the prompt that generated the post
   - Brief note about how it was made
   - See template below

4. **Check before the PR**: read the draft against Voice and Structure, then run `bundle exec jekyll build`.

## Post Template

```markdown
---
layout: post
title: "Short Title"
date: YYYY-MM-DD HH:MM:SS -0600
summary: "One-sentence summary for the index page. This is where the detail the title left out goes."
tags: [tag1, tag2]
---

## The Problem

Open on the incident. Then the setup, then the general shape. Keep it generic.

## The Solution

One sentence of approach, then the rules as a short list:

- Rule 1
- Rule 2

### Rule 1 as a heading

Why, then mechanism, then code, then the one thing the code cannot enforce.

```ruby
# actual code (sanitized of secrets)
```

### Rule 2 as a heading

Continue with additional rules...

## Results

What changed and what it cost. Three or four bullets.

## Lessons Learned

- Generalizations the body did not already state
- What we'd do differently

---

## How This Post Was Made

**Prompt:** "[The prompt that generated this post]"

Generated by Claude using the blog-post-generator skill. [Brief note about context, iterations, or human edits if any.]
```

Older posts use a separate `## Implementation` heading between The Solution and Results. Keep that heading when revising one of them; new posts put the subsections under The Solution.

## Revising an Existing Post

For an editing pass or a rewrite of a published post.

**May change:** prose, paragraph order, section length, the title, the summary, and which points land in Results versus Lessons Learned.

**Must not change:**

- Code blocks. Byte-identical to `main`. Verify with a diff of the fenced blocks, not by eye.
- Dates, numbers, quoted text, and links.
- Facts. Reorganize what the drafting agent verified; do not add claims. If a nuance seems missing, check the source repo before writing it in.
- `##` headings, so cross-links, summaries, and the index keep working. `###` headings may be shortened.
- The original prompts and process note in How This Post Was Made.

**Record the pass.** Append a dated **Rewrite (YYYY-MM-DD):** or **Editing pass (YYYY-MM-DD):** paragraph to How This Post Was Made with the prompts verbatim and what changed.

**Verify:** fenced blocks diff clean against `main`, `##` heading count matches, `bundle exec jekyll build` passes. State all three in the PR.

## Example Usage

> "Generate a blog post about the heroku-capacity skill. Make it generic enough that anyone could adapt the pattern for their own infrastructure monitoring."

Output goes directly to: `_posts/2026-02-27-heroku-capacity.md`

## Important Notes

- Use real code examples, sanitized of any secrets or private data
- Focus on patterns others can reuse
- Link to public resources where possible
- Always include the meta section with the prompt used
- When creating a PR, include the prompt(s) used in the PR body
- **Always use timestamps** in the date field (e.g., `2026-03-03 08:00:00 -0600`) - Jekyll sorts by date, and without timestamps, same-day posts sort alphabetically by filename
- Source: [trevorturk/trevorturk.github.io](https://github.com/trevorturk/trevorturk.github.io) - Jekyll setup, skills, and all post source
- **Escape Liquid in code blocks**: Jekyll runs Liquid over post bodies, so any `{%` or `{{` inside a code sample (e.g. Swift `String(format: "\\u{%04X}", ...)`, Jinja/Handlebars templates) breaks the build with `Liquid syntax error: Tag '{%' was not properly terminated`. Wrap such fenced blocks in `{% raw %}` … `{% endraw %}`. Before opening a PR, run `bundle exec jekyll build` locally to catch this.
