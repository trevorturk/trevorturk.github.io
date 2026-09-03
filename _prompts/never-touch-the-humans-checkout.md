---
layout: prompts
title: "Never Touch the Human's Checkout"
post_url: /never-touch-the-humans-checkout/
post_date: 2026-08-25
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

**Prompt 1:** "please kick off a big batch to look through all skills looking for other topics that might be interesting to blog about. we could look at git history, but I think since we've been using claude/codex for the last ~year we should have most of the interesting stuff built into the skills by now. however, you can also look at the changelog view in the iOS repo for other highlights that might be worth dispatching research about. come back to me with a list of possible topics (that haven't already been covered in the blog) …"

**Prompt 2:** "lets do 4, 20, 21, 22 -- the others I think are not worth it"

Ten Claude agents mined the iOS, web, and Android skills, the iOS changelog, and the plan indexes for uncovered topics; the owner picked four from the ranked list. This post was researched and drafted by one agent from the cited skills, plans, and code, under the why-then-how voice and self-contained-code brief settled in the previous localization batch, then reviewed before publishing.

**Rewrite (2026-09-01):** The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" After a first line edit they followed up with "this looks better, but I wonder if you wanted to go farther with a more holistic rewrite instead of an editing pass?" and "agreed on the rewrite on this same post first, also holistically review and update the relevant skills etc". Claude Fable 5.1 rewrote the prose: the post now opens on the incidents, each section says its part once, Results is what changed and what it cost, and Lessons Learned holds only what the body does not already say. Code blocks, dates, and numbers are unchanged, and no facts were added. This was the pilot for a pass over the whole archive, and the standard is recorded in the blog-post-generator skill.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The PreToolUse hook never shipped: the 2026-08-04 PR tried one and dropped it in review, so the post now says the rule is enforced by AGENTS.md wording, and the quoted rule is AGENTS.md's current text rather than the pull-requests skill's paraphrase. The QA checklist was expanded to the skill's current six items, the build and cleanup scripts are labeled as trimmed from the real ones, the DerivedData cost now gives both figures in AGENTS.md (~6 GB per worktree, ~8 GB for a private agent cache), and the "structurally impossible" result is dated to August 2026.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 4, run after batch 3 (#70) and the prompts move (#71) merged. This post is the skill's reference post and got the same pass as the rest: contractions and "we" where a person would say them, cleft constructions and sentences that restated the one before cut, "PreToolUse hook" and "content-addressed" defined at first use, one name each for the downloaded packages and the compile cache, and the summary rewritten the same way. Judgment calls: "the human" stays as the name for the person because the title and the fixed checklist use it; "The last item looks trivial" now reads "Item 4", since the `git pull` rule became item 4 when the fact check expanded the checklist to six items; and the "mental model flips" figure and the "two worlds touch" closer were dropped. Code blocks, the quoted rule, the checklist, headings, dates, and numbers are unchanged. Prompts, verbatim:

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
