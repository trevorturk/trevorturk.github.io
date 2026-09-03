---
layout: prompts
title: "Mining the Support Inbox for Silent Bugs"
post_url: /support-inbox-as-telemetry/
post_date: 2026-07-29
model: "Claude"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and plan docs, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the subscription bug itself, the three-part fix moved out of Results into the section that found the mechanism, Results is the counts and what remains after the corpus was deleted, and Lessons Learned went from eight bullets to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The classification wall-clock figure was an estimate from the sample, not a measurement, and now says so with the parse time given exactly (49 seconds); the raw corpus is described as not retained rather than deleted, and the committed aggregates and CSV are noted as swept out of the plans tree in July 2026. The fix is now described as implemented on a branch and awaiting release QA as of September 2026, the weekly inbox digest as folded into the daily digest in August 2026, and the anonymization rule in its source wording. The digester excerpt gained the two lines that define `messages` and `first_inbound`.
