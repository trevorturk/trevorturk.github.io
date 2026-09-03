---
layout: prompts
title: "Never Touch the Human's Checkout"
post_url: /never-touch-the-humans-checkout/
post_date: 2026-08-25
model: "Claude"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

**Prompt 1:** "please kick off a big batch to look through all skills looking for other topics that might be interesting to blog about. we could look at git history, but I think since we've been using claude/codex for the last ~year we should have most of the interesting stuff built into the skills by now. however, you can also look at the changelog view in the iOS repo for other highlights that might be worth dispatching research about. come back to me with a list of possible topics (that haven't already been covered in the blog) …"

**Prompt 2:** "lets do 4, 20, 21, 22 -- the others I think are not worth it"

Ten Claude agents mined the iOS, web, and Android skills, the iOS changelog, and the plan indexes for uncovered topics; the owner picked four from the ranked list. This post was researched and drafted by one agent from the cited skills, plans, and code, under the why-then-how voice and self-contained-code brief settled in the previous localization batch, then reviewed before publishing.

**Rewrite (2026-09-01):** The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" After a first line edit they followed up with "this looks better, but I wonder if you wanted to go farther with a more holistic rewrite instead of an editing pass?" and "agreed on the rewrite on this same post first, also holistically review and update the relevant skills etc". Claude Fable 5.1 rewrote the prose: the post now opens on the incidents, each section says its part once, Results is what changed and what it cost, and Lessons Learned holds only what the body does not already say. Code blocks, dates, and numbers are unchanged, and no facts were added. This was the pilot for a pass over the whole archive, and the standard is recorded in the blog-post-generator skill.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The PreToolUse hook never shipped: the 2026-08-04 PR tried one and dropped it in review, so the post now says the rule is enforced by AGENTS.md wording, and the quoted rule is AGENTS.md's current text rather than the pull-requests skill's paraphrase. The QA checklist was expanded to the skill's current six items, the build and cleanup scripts are labeled as trimmed from the real ones, the DerivedData cost now gives both figures in AGENTS.md (~6 GB per worktree, ~8 GB for a private agent cache), and the "structurally impossible" result is dated to August 2026.
