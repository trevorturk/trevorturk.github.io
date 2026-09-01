---
layout: post
title: "Closing the Loop on Translation Bugs"
date: 2026-08-24 09:40:00 -0600
summary: "After shipping 27 languages nobody on the team can proofread, the customers are the QA wave. A feedback button, a computed inbox, a grep-by-value fix, and a human send gate make the loop from bug report to shipped fix to reply cheap on both ends and clean on privacy."
tags: [workflow, ai-agents, localization, i18n]
---

## The Problem

A model-translated string fails in a way only a native speaker sees. "Clear" comes back as transparent where the forecast meant cloudless. A button verb lands in the wrong register. A unit abbreviation reads as a typo. Each string is plausible on its own and wrong in context, and [Hello Weather](https://helloweather.com) has thousands of them in 27 languages. The one person who runs the product reads a handful of those languages. He will not catch any of these, because he cannot read the screen.

The people who can are the customers using the app in that language every day. They are the real QA wave. But a native speaker noticing a bad string does nothing on its own. The report has to survive four handoffs, and each one quietly drops reports:

1. The customer has to report the error without friction, and the report has to carry enough context to act on. If they must name the screen, in English, from memory, most just close the app.
2. The report has to land somewhere you actually look, get flagged as needing a reply, and know how to route the reply back.
3. You have to find the offending string among thousands of keys, fix it correctly, and not destroy the other 26 translations doing it.
4. You have to reply in the customer's language, which you cannot read, without sending a machine-drafted sentence you never fully saw.

This post walks the four links. It is a sequel to [how the support inbox became our localization telemetry](/support-inbox-as-telemetry/). That post is why we invested in translation quality; this one is the machinery that turns one bug report into a shipped fix.

## The Solution

Four links, each cheap on both ends. A human sits at the two points where judgment matters, the fix and the send. Everything else is plumbing.

### Link 1: A feedback button that pre-fills the report

The customer knows the language and you know the code, and the report has to bridge the two. "A word looks wrong somewhere in settings" cannot be acted on. So the entry point is a card at the top of the Language screen that builds a pre-addressed email, with the routing facts already in the subject line. Here is the builder and a minimal card that opens it, with a fallback for a device with no mail app configured:

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

The subject line carries three facts and no more. The language display name (`Deutsch`) is human-scannable in a triage queue. The language code (`de`) is the column you will grep in the catalog. The app version tells you whether the string still exists in current code. The `language` passed in is the one the user currently has selected, so the report is always about the language they are looking at. When `openURL` cannot hand off to a mail composer, the card offers to copy the address rather than dead-ending.

What the subject omits matters as much as what it carries. No device model, OS version, coordinates, or account identifier. A translation bug is a property of a string and a language, so none of the usual diagnostic telemetry is attached. Nothing sensitive can leak because nothing sensitive was collected.

### Link 2: An inbox with no status field

Triage rots because someone has to keep a status field honest, and nobody does. A "needs reply" flag gets set and forgotten. A thread that got a second customer message still shows answered. So we do not store the status at all. The customer is waiting when their last message is newer than ours. That is a fact about two timestamps, and we derive it every time it is asked.

Two decisions carry this link, and both are small enough to show in full: the waiting predicate and the sender rewrite, with a test that pins the behavior.

```ruby
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

Run that file and the three tests pass. The same predicate appears once more as the SQL the list query filters on, so the CLI and the in-memory check agree by construction. If a customer replies again, the timestamp moves and the thread reappears in the queue on its own. A computed queue cannot drift.

The Reply-To rewrite fixes a specific trap. Reports sent through the website's contact form arrive from our own transactional sender, with the actual customer in the `Reply-To` header. Triage naively on `From` and every one of those threads looks like it came from us. It never flags as waiting, and a reply routes back to ourselves. The substitution at sync time re-attributes the thread to whoever actually wrote it. The local cache is full of customer email, so it lives in a gitignored scratch directory and is never committed.

### Link 3: Grep the value to find the key

The customer quotes a bad string in their language, but the catalog is keyed by English, so you cannot search for what they sent. The move is to grep the catalog for the translated *value*. Every language's value lives under the one English key, so a hit on the German string lands you inside the English entry, with all 27 translations side by side.

Here is the sequence a maintainer runs, from syncing the inbox to sending the reply:

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

The correction itself is a one-row edit inside that entry: fix the value and set its state.

```json
"de" : {
  "stringUnit" : {
    "state" : "translated",
    "value" : "der korrigierte Wert"
  }
}
```

The `state` line is the part that bites. A row left `new` is treated as unfinished and skipped at runtime, so the code's default English renders instead of your correction. A fix with the wrong state ships nothing, and this holds even for an English override row. There is also no key-level fallback: a language missing a row for a shipped key renders the raw English key on screen. So a correction is a good moment to check the whole 27-language set, because fixing one language often surfaces a different one quietly missing the same string.

That final `git diff` guards a String Catalog hazard. A build regenerates the catalog and can prune keys it cannot currently see in code, taking their translations with them. The count of deleted `"value"` lines should be zero. A non-zero count means the build is subtracting translations, and you restore the file rather than commit it. Treat the catalog as a file the build can silently shrink, and diff it like one.

### Link 4: Draft several, show the whole text, send only what was picked

The send needs a human because the owner is replying in a language he cannot read, and no automated check can tell him a sentence is right. So the policy is to reply in the customer's language with an English gloss in the draft, and to put a person between the model and Send who sees every word. This is convention, not a language detector. The model drafts, a human approves, the CLI sends. The approval shape is what makes it safe:

> The model presents at least three meaningfully different drafts. Each option carries the complete reply text, prefixed "Full reply:", with an English gloss for non-English drafts. The owner picks one. The CLI sends only the selected text, verbatim.

The full text lives inside each option because previews truncate, and a truncated preview of a sentence you cannot read is not an approval. The owner sees every word that will be sent, in both languages, before choosing. Once chosen, `reply --archive` sends that exact string and archives the thread.

One caveat. The `reply` command is allowlisted, so an agent can invoke it with no per-send permission prompt, and nothing at the tool level forces the text through the selection step. That is a convention, not a technical block, and we took the trade on purpose. A hard prompt on every send would only repeat what the selection already established, since by then the owner has read the full text and picked it. A redundant prompt trains him to click through prompts, which is worse for safety than the convention it replaces.

### The matching rule on the public side: declared is not launched

A sibling rule governs App Store review replies. An app can *declare* 27 languages in its store metadata long before those languages *launch* in a shipping build, and both the store's language list and the app's declared localizations report the aspiration, not the reality. So a language is not claimed live until it is. Until a translation has shipped, the only sanctioned forward-looking phrasing is that translations are in progress. A public promise that a language works when it does not is one you walk back a single one-star review at a time. (More on treating [store listings as ASO rather than translation](/store-listings-as-aso-not-translation/).)

## Results

- A bug report becomes a fix in one sitting. The subject line carries language, code, and version; the code leads to the catalog key; one row and a rebuild ships the correction to everyone. No screen-naming, no repro hunt, no diagnostic data to collect or justify.
- Triage no longer needs tending. "Waiting" is derived from two timestamps, so the queue cannot fall out of sync, and web-form mail routes to the real customer instead of looping back to us.
- The cost is a send gate that is convention rather than tooling. The `reply` command runs without a per-send prompt, and the only thing between a draft and a customer is that the owner read the full text and picked it.

## Lessons Learned

- **Collect only the identifier you will act on.** A report that carries the exact key to grep, and no telemetry, is cheaper to send and has nothing to justify or leak.
- **Derive any status a human would have to keep honest.** If the fact is already in timestamps you record, compute it on every read; a stored flag drifts the first time someone forgets it.
- **A file a build regenerates is a file a build can shrink.** Diff the generated artifact after every rebuild and treat deletions as a stop, not a cleanup.
- **Put the gate where the judgment happens, and nowhere else.** A prompt that repeats a choice already made teaches the human to click through prompts.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the kind of error a model makes and the owner cannot see rather than on how much text the app carries, the title is shortened, the spaced-hyphen clauses are split into sentences, Results names the allowlisted send as the cost, and Lessons Learned holds four rules that do not repeat a heading. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
