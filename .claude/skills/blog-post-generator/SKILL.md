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

Most posts are about architectural work: a pattern chosen, a solution found, a simplification made, a workflow adopted. Some are failure stories. A few are how-tos, design records, or essays. The rules below fit all of them. They were settled on 2026-09-01 by surveying the archive and rewriting every post; the reference post is `_posts/2026-08-25-never-touch-the-humans-checkout.md`.

**The opening is the most specific true thing.** The first paragraph names what prompted the work, in terms the reader can picture. Depending on the post, that is:

- a constraint or number the design had to answer ("Every 30 minutes, thousands of devices wake up")
- the gap between what the platform gives and what the product needs ("Most vendors return sunrise and sunset. The app shows six more fields.")
- the plausible approach and the one specific way it falls short
- a design question the codebase was answering badly ("Which one goes first?")
- a cost that stopped being acceptable
- a failure, if there was one

"Why now" is mandatory. "What broke" is not. Never open on company framing ("Hello Weather carries more text than a typical weather app"), a generic industry observation ("X is surprisingly complex", "When building an application that..."), or the reader's situation ("You're using a third-party library. Something isn't working."). Product context (one person, 27 languages, a model doing the translating) appears once, after the opening, and only where it changes a decision.

**Headings are a default, not a mandate.** Problem / Solution / Results / Lessons Learned is the shape for work that has a before and an after. A tour of a system takes one h2 per component. A how-to keeps its steps. An essay names its h2s after its ideas. Whatever the shape, each h2 has one job and nothing is said in two of them.

**Each mechanism section runs why, mechanism, code, limit.** The alternative rejected and the reason, in a sentence or two. The mechanism. The code. Then the one thing the code cannot enforce, if there is one. Reasons live next to the mechanism they justify; a separate "Why This Works" section that restates the Solution is the pattern to avoid in new posts.

**After code, one paragraph on what to notice.** Name the non-obvious line. Never re-walk the control flow or list "key points". Use the word "notice" at most once per post.

**Results is what changed and what it cost.** Three or four bullets, or a short paragraph. Numbers, before/after, diff size, an accepted downside. A bullet that could sit in the Solution is a restatement; cut it. Where nothing was measured, say so, or state the trade-off accepted instead. Architecture posts name the trade-off: what got worse, and why that was acceptable.

**Lessons Learned is three to five rules that transfer.** Each bullet is something an agent in a different codebase could act on. Good forms: the decision criterion ("split state by whether it conflicts, not by what it is"), the boundary where the pattern stops applying ("if CPU-bound, threads win"), the downside accepted. Cut any bullet that matches a heading, a bolded rule in the body, or a Result. Test: remove the post's nouns; if nothing is left, cut it. Under about 25 words each. Cross-links belong in the body, not here.

**Sentences.** One idea per sentence, about 20 words. Split anything that chains clauses with semicolons or em-dashes. Vary openers; never start consecutive paragraphs or sections with the same construction ("Two things made this worse", "The first sentence... The second sentence..."). Bold lead-ins only on list items, never on paragraphs. No bolded "The rule:" closers at the end of a section; the rule is the section.

**Cut flourishes that carry no information.** These recur across the archive and go on sight: "load-bearing", "the key insight", "this is where the pattern shines", "the beauty of", "nightmare", "worth stealing", "the interesting part", "is the whole point", "Nobody wrote that", "draws blood", "the honest part", "Read that twice". One rhetorical device per section at most.

**Titles.** One clause, under about ten words. The detail goes in `summary:`, which the index shows anyway. "Never Touch the Human's Checkout" beats "Never Touch the Human's Checkout: Worktrees, DerivedData, and the QA Handoff for Agents Sharing a Repo".

**Length.** 1,500 to 2,200 words of prose is the usual range. A code tour may run longer in code, not in prose.

**Code.** Self-contained and runnable as shown, sanitized, with illustrative names. Trim to the lines that carry the pattern, and say so.

**Facts.** Every date, number, and claim comes from the source repos, skills, or plans. Say where a rule is enforced (a file, a hook) and when it landed if the history shows it. Caveats are facts too: "not yet in the serving path", "this is a design record, not a shipping report" stay.

## Workflow

1. **Gather context** about the topic:
   - Related code, skills, or scripts
   - Design decisions and tradeoffs, including the alternative rejected
   - Real examples (sanitized of secrets)
   - What prompted the work: the constraint, gap, question, cost, or failure the opening will name
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

The default shape. Rename or add h2s when the work has a different shape (see Headings above).

```markdown
---
layout: post
title: "Short Title"
date: YYYY-MM-DD HH:MM:SS -0600
summary: "One-sentence summary for the index page. This is where the detail the title left out goes."
tags: [tag1, tag2]
---

## The Problem

Open on the most specific true thing. Then the setup, then the general shape. Keep it generic.

## The Solution

Approach in a sentence or two. If there are three or more parts, list them:

- Part 1
- Part 2

### Part 1 as a heading

Why (the alternative rejected), then mechanism, then code, then the limit.

```ruby
# actual code (sanitized of secrets)
```

### Part 2 as a heading

Continue with additional parts...

## Results

What changed and what it cost. Three or four bullets.

## Lessons Learned

- Rules that transfer to another codebase
- What we'd do differently

---

## How This Post Was Made

**Prompt:** "[The prompt that generated this post]"

Generated by Claude using the blog-post-generator skill. [Brief note about context, iterations, or human edits if any.]
```

Some older posts use a separate `## Implementation` heading between The Solution and Results, or topical h2s instead of Solution. Keep those headings when revising them.

## Revising an Existing Post

For an editing pass or a rewrite of a published post. The archive-wide rewrite of 2026-09-01 followed these rules; the pilot commit on the worktrees post shows the before and after.

**May change:** prose, paragraph order, section length, the title, the summary, `###` headings, which points land in Results versus Lessons Learned, and hard-wrapped lines (unwrap them; the rest of the archive is one paragraph per line).

**Must not change:**

- Code blocks, byte for byte. Verify with a diff of the fenced blocks, not by eye.
- `##` headings, in text and order, so cross-links, summaries, and the index keep working. Do not add a Results or Lessons h2 to a post that has none; state what shipped and what it cost in the last body section instead.
- Dates, numbers, quoted text, tables, blockquotes, and links, including links to other posts.
- Facts. Reorganize what the drafting agent verified; do not add claims. If a nuance seems missing, check the source repo before writing it in. Caveats and admissions stay.
- Anonymization. If a post says "Vendor A", keep it. Do not add or remove product, vendor, or person names.
- The original prompts and process note in How This Post Was Made.

**Record the pass.** Append a dated **Rewrite (YYYY-MM-DD):** or **Editing pass (YYYY-MM-DD):** paragraph to How This Post Was Made with the prompts verbatim and one sentence on what changed in this post.

**Verify with `bin/verify-post _posts/<file>.md`.** It diffs the fenced blocks and `##` headings against `main` (ignoring `##` lines inside fences), checks the date, the original prompt lines, internal links, `{% raw %}` balance, and the Rewrite paragraph, and prints the prose word count before and after. Then `bundle exec jekyll build`. State both in the PR.

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
