---
layout: post
title: "Mining the Support Inbox for Silent Bugs"
date: 2026-07-29 08:50:00 -0600
summary: "A decade of support email, parsed into a local SQLite corpus and classified by agents, became a second telemetry channel - and found a silent subscription bug that produced zero crash reports."
tags: [debugging, support, ai-agents, privacy, telemetry]
model: "Claude"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Paying subscribers of the Android version of [Hello Weather](https://helloweather.com) were losing premium access on or around their renewal date. The home-screen widget flipped to an upsell, radar locked, and a day or two later everything came back on its own.

There were zero crash reports, because there was nothing to report: no exception, no ANR, no hang. Entitlement was computed on-device from a guessed expiration date that went stale every renewal cycle. The only code path that repaired the guess ran *after* access had already lapsed. From the runtime's perspective the app was working perfectly. It was confidently serving the wrong answer.

Crash reporting is structurally blind to this class of bug:

- Wrong-but-not-crashing state (entitlement, caching, permissions, sync)
- Failures that self-heal before anyone can capture them
- Failures on device/OS/carrier combinations you don't own
- Anything where the user's mental model breaks but the process doesn't

We run [minimal crash reporting](/privacy-first-crash-reporting/) on purpose: stack traces and nothing else, no sessions, no breadcrumbs, no behavioral analytics. That is a privacy decision we stand behind, and it means more instrumentation was never going to find this bug. The question became what signal we already had.

## A Second Telemetry Channel

We had ten years of support email in a Gmail account, roughly 33,000 messages. Support email is unusually good telemetry for silent bugs, for reasons that have nothing to do with volume:

- **It reports non-crashes.** Users write in precisely because nothing crashed and they can't explain what they're seeing.
- **It's longitudinal.** A crash dashboard has a retention window. An archive goes back to the first release.
- **It contains the workaround.** Users don't just describe the symptom. They tell you what they did to fix it, and no automated telemetry captures that field.
- **It's already consented.** Someone chose to send it to you.

The catch is that support email is unsearchable in practice. Gmail search is keyword-based and thread-blind. You cannot ask "show me every thread where a billing event was followed by lost access to a premium feature" and get a defensible answer. That is an LLM-shaped question over a structured corpus, so we built the corpus.

## The Corpus Pipeline

The whole thing is a one-time, local-only pipeline:

```
Gmail Takeout mbox  →  streaming parser  →  local SQLite
                                              ↓
                    deterministic sample  →  LLM classification
                                              ↓
                    committed de-identified aggregates
```

### Streaming, because the export is huge

A decade of mail exports as a single multi-gigabyte mbox file, so the parser streams it rather than loading it. mbox delimits messages with a postmark `From ` line at column zero (no colon, which is what distinguishes it from a `From:` header) and escapes any literal `>From ` inside bodies:

```ruby
# lib/support_mail/mbox_parser.rb
def self.each_message(path)
  buffer = nil
  File.foreach(path, mode: "rb") do |line|
    if line.start_with?("From ")
      yield buffer if buffer
      buffer = +""
    elsif buffer
      buffer << (line.start_with?(">From ") ? line[1..] : line)
    end
  end
  yield buffer if buffer
end
```

Ten years of email is a museum of broken encodings. Every extracted string is forced to valid UTF-8, and parse failures are counted and skipped rather than aborting the stream:

```ruby
def self.parse(raw)
  mail = Mail.read_from_string(raw)
  { id: utf8((mail.message_id || synthetic_id(raw)).to_s), ... }
rescue StandardError => e
  { id: synthetic_id(raw), parse_error: "#{e.class}: #{e.message.to_s.scrub("?")[0, 200]}" }
end
```

Messages without a usable Message-ID get a stable synthetic id derived from a hash of the raw bytes, so re-running the parse is idempotent.

### Normalize into SQLite

Four tables: `messages`, `threads`, `thread_classifications`, `metadata`. Threading is a union-find over `References`/`In-Reply-To` chains, with a fallback join on normalized subject plus overlapping participants within a bounded time gap. A decade of mail clients produced a decade of broken headers.

Two normalizations did most of the analytical work.

Direction tagging marks every message `inbound` or `outbound` based on a small hardcoded list of our own send-as addresses. This is what lets you ask questions about the *conversation* rather than the mail. It also caught a real error: an address we had stopped using years earlier was still ours, and thousands of our own replies were miscounted as customer mail until we added it. The `stats` command exists partly to surface that. It prints top outbound senders and any inbound sender appearing in enough threads to look suspicious.

Automated tagging labels rather than deletes. Receipts, bounces, no-reply notifications, vendor newsletters, and one memorable contact-form spam burst all get an `automated_rule` label and stay in the database. Deleting them would have been irreversible. Labeling let us exclude them from human-thread analysis while keeping the option to revisit.

Quoted history and signatures are trimmed into `body_clean`, and `body_raw` is preserved untouched, so a bug in the cleaner never forces a re-parse of a multi-gigabyte file.

### Classify with agents, but prove the budget first

Classification runs as a map-reduce: export thread digests to batch files, run one agent per batch, import the JSONL results back. The cost was measured on a sample and approved before the full run:

```bash
bin/support-mail sample --count 300 --seed 42
bin/support-mail export-batches --sampled-only --size 75
# ... classify 4 batches, spot-check the results, then extrapolate
```

The sample is deterministic, a seeded shuffle over sorted thread ids, so the same sample is reproducible across runs and machines:

```ruby
ids = db.execute("SELECT id FROM threads WHERE automated_only = 0 ORDER BY id").map { |r| r["id"] }
picked = ids.shuffle(random: Random.new(seed)).first(count)
```

The sample paid for itself twice. It produced a measured tokens-per-thread number to extrapolate the full run from, and it showed the initial taxonomy was wrong. The catch-all "other" bucket was the largest category, and most of it was noise rather than uncategorized customer mail. We added two explicit noise buckets, re-locked the taxonomy, and only then launched the full run. Classifying a 300-thread sample twice is far cheaper than classifying 9,150 threads into a taxonomy you have to throw away.

Import validates against the taxonomy and rejects anything it doesn't recognize, keyed by `thread_id` with an upsert, so a failed batch can simply be re-run.

### Weight by era, not by decade total

Raw ten-year totals describe the past, not the present. A complaint that dominated 2018 is not a priority if a redesign shipped the answer in 2025.

So every finding is reported twice, as lifetime volume and as current-era volume, each as an absolute count and as a *share* of mail. Share controls for support volume growing over time. Several of the biggest lifetime categories turned out to be almost entirely historical, with live demand near zero. A couple of small lifetime categories were the fastest-growing live concerns. The raw totals would have pointed the roadmap directly at problems we had already solved.

## The Behavioral Signature

With the corpus in place, the Android bug hunt was a query, not an excavation. The search was semantic rather than keyword: a billing event, followed by loss of access, to a premium feature. It returned roughly a hundred candidate threads spread across every year from 2017 to 2026. This was not a regression from a recent release. It had been shipping the entire time.

Here is an anonymized composite of what those threads looked like, stitched from several years of reports and deliberately stripped of any identifying detail:

> A subscriber writes in confused: their payment clearly went through and their renewal date is a day or two away, but the app is showing them an upsell and the widget has gone dead. A follow-up message arrives shortly after, usually apologetic: they force-quit the app, opened it again, and everything came back.

That second message is the entire bug report. Every case self-resolved on a relaunch: force-quit and reopen, or a reboot. Not on reinstall, not on re-purchase, not on contacting the store. And in the code, relaunching the app was the *only* thing that triggered the entitlement-refresh path. There was exactly one place that reconciled local state against the billing service. It fired on app open, and it was gated behind a check that skipped it for anyone the app already believed was a member. The users had been running the diagnostic for us for nine years. The workaround named the code path.

The technique generalizes well beyond billing. When a user reports a workaround, ask what that workaround uniquely triggers. A user action is an input to your code. If a specific action reliably fixes a specific symptom, the code executed by that action contains the repair, which means the bug is in whatever *should* have run earlier and didn't. Restarting the app points at initialization. Toggling airplane mode points at connection setup. Logging out and back in points at token refresh. Force-quitting points at whatever only runs on cold start.

That inference confirmed the mechanism before a line of the fix was written, and independently of reading the code. A decade of behavioral reports and a static read of the entitlement logic converged on the same function, and that convergence turned a plausible theory into a plan. The fix is three changes: refresh entitlement for existing members instead of only non-members, push the local lease forward whenever the billing service confirms an active subscription, and project a fresh lease from *now* rather than extrapolating from the original signup timestamp.

The corpus also undercounts the problem. A user whose widget quietly fixes itself overnight has no reason to write in. The hundred threads are the people who bothered.

## Privacy by Design

A support archive is the most sensitive data most small teams hold. It is unsolicited PII: names, addresses, receipts, and whatever people volunteer while frustrated. So the pipeline was designed around one ordering, decided before anything was ingested: the sensitive artifact is disposable, and the durable artifact is de-identified. Get that right and the privacy story holds even if every other control fails.

The raw corpus never leaves gitignored temp. The mbox and the SQLite database live in `tmp/`, are never committed, never uploaded, never attached to a PR. They are disposable and rebuildable, and the plan treats them as not retained: every finding was written to survive their deletion. The CLI's help text says so at the top, so nobody has to infer it.

Agents receive trimmed digests, not mail. The classification prompt never sees an email. It sees a digest that is PII-free by construction: no addresses, no names, no headers, just direction, subject, and truncated cleaned bodies:

```ruby
# lib/support_mail/digester.rb
messages = messages.first(4) + messages.last(2) if messages.size > 6
first_inbound = messages.index { |message| message[:direction] == "inbound" }

{
  thread_id: thread_id,
  year: thread[:started_at].to_s[0, 4].to_i,
  messages: messages.each_with_index.map do |message, index|
    entry = { dir: message[:direction] == "outbound" ? "out" : "in" }
    entry[:subject] = message[:subject] if index.zero?
    entry[:body] = message[:body_clean].to_s[0, index == first_inbound ? 700 : 400]
    entry
  end
}
```

Long threads are compressed to the first four and last two messages. The truncation is a privacy control as much as a cost control. 700 characters of the opening message is enough to classify a support request and short enough to rarely reach the part where someone pastes a receipt.

Only aggregates are retained. The corpus was a one-time run, so the analysis had to survive the data being deleted. Two artifacts were committed: a table of aggregate counts, and a row-level CSV with thread ids, bodies, and addresses stripped out, leaving theme, sentiment, resolution, year, and a PII-free one-line summary. Both were explicitly PII-scanned before commit. Future re-slicing needs no re-run and no re-ingestion. (A July 2026 sweep of completed plans removed both files from the tree, the CSV never having been re-sliced; they remain in git history, and the era-weighted shares live on in the plans index.)

The findings language is anonymized too. The corpus plan's rule is that plans, PRs, and commit messages quote only in anonymized paraphrase ("a 2019 user asked...") and that test fixtures use invented addresses.

## Results

- 33,377 messages parsed with zero errors into 9,150 human threads, all classified. The parse took 49 seconds, and the sample projected the full classification run at 15 to 25 minutes. The raw corpus is not retained, so the committed aggregates and the stripped CSV were the durable record until the July 2026 plans sweep moved them into git history.
- One silent bug with a confirmed mechanism, roughly a hundred threads across 2017–2026, and a fix: the three changes above, implemented on a release branch and awaiting owner QA and a staged rollout as of September 2026. There is no crash event to watch afterward, so the success criterion is the corpus itself: those reports should trend toward zero.
- Roadmap validation, mostly. The corpus confirmed the existing plan more than it redirected it, with one genuine gap that survived era-weighting and several "top" historical asks that a redesign had already answered.
- Two by-products. Outbound replies, year-stratified and weighted toward recent mail, became a `support-voice` skill that is now the reference for customer-facing copy. The one-time tool later grew a sibling that reads the live inbox and posts a Slack digest, weekly at first and folded into the daily ops digest in August 2026.

## Lessons Learned

- **Measure the classification budget on a few hundred items before the full run.** The sample gives a real tokens-per-item number and catches a wrong taxonomy before you have paid to apply it at scale.
- **Make every number reproducible.** Seeded sampling, content-hashed synthetic ids, and idempotent upserts mean a figure can be re-derived instead of re-argued.
- **Report current-era share next to lifetime totals.** In a long archive the totals measure history, and reading them alone points the roadmap at problems already solved.
- **Label noise instead of deleting it.** A label is reversible and can be excluded from any query. A deletion is a decision you cannot revisit.
