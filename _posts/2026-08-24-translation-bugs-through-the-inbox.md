---
layout: post
title: "Closing the Loop on Translation Bugs: A Feedback Button, One Inbox, and a Human Send Gate"
date: 2026-08-24 09:40:00 -0600
summary: "After shipping 27 languages nobody on the team can proofread, your customers are the QA wave - so the loop from bug report to shipped fix to reply has to be cheap on both ends and clean on privacy."
tags: [workflow, ai-agents, localization, i18n]
---

## The Problem

[Hello Weather](https://helloweather.com) ships in 27 languages. One person runs the product, and he reads a handful of them. That gap is the whole problem: you can translate an app you cannot proofread, and a weather app built by a small team carries far more text than most - every stat label, forecast phrase, and settings row is copy, and the copy is part of the layout. When a model translates all of it, the strings come back individually plausible and occasionally wrong in context. "Clear" as in transparent where you meant cloudless. A button verb in the wrong register. A unit abbreviation that reads to a native speaker as a typo. You will not catch any of these, because you cannot read the screen.

The people who can are the ones using the app in that language every day. They are the real QA wave. But a native speaker noticing a bad string does nothing on its own; the report has to survive four handoffs, and each one quietly drops reports:

1. The customer has to report the error without friction, and the report has to carry enough context to act on. If they must name the screen, in English, from memory, most just close the app.
2. The report has to land somewhere you actually look, get flagged as needing a reply, and know how to route the reply back.
3. You have to find the offending string among thousands of keys, fix it correctly, and not destroy the other 26 translations doing it.
4. You have to reply in the customer's language - which you cannot read - without sending a machine-drafted sentence you never fully saw.

We built each link deliberately. This post is the whole loop, and it is a sequel to [how the support inbox became our localization telemetry](/support-inbox-as-telemetry/): that post is why we invested in translation quality, this one is the machinery that turns one bug report into a shipped fix.

## The Solution

Four links, each cheap on both ends, with a human at exactly the two points where judgment matters - the fix and the send - and boring plumbing everywhere else.

### Link 1: A feedback button, because a report you can't reproduce is a report you can't fix

The customer knows the language and you know the code, and the report has to bridge that. If it arrives as "a word looks wrong somewhere in settings," you cannot act. So the entry point is a card at the top of the Language screen that builds a pre-addressed email whose subject line already carries what you need to route and reproduce - the builder, and a minimal card that opens it with a fallback for a device with no mail app configured:

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

Notice the subject line is the entire design. It carries three facts and no more: the language display name (`Deutsch`, human-scannable in a triage queue), the language code (`de`, literally the column you will grep in the catalog), and the app version (which tells you whether the string still exists in current code). The `language` passed in is the one the user currently has selected, so the report is always about the language they are looking at. And when `openURL` cannot hand off to a mail composer, the card offers to copy the address rather than dead-ending.

What the subject omits matters as much as what it carries. No device model, no OS version, no coordinates, no account identifier. A translation bug is a property of a string and a language; none of the usual diagnostic telemetry is relevant, so none is attached. The report is privacy-clean by construction - nothing sensitive can leak because nothing sensitive was collected.

### Link 2: An inbox where "who is waiting" is a computation, not a label

The reason triage rots is that someone has to keep a status field honest, and nobody does. A "needs reply" flag gets set and forgotten; a thread that got a second customer message still shows answered. So we do not store the status at all. "Waiting" is a fact about two timestamps - the customer is waiting when their last message is newer than ours - and we derive it every time.

Two decisions carry that link, and both are small enough to show in full. Here is the waiting predicate and the sender-rewrite, with a test that pins the behavior:

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

Run that file and the three tests pass. The same predicate is expressed once more as the SQL the list query filters on, so the CLI and the in-memory check agree by construction. There are no labels and no flag to remember: if a customer replies again, the timestamp moves and the thread reappears in the queue on its own. A computed queue cannot drift.

The Reply-To rewrite fixes a specific trap. Reports sent through the website's contact form arrive from our own transactional sender, with the actual customer in the `Reply-To` header. Triage naively on `From` and every one of those threads looks like it came from us - it never flags as waiting, and a reply routes back to ourselves. The substitution at sync time re-attributes the thread to whoever actually wrote it. The local cache is full of customer email, so it lives in a gitignored scratch directory and is never committed.

### Link 3: Grep the value to find the key, fix one row, keep the all-languages discipline

The fix starts from a mismatch: the customer quotes a bad string in their language, but the catalog is keyed by English, so you cannot search for what they sent. The move that closes the gap is to grep the catalog for the translated *value* - because every language's value lives under the one English key, a hit on the German string lands you inside the English entry, where you read all 27 translations side by side.

Here is the sequence a maintainer actually runs, from syncing the inbox to sending the reply:

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

The `state` line is the part that bites. A row left `new` is treated as unfinished and skipped at runtime, so the code's default English renders instead of your correction - a fix with the wrong state ships nothing. This holds even for an English override row. And because there is no key-level fallback, a language missing a row for a shipped key renders the raw English key on screen, so a correction is a good moment to check the whole 27-language set - fixing one language often surfaces a different one quietly missing the same string.

That final `git diff` guards a String Catalog hazard: a build regenerates the catalog and can prune keys it cannot currently see in code, taking their translations with them. The count of deleted `"value"` lines should be zero; a non-zero count means the build is subtracting translations, and you restore the file rather than commit it. Treat the catalog as a file the build can silently shrink, and diff it like one.

### Link 4: Draft several, show the whole text, send only what was picked

The send needs a human because the owner is replying in a language he cannot read, and no automated check can tell him a sentence is right. So the policy is to reply in the customer's language with an English gloss in the draft, and to put a person between the model and Send who sees every word. This is convention, not a language detector - the model drafts, a human approves, the CLI sends. The approval shape is what makes it safe:

> The model presents at least three meaningfully different drafts. Each option carries the complete reply text, prefixed "Full reply:", with an English gloss for non-English drafts. The owner picks one. The CLI sends only the selected text, verbatim.

The full text has to live inside each option because previews truncate, and a truncated preview of a sentence you cannot read is not an approval. The owner sees every word that will be sent, in both languages, before choosing. Once chosen, `reply --archive` sends that exact string and archives the thread.

The honest caveat: the `reply` command is allowlisted, so an agent can invoke it with no per-send permission prompt, and nothing at the tool level forces the text through the selection step. That is a convention, not a technical block, and we took the trade on purpose. A hard prompt on every send would only repeat what the selection already established - by then the owner has read the full text and picked it - and a redundant prompt trains him to click through prompts, which is worse for safety than the convention it replaces. The gate is real because the selection is real.

### The matching rule on the public side: declared is not launched

A sibling honesty rule governs App Store review replies, and it exists because of one trap: an app can *declare* 27 languages in its store metadata long before those languages actually *launch* in a shipping build. Both the store's language list and the app's declared localizations report the aspiration, not the reality. So the rule is to not claim a language is live until it is; until a translation has shipped, the only sanctioned forward-looking phrasing is that translations are in progress. A public promise that a language works when it does not is one you walk back a single one-star review at a time. (More on treating [store listings as ASO rather than translation](/store-listings-as-aso-not-translation/).)

## Results

- **A bug report now becomes a fix in one sitting.** A customer's tap produces a subject line carrying the language, code, and version; the code leads straight to the catalog key; one row and a rebuild ships the correction to everyone. No screen-naming, no repro hunt, no diagnostic data to collect or justify.
- **Triage no longer needs tending.** "Waiting" is derived from two timestamps every time it is asked, so the queue cannot fall out of sync, and web-form mail routes to the real customer instead of looping back to us.
- **The send stays trustworthy in languages the owner can't read.** Every reply is a full draft he read in both languages before it went out - the loop is cheap without giving up the one judgment that has to stay human.

## Lessons Learned

- **Pre-encode the routing metadata and nothing else**, because the cheapest bug report arrives with the exact identifier you will act on and none of the data you would otherwise have to store and justify.
- **Make triage a computation, not a label**, because "who is waiting" is a fact about timestamps you already have, and a derived queue can never drift the way a hand-maintained status field does.
- **Grep the translated value to reach the English key, and respect the catalog's traps**: set the state to `translated` or the fix does not render, keep every language filled because there is no fallback, and diff the catalog after the build because regeneration can subtract silently.
- **Put a human between the model and Send and let them see the whole text**, because when you reply in a language you cannot read, the only real approval is one where every word was shown before it went out.
- **Do not claim launched until launched**, because declared languages are aspirations and shipped ones are facts, and the gap gets paid off in public.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
