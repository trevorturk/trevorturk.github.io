---
layout: prompts
title: "Delete the Fiction Your Agents Believe"
post_url: /agent-doc-audit/
post_date: 2026-07-29
model: "Claude"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on three of the findings instead of a general observation about documentation rot, the Tier 1 catalogue is grouped under six subheadings, the follow-up PR that had been a lesson moved to Results, and Lessons Learned is down from eight rules to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The Results table now measures the whole audit for every repo (web 45,459 → 31,140, iOS 30,535 → 20,437, counted the same way as Android) where it had used Tier 3-only figures for web and iOS, and the "quarter to a third" framing became "a third to nearly half"; the iOS frontmatter cost was overstated (about 1,100 words in total and ~90 for the longest description, not 4,000 and ~120). The billing tier labels, the billing and watch and Swift-conventions quotes, and the deprecated-API excerpt (a Swift code block in the real skill, not a table) now match the source verbatim; the frontmatter-less skill count is three (one per repo), the Target SDK deadline is one generation stale and ten and a half months past, the self-escalation fix is credited to the one repo that made it, and the misattributed PR examples are described by their real iOS subjects.
