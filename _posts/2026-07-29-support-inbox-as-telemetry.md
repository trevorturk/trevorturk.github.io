---
layout: post
title: "Mining the Support Inbox for Silent Bugs"
date: 2026-07-29 08:50:00 -0600
summary: "We parsed ten years of support email into a local SQLite database, had agents classify the threads, and found a subscription bug that never produced a crash report."
tags: [debugging, support, ai-agents, privacy, telemetry]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Paying subscribers on the Android version of [Hello Weather](https://helloweather.com) were losing premium access around their renewal date. The home-screen widget switched to an upsell, radar locked, and a day or two later everything came back on its own.

There were no crash reports, because nothing crashed: no exception, no ANR (Android's "app not responding" dialog), no hang. Entitlement, meaning whether the app treats you as a paying member, was worked out on the device from a guessed expiration date. That guess went stale every time the subscription renewed. The only code that corrected it ran after access had already lapsed. As far as the runtime could tell, the app was working fine. It was just giving the wrong answer.

Crash reporting can't see this kind of bug at all:

- State that's wrong but doesn't crash, like a subscription status or a cache
- Failures that fix themselves before anyone can capture them
- Failures on devices, OS versions, or carriers you don't own
- Anything where the user is confused but the process is fine

We run [minimal crash reporting](/privacy-first-crash-reporting/) on purpose: stack traces and nothing else. No sessions, no breadcrumbs, no behavioral analytics. That's a privacy decision we stand behind, so adding more tracking wasn't an option. The question was what signal we already had.

## A Second Telemetry Channel

We had ten years of support email in a Gmail account, roughly 33,000 messages. Support email is good evidence for silent bugs, and not because there's a lot of it:

- **It reports things that didn't crash.** People write in because nothing crashed and they can't explain what they're seeing.
- **It goes back years.** A crash dashboard only keeps so much history. The archive goes back to the first release.
- **It includes the workaround.** People don't just describe the symptom. They tell you what they did to fix it, and no automated telemetry records that.
- **The sender consented.** They chose to write to you.

The catch is that you can't really search it. Gmail search matches keywords and doesn't understand threads. You can't ask "show me every thread where a payment was followed by losing a premium feature" and trust the answer. A language model can answer that question, but only over a corpus, meaning a cleaned-up copy of the mail in a database. So we built the corpus.

## The Corpus Pipeline

The pipeline runs once, on one machine:

```
Gmail Takeout mbox  →  streaming parser  →  local SQLite
                                              ↓
                    deterministic sample  →  LLM classification
                                              ↓
                    committed de-identified aggregates
```

### Streaming, because the export is huge

Ten years of mail exports as one mbox file several gigabytes long, so the parser reads it line by line instead of loading it into memory. In mbox, each message starts with a line that begins `From ` at column zero. There's no colon, which is how you tell it apart from a `From:` header. A body line that happens to start with `From ` is stored as `>From ` so it doesn't look like a new message, and the parser strips that `>` back off:

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

Ten years of email has every broken encoding you can think of. Every extracted string is forced to valid UTF-8, and a message that fails to parse is counted and skipped instead of stopping the run:

```ruby
def self.parse(raw)
  mail = Mail.read_from_string(raw)
  { id: utf8((mail.message_id || synthetic_id(raw)).to_s), ... }
rescue StandardError => e
  { id: synthetic_id(raw), parse_error: "#{e.class}: #{e.message.to_s.scrub("?")[0, 200]}" }
end
```

A message without a usable Message-ID gets a made-up id from a hash of its raw bytes, so re-running the parse produces the same ids every time.

### Normalize into SQLite

Four tables: `messages`, `threads`, `thread_classifications`, `metadata`. Messages are grouped into threads by following the `References` and `In-Reply-To` headers, using union-find so any two messages that share a chain land in the same thread. When those headers are missing or broken, which ten years of mail clients guarantees, the fallback groups messages with the same normalized subject and overlapping participants within a bounded time gap.

Two cleanup steps did most of the analytical work.

Direction tagging marks every message `inbound` or `outbound`, using a short hardcoded list of our own send-as addresses. With that tag you can ask about the conversation instead of the individual mail. It also caught a real mistake. An address we'd stopped using years earlier was still ours, and thousands of our own replies counted as customer mail until we added it to the list. The `stats` command exists partly to catch that. It prints the top outbound senders and any inbound sender that appears in enough threads to look suspicious.

Automated mail is labeled, not deleted. Receipts, bounces, no-reply notifications, newsletters, and one contact-form spam burst all get an `automated_rule` label and stay in the database. We can leave them out of any query about human threads and still go back to them later. Deleting them would have been permanent.

Quoted replies and signatures are trimmed off into `body_clean`, and `body_raw` keeps the original, so a bug in the cleaner doesn't mean re-parsing a multi-gigabyte file.

### Classify with agents, but prove the budget first

Classification works in batches: export thread digests to batch files, run one agent per batch, and import the JSONL results. We measured the cost on a sample and had it approved before the full run:

```bash
bin/support-mail sample --count 300 --seed 42
bin/support-mail export-batches --sampled-only --size 75
# ... classify 4 batches, spot-check the results, then extrapolate
```

The sample is a seeded shuffle over sorted thread ids, so the same seed gives the same sample on any machine:

```ruby
ids = db.execute("SELECT id FROM threads WHERE automated_only = 0 ORDER BY id").map { |r| r["id"] }
picked = ids.shuffle(random: Random.new(seed)).first(count)
```

The sample did two jobs. It gave us a measured tokens-per-thread number to estimate the full run from, and it showed our first set of categories was wrong. The catch-all "other" bucket was the biggest category, and most of what was in it was noise, not customer mail we'd failed to categorize. We added two explicit noise buckets, fixed the category list, and only then ran everything. Classifying a 300-thread sample twice is much cheaper than classifying 9,150 threads into categories you have to throw away.

Import checks each result against the category list and rejects anything it doesn't recognize. Results are upserted by `thread_id`, so a failed batch can just be re-run.

### Weight by era, not by decade total

Ten-year totals describe the past. A complaint that dominated 2018 isn't a priority if a redesign answered it in 2025.

So every finding is reported twice, once for the whole decade and once for the current era. Each is given as a count and as a share of all mail, because share corrects for support volume growing over the years. Several of the biggest lifetime categories turned out to be almost entirely historical, with almost no current demand. A couple of small lifetime categories were the fastest-growing current concerns. The raw totals would have pointed the roadmap at problems we'd already solved.

## The Behavioral Signature

With the corpus built, the Android bug hunt was one query. It was a search by meaning, not by keyword: a billing event, then loss of access, to a premium feature. It returned roughly a hundred candidate threads spread across every year from 2017 to 2026. So this wasn't a regression from a recent release. The bug had been in the app the whole time.

Here's an anonymized composite of what those threads looked like, stitched together from several years of reports with identifying details removed:

> A subscriber writes in confused: their payment clearly went through and their renewal date is a day or two away, but the app is showing them an upsell and the widget has gone dead. A follow-up message arrives shortly after, usually apologetic: they force-quit the app, opened it again, and everything came back.

That second message is the bug report. Every case cleared up after a relaunch, either force-quit and reopen or a reboot. Not after a reinstall, a re-purchase, or a call to the store. And in the code, relaunching the app was the only thing that ran the entitlement refresh. One place reconciled local state against the billing service. It ran on app open, behind a check that skipped it for anyone the app already believed was a member. Users had been running that diagnostic for us for nine years, and their workaround named the code path.

The same trick works well beyond billing. When a user reports a workaround, ask what code that workaround runs that nothing else does. If a specific action reliably fixes a specific symptom, the code behind that action contains the repair, so the bug is in whatever should have run earlier and didn't. Restarting the app points at initialization. Toggling airplane mode points at connection setup. Logging out and back in points at token refresh. Force-quitting points at whatever only runs on a cold start.

That reasoning confirmed the mechanism before we wrote any of the fix, and without depending on the code. Ten years of reports and a read of the entitlement logic both pointed at the same function, and that agreement turned a plausible theory into a plan. The fix is three changes. Refresh entitlement for existing members, not just non-members. Push the local expiration date forward whenever the billing service confirms an active subscription. And compute a fresh expiration date from now instead of extending from the original signup date.

The corpus also undercounts the problem. A user whose widget fixes itself overnight has no reason to write in. The hundred threads are only the people who bothered to write.

## Privacy by Design

A support archive is the most sensitive data most small teams hold. It's personal information (PII) nobody asked for: names, addresses, receipts, and whatever people volunteer while frustrated. So we settled one rule before ingesting anything: the sensitive copy is disposable, and the copy we keep has the personal information removed. If that holds, the privacy story holds even when another control fails.

The raw data never leaves a gitignored temp directory. The mbox and the SQLite database live in `tmp/`, and they're never committed, uploaded, or attached to a PR. They can be rebuilt from the export, and the plan treats them as not retained, so every finding was written to make sense after they're gone. The CLI's help text says so at the top.

Agents get trimmed digests, not mail. The classification prompt never sees an email. It sees a digest with no addresses, no names, and no headers, just direction, subject, and shortened cleaned bodies:

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

Long threads are cut down to the first four and last two messages. The truncation protects privacy as much as it saves tokens. 700 characters of the opening message is enough to classify a request and usually stops before the part where someone pastes a receipt.

Only summaries are kept. The corpus was a one-time run, so the analysis had to make sense after the data was deleted. We committed two files: a table of aggregate counts, and a row-level CSV with thread ids, bodies, and addresses stripped out. What's left in the CSV is theme, sentiment, resolution, year, and a one-line summary with no personal information. Both files were scanned for PII before commit, and any later re-slicing could use them without re-running the pipeline. (A July 2026 sweep of completed plans removed both files from the tree; the CSV was never re-sliced. They remain in git history, and the era-weighted shares live on in the plans index.)

The findings are anonymized too. The corpus plan's rule is that plans, PRs, and commit messages quote only in anonymized paraphrase ("a 2019 user asked...") and that test fixtures use invented addresses.

## Results

- 33,377 messages parsed with no errors into 9,150 human threads, all classified. The parse took 49 seconds, and the sample projected the full classification run at 15 to 25 minutes. The raw data isn't retained, so the committed counts and the stripped CSV were the record until the July 2026 plans sweep moved them into git history.
- One silent bug with a confirmed mechanism, roughly a hundred threads across 2017–2026, and a fix. The three changes above are on a release branch and waiting on owner QA and a staged rollout as of September 2026. There's no crash event to watch afterward, so the measure of success is the support inbox: those reports should fall toward zero.
- Mostly confirmation of the roadmap. The corpus backed the existing plan more than it changed it. One real gap held up after era-weighting, and several of the biggest historical asks had already been answered by a redesign.
- Two by-products. Our outbound replies, sampled by year and weighted toward recent mail, became a `support-voice` skill that's now the reference for customer-facing copy. The one-time tool later got a sibling that reads the live inbox and posts a Slack digest, weekly at first and folded into the daily ops digest in August 2026.

## Lessons Learned

- **Classify a few hundred items before the full run.** The sample gives you a real tokens-per-item number and catches a wrong set of categories before you've applied it to everything.
- **Make every number reproducible.** With seeded sampling, hashed ids, and upserts, you can recompute a figure instead of arguing about it.
- **Report current-era share next to lifetime totals.** In a long archive the totals measure history, and on their own they point the roadmap at problems you've already solved.
- **Label noise instead of deleting it.** You can exclude a label from any query and change your mind later. You can't undelete.
