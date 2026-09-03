---
layout: prompts
title: "Delete the Fiction Your Agents Believe"
post_url: /agent-doc-audit/
post_date: 2026-07-29
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on three of the findings instead of a general observation about documentation rot, the Tier 1 catalogue is grouped under six subheadings, the follow-up PR that had been a lesson moved to Results, and Lessons Learned is down from eight rules to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The Results table now measures the whole audit for every repo (web 45,459 → 31,140, iOS 30,535 → 20,437, counted the same way as Android) where it had used Tier 3-only figures for web and iOS, and the "quarter to a third" framing became "a third to nearly half"; the iOS frontmatter cost was overstated (about 1,100 words in total and ~90 for the longest description, not 4,000 and ~120). The billing tier labels, the billing and watch and Swift-conventions quotes, and the deprecated-API excerpt (a Swift code block in the real skill, not a table) now match the source verbatim; the frontmatter-less skill count is three (one per repo), the Target SDK deadline is one generation stale and ten and a half months past, the self-escalation fix is credited to the one repo that made it, and the misattributed PR examples are described by their real iOS subjects.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 4, run after batch 3 (#70) and the prompts move (#71) merged. Prose rewritten paragraph by paragraph from an ELI5 of the post: first person and contractions throughout, cleft constructions and nominalizations removed, restating sentences cut, "surfaced" replaced with "reported", and the front-matter summary rewritten in the same register. Judgment calls: the rhetorical "the same truth stated in four places, drifting apart at four different rates" became "several places" because the example that follows it has three copies, and "a case-sensitive checkout, Linux CI or a case-sensitive volume" kept only the Linux CI example. Code blocks, headings, dates, numbers, quotes, tables, blockquotes, and links are unchanged. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"

**Prompt 10:** "usage is fine, please continue -- one more thing -- at the end (or perhaps with future batches?) I'd like to change the "How This Post Was Made" sections in all posts to not have the prompt in the post itself, rather, the prompts should be moved into PR body if editable, or comments, then the "How This Post Was Made" can have the last edit date and a link to the Pull Requests / Prompts -- then there's less cruft at the end for readers that just want to copy paste a post into their agent -- wdyt?"

**Prompt 11:** "sounds good"

**Prompt 12:** "for the "How This Post Was Made" maybe we move the date down there as well, so it's like first posted by MODEL on DATE, last edited by MODEL on DATE, View: PROMPTS, HISTORY or something along those lines, so it's all in one place?"

**Prompt 13:** "agreed, go"
