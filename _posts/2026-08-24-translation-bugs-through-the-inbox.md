---
layout: post
title: "Closing the Loop on Translation Bugs"
date: 2026-08-24 09:40:00 -0600
summary: "We ship 27 languages and can't proofread most of them, so the customers find the translation bugs. A feedback button, an inbox that computes who's waiting, a grep for the quoted string, and a person reading every draft before it's sent turn one bug report into a shipped fix and a reply."
tags: [workflow, ai-agents, localization, i18n]
---

## The Problem

A model translates a string and gets it wrong in a way only a native speaker would notice. "Clear" comes back meaning transparent when the forecast meant cloudless. A button's verb has the wrong tone. A unit abbreviation looks like a typo. Each string looks fine on its own and is wrong in context. [Hello Weather](https://helloweather.com) has thousands of strings in 27 languages, and the one person who runs it reads a handful of those languages. He won't catch these because he can't read the screen.

The people who can catch them are the customers who use the app in that language every day. They're our proofreaders. But a customer noticing a bad string doesn't fix anything by itself. The report has to get through four steps, and each one loses reports:

1. The customer has to send the report easily, and it has to carry enough for us to act on. If they have to name the screen, in English, from memory, most of them close the app instead.
2. The report has to land somewhere we look, show up as needing a reply, and carry the right address to reply to.
3. We have to find the bad string among thousands of keys and fix it without breaking the other 26 translations.
4. We have to reply in the customer's language, which we can't read, without sending a model-written sentence nobody looked at.

This post goes through the four steps. It follows [how the support inbox became our localization telemetry](/support-inbox-as-telemetry/). That post explains why we put work into translation quality. This one is the tooling that turns one bug report into a shipped fix.

## The Solution

Four steps, each cheap for the customer and for us. A person does the two parts that take judgment, the fix and the send. The rest is plumbing.

### Step 1: A feedback button that pre-fills the report

The customer knows the language and we know the code, and the report has to connect the two. We can't do anything with "a word looks wrong somewhere in settings". So the Language screen has a card at the top that opens a pre-addressed email with the language and app version already in the subject line. Here's the builder and a minimal card that opens it, with a fallback for a device that has no mail app set up:

```swift
import SwiftUI

struct Language {
    let code: String        // "de"
    let displayName: String // "Deutsch"
}

struct TranslationFeedback {
    static let address = "support@helloweather.com"

    static func mailtoURL(language: Language, appVersion: String) -> URL? {
        var components = URLComponents()
        components.scheme = "mailto"
        components.path = address
        components.queryItems = [
            URLQueryItem(
                name: "subject",
                value: "Translation feedback — \(language.displayName) (\(language.code)) — v\(appVersion)"
            ),
        ]
        return components.url
    }
}

struct TranslationFeedbackCard: View {
    let language: Language
    let appVersion: String

    @Environment(\.openURL) private var openURL
    @State private var showMailFallback = false

    var body: some View {
        Button {
            sendReport()
        } label: {
            Text("Notice any translation errors? Tap here to send us feedback.")
        }
        .alert("Couldn't open your email app.", isPresented: $showMailFallback) {
            Button("Copy Email Address") {
                UIPasteboard.general.string = TranslationFeedback.address
            }
            Button("Cancel", role: .cancel) {}
        } message: {
            Text(verbatim: TranslationFeedback.address)
        }
    }

    private func sendReport() {
        guard let url = TranslationFeedback.mailtoURL(language: language, appVersion: appVersion) else {
            showMailFallback = true
            return
        }
        openURL(url) { accepted in
            if !accepted { showMailFallback = true }
        }
    }
}
```

The subject line carries three facts. The language name (`Deutsch`) is easy to scan in a list of threads. The language code (`de`) is the row we'll grep for in the catalog. The app version tells us whether the string still exists in current code. The `language` passed in is the one the customer has selected, so the report is always about the language they're looking at. If `openURL` can't open a mail composer, the card offers to copy the address instead of doing nothing.

The subject leaves out the device model, OS version, location, and account. A translation bug is about a string and a language, so we don't attach the usual diagnostic data. Nothing sensitive can leak because we didn't collect it.

### Step 2: An inbox with no status field

Triage falls apart because someone has to keep a status field up to date, and nobody does. A "needs reply" flag gets set and forgotten. A thread where the customer wrote back still shows as answered. So we don't store a status at all. The customer is waiting when their last message is newer than ours. We compute that from the two timestamps every time we list the queue.

This step comes down to two small pieces of code: the waiting check and the sender rewrite. Here they are with a test:

```ruby
# Trimmed to the two addresses this post discusses; the real list also
# carries our other send-as aliases. Addresses are lowercased at parse time.
OUR_ADDRESSES = ["support@helloweather.com", "robots@helloweather.com"].freeze

def outbound?(email)
  OUR_ADDRESSES.include?(email.to_s.downcase)
end

# The customer had the last word: their newest message is newer than ours.
# Timestamps are ISO-8601 strings, so string comparison is chronological.
def waiting?(row)
  row[:last_inbound_at] &&
    (row[:last_outbound_at].nil? || row[:last_inbound_at] > row[:last_outbound_at])
end

# Web-form mail arrives From our own transactional address, with the real
# customer in Reply-To. Rewrite the sender so triage and replies see them.
def effective_from_email(parsed)
  reply_to = parsed[:reply_to_email].to_s
  return reply_to if outbound?(parsed[:from_email]) && !reply_to.empty? && !outbound?(reply_to)

  parsed[:from_email]
end

if __FILE__ == $PROGRAM_NAME
  require "minitest/autorun"

  class InboxLogicTest < Minitest::Test
    def test_customer_last_word_is_waiting
      assert waiting?(last_inbound_at: "2026-08-24T10:00:00Z",
                      last_outbound_at: "2026-08-23T09:00:00Z")
    end

    def test_we_replied_last_is_not_waiting
      refute waiting?(last_inbound_at: "2026-08-23T09:00:00Z",
                      last_outbound_at: "2026-08-24T10:00:00Z")
    end

    def test_web_form_sender_is_rewritten_to_customer
      parsed = { from_email: "support@helloweather.com",
                 reply_to_email: "reader@example.com" }
      assert_equal "reader@example.com", effective_from_email(parsed)
    end
  end
end
```

Run that file and the three tests pass. The same check appears once more as SQL in the list query, so the CLI and the Ruby version agree. If a customer replies again, their timestamp moves and the thread comes back into the queue on its own.

The Reply-To rewrite fixes one specific trap. Reports sent through the website's contact form arrive from our own sending address, with the real customer in the `Reply-To` header. If we went by `From`, every one of those threads would look like it came from us. It would never show as waiting, and a reply would go back to us. Swapping in the Reply-To address at sync time credits the thread to the person who wrote it. The local cache is full of customer email, so it lives in a gitignored scratch directory and never gets committed.

### Step 3: Grep the value to find the key

The customer quotes the bad string in their language, but the catalog is keyed by English. So we grep the catalog for the translated text and let the hit take us to the key. Every language's version sits under one English key, so a hit on the German string lands inside that key's entry, with all 27 translations together.

Here's the sequence we run, from syncing the inbox to sending the reply:

```bash
bin/gmail pull                                    # sync inbox into tmp/gmail.sqlite3
bin/gmail list --waiting --sort oldest            # triage queue, longest wait first
bin/gmail show THREAD_ID                           # cleaned bodies, flags screenshots
bin/gmail attachments THREAD_ID                    # pull the screenshot, if any

# The customer quoted a German string; the catalog is keyed by English,
# so search for the value to reach the key:
grep -n "der falsche Wert" \
  HelloWeather/HelloWeather/Resources/Localizable.xcstrings

# ...correct that key's "de" row (next block), rebuild, then vet the catalog:
git diff -- Localizable.xcstrings | grep -c '^-.*"value"'   # expect 0

bin/gmail reply THREAD_ID --message "…" --archive  # send the picked draft, archive
```

The fix is a one-row edit inside that entry: change the value and set its state.

```json
"de" : {
  "stringUnit" : {
    "state" : "translated",
    "value" : "der korrigierte Wert"
  }
}
```

The `state` line is the one that catches people. We learned this on an English override row: it was left as `new`, the app skipped it at runtime, and the default in code showed instead of the fix. A fix with the wrong state ships nothing, so every corrected row gets `translated`. There's also no fallback per key. A language that's missing a row for a shipped key shows the raw English key on screen. So a fix is a good time to check all 27 languages for that key, because fixing one language often turns up another that's missing the same string.

The `git diff` at the end guards against a String Catalog problem. A build regenerates the catalog and can drop keys it can't see in code at the moment, and their translations go with them. The count of deleted `"value"` lines should be zero. If it isn't, the build removed translations, and we restore the file instead of committing it.

### Step 4: Draft several, show the whole text, send only what was picked

The send needs a person because the owner is replying in a language he can't read, and no automated check can tell him whether a sentence is right. So the rule is to reply in the customer's language with an English gloss (a plain translation of the draft) alongside it, and to have a person read every word before it's sent. This is a convention, not something the tool detects. The model drafts, the owner approves, the CLI sends. The approval works like this:

> The model presents at least three meaningfully different drafts. Each option carries the complete reply text, prefixed "Full reply:", with an English gloss for non-English drafts. The owner picks one. The CLI sends only the selected text, verbatim.

Each option includes the full text because previews get cut off, and a cut-off preview of a sentence you can't read isn't an approval. The owner sees every word that will go out, in both languages, before choosing. Then `reply --archive` sends that exact text and archives the thread.

One caveat. The `reply` command is allowlisted, so an agent can run it without a permission prompt on each send, and nothing in the tool forces the text through the selection step. That's a convention, not a technical block, and we chose it on purpose. A prompt on every send would only repeat what the selection already did, because by then the owner has read the full text and picked it. A prompt that repeats a choice teaches him to click through prompts, and that's worse for safety than the convention.

### The same rule for App Store replies: listed is not shipped

A related rule covers replies to App Store reviews. An app can list 27 languages in its store metadata long before those languages ship in a build, and both the store's language list and the app's declared localizations show what's planned, not what works. So we don't say a language is live until it is. Until a translation has shipped, the only thing we say about the future is that translations are in progress. If we say a language works and it doesn't, we get one-star reviews about it. (More on treating [store listings as ASO rather than translation](/store-listings-as-aso-not-translation/).)

## Results

- A bug report becomes a fix in one sitting. The subject line carries the language, its code, and the app version. The quoted string leads to the catalog key and the language code to the row. One row and a rebuild ship the fix to everyone. The customer doesn't name a screen, we don't hunt for a repro, and there's no diagnostic data to collect.
- Triage needs no upkeep. "Waiting" comes from two timestamps, so the queue can't get out of sync, and web-form mail goes to the real customer instead of back to us.
- The cost is that the send gate is a convention, not tooling. The `reply` command runs without a prompt on each send, and the only thing between a draft and a customer is that the owner read the full text and picked it.

## Lessons Learned

- **Collect only what you'll act on.** A report that carries the language code and app version, and no telemetry, is easier to send and has nothing in it to leak.
- **Compute any status a person would have to keep up to date.** If the answer is already in timestamps you record, work it out on every read. A stored flag goes stale the first time someone forgets it.
- **A file the build regenerates is a file the build can shrink.** Diff it after every rebuild and treat deleted lines as a reason to stop, not as cleanup.
- **Put the confirmation where the judgment happens, and nowhere else.** A prompt that repeats a choice already made teaches the person to click through prompts.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the kind of error a model makes and the owner cannot see rather than on how much text the app carries, the title is shortened, the spaced-hyphen clauses are split into sentences, Results names the allowlisted send as the cost, and Lessons Learned holds four rules that do not repeat a heading. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The Ruby excerpt now says its address list is trimmed and that lowercasing happens at parse time, since the real constant carries more send-as aliases than the two shown. The String Catalog `state` paragraph was narrowed to what the source records: the runtime skip of a `new` row was observed on an English override row, not asserted for every language. The Swift excerpt, the `waiting?` and `effective_from_email` code, the SQL twin of the predicate, the `bin/gmail` commands, the allowlist, the three-drafts rule, and the 27-language count all matched.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 2, run after batch 1 (#68) merged. The post now says "we" and "can't" where it said "the team" and "cannot", the four handoffs/links are called steps throughout (the ### headings too), "gloss" is defined at first use, "surfaces" became "turns up", and the closing lines on the computed queue, the shrinking catalog, and the one-star reviews were cut or made literal. One judgment call: "the wrong register" for a button verb became "the wrong tone", the ordinary word for the same thing. Code blocks, ## headings, numbers, links, the blockquote, and the earlier meta paragraphs are unchanged. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"
