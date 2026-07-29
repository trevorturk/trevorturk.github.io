---
layout: post
title: "Mining the Support Inbox for Bugs Your Crash Reporter Can't See"
date: 2026-07-29 08:50:00 -0600
summary: "A decade of support email, parsed into a local SQLite corpus and classified by agents, became a second telemetry channel - and found a silent subscription bug that produced zero crash reports."
tags: [debugging, support, ai-agents, privacy, telemetry]
---

## The Problem

A crash reporter tells you when your app dies. It tells you nothing when your app is alive, responsive, and simply *wrong*.

We found a subscription bug in the Android version of [Hello Weather](https://helloweather.com) that fit that description exactly. Paying subscribers were losing premium access on or around their renewal date. The home-screen widget flipped to an upsell, radar locked, and then - a day or two later - everything came back on its own.

Zero crash reports. Not one. There was nothing to report: no exception, no ANR, no hang. Entitlement was computed on-device from a guessed expiration date that went stale every renewal cycle, and the only code path that repaired the guess ran *after* access had already lapsed. From the runtime's perspective, the app was working perfectly. It was confidently serving the wrong answer.

This is the class of bug that crash reporting is structurally blind to:

- Wrong-but-not-crashing state (entitlement, caching, permissions, sync)
- Failures that self-heal before anyone can capture them
- Failures on device/OS/carrier combinations you don't own
- Anything where the user's mental model breaks but the process doesn't

We deliberately run [minimal crash reporting](/privacy-first-crash-reporting/) - stack traces and nothing else, no sessions, no breadcrumbs, no behavioral analytics. That's a privacy decision we stand behind, and it means we will never find this class of bug by adding more instrumentation. So the question became: what signal do we already have?

## A Second Telemetry Channel

We had ten years of support email sitting in a Gmail account. Roughly 33,000 messages.

Support email is unusually good telemetry for silent bugs, for reasons that have nothing to do with volume:

- **It's the only channel that reports non-crashes.** Users write in precisely because nothing crashed and they can't explain what they're seeing.
- **It's longitudinal.** A crash dashboard has a retention window. An archive goes back to the first release.
- **It contains the workaround.** Users don't just describe the symptom - they tell you what they did to fix it. That's the most valuable field in the message, and no automated telemetry captures it.
- **It's already consented.** Someone chose to send it to you.

The problem was always that support email is unsearchable in practice. Gmail search is keyword-based and thread-blind. You cannot ask "show me every thread where a billing event was followed by lost access to a premium feature" and get a defensible answer.

That's an LLM-shaped question over a structured corpus. So we built the corpus.

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

A decade of mail exports as a single multi-gigabyte mbox file. Never slurp it. mbox delimits messages with a postmark `From ` line at column zero - note the absent colon, which is what distinguishes it from a `From:` header - and escapes any literal `>From ` inside bodies:

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

Ten years of email is a museum of broken encodings. Every extracted string gets forced to valid UTF-8, and parse failures are counted and skipped rather than aborting the stream:

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

Four tables: `messages`, `threads`, `thread_classifications`, `metadata`. Threading is a union-find over `References`/`In-Reply-To` chains, with a fallback join on normalized subject plus overlapping participants within a bounded time gap, because a decade of mail clients produced a decade of broken headers.

Two normalizations did most of the analytical work:

**Direction tagging.** Every message is `inbound` or `outbound` based on a small hardcoded list of our own send-as addresses. This is what lets you ask questions about the *conversation* rather than the mail. It also caught a real error: an address we'd stopped using years earlier was still ours, and thousands of our own replies had been miscounted as customer mail until we added it. The `stats` command exists partly to surface that - it prints top outbound senders and any inbound sender appearing in enough threads to look suspicious.

**Automated tagging - tag, never delete.** Receipts, bounces, no-reply notifications, vendor newsletters, and one memorable contact-form spam burst all get an `automated_rule` label and stay in the database. Deleting them would have been irreversible; labeling them let us exclude them from human-thread analysis while keeping the option to revisit.

Quoted history and signatures get trimmed into `body_clean`, while `body_raw` is preserved untouched - so a bug in the cleaner never forces a re-parse of a multi-gigabyte file.

### Classify with agents, but prove the budget first

Classification runs as a map-reduce: export thread digests to batch files, run one agent per batch, import the JSONL results back.

The discipline that mattered: **the cost was measured on a sample and approved before the full run.**

```bash
bin/support-mail sample --count 300 --seed 42
bin/support-mail export-batches --sampled-only --size 75
# ... classify 4 batches, spot-check the results, then extrapolate
```

The sample is deterministic - a seeded shuffle over sorted thread ids - so the same sample is reproducible across runs and machines:

```ruby
ids = db.execute("SELECT id FROM threads WHERE automated_only = 0 ORDER BY id").map { |r| r["id"] }
picked = ids.shuffle(random: Random.new(seed)).first(count)
```

The sample paid for itself twice. It produced a measured tokens-per-thread number to extrapolate the full run against, and it revealed that the initial taxonomy was wrong: the catch-all "other" bucket was the largest category, and most of it was noise rather than uncategorized customer mail. We added two explicit noise buckets, re-locked the taxonomy, and only then launched the full run. Classifying a 300-thread sample twice is far cheaper than classifying 9,150 threads into a taxonomy you have to throw away.

Import validates against the taxonomy and rejects anything it doesn't recognize, keyed by `thread_id` with an upsert - so a failed batch can simply be re-run.

### Weight by era, not by decade total

The trap in any long archive: **raw ten-year totals describe the past, not the present.** A complaint that dominated 2018 is not a priority if a redesign shipped the answer in 2025.

So every finding is reported twice - lifetime volume and current-era volume, both as an absolute count and as a *share* of mail (share controls for support volume growing over time). Several of the biggest lifetime categories turned out to be almost entirely historical, with live demand near zero. A couple of small lifetime categories turned out to be the fastest-growing live concerns. Reading the raw totals would have pointed the roadmap directly at problems we had already solved.

## The Behavioral Signature

With the corpus in place, the Android bug hunt was a query, not an excavation.

The search was semantic, not keyword: *billing event, followed by loss of access, to a premium feature*. That returned roughly a hundred candidate threads - and critically, they were spread across every single year from 2017 to 2026. This was not a regression from a recent release. It had been shipping the entire time.

Then came the part that actually solved it.

Here is an anonymized composite of what those threads looked like, stitched from several years of reports and deliberately stripped of any identifying detail:

> A subscriber writes in confused: their payment clearly went through and their renewal date is a day or two away, but the app is showing them an upsell and the widget has gone dead. A follow-up message arrives shortly after, usually apologetic: they force-quit the app, opened it again, and everything came back.

That second message is the entire bug report.

**Every single case self-resolved on force-quit and reopen.** Not on reinstall, not on re-purchase, not on contacting the store - on relaunching the app. And in the code, relaunching the app was the *only* thing that triggered the entitlement-refresh path. There was exactly one place that reconciled local state against the billing service, it fired on app open, and it was gated behind a check that skipped it for anyone the app already believed was a member.

The users had been running the diagnostic for us for nine years. The workaround named the code path.

This is the transferable technique, and it generalizes well beyond billing:

**When a user reports a workaround, ask what that workaround uniquely triggers.** A user action is an input to your code. If a specific action reliably fixes a specific symptom, the code executed by that action contains the repair - which means the bug is in whatever *should* have run earlier and didn't. Restarting the app points at initialization. Toggling airplane mode points at connection setup. Logging out and back in points at token refresh. Force-quitting points at whatever only runs on cold start.

That inference confirmed the mechanism *before* a single line of the fix was written, and independently of reading the code. Two separate lines of evidence - a decade of behavioral reports and a static read of the entitlement logic - converged on the same function. That convergence is what turned a plausible theory into a plan.

The corpus also made clear that it *undercounts* the problem. A user whose widget quietly fixes itself overnight has no reason to write in. The hundred threads are the people who bothered.

## Privacy by Design

A support archive is the most sensitive data most small teams hold. It is unsolicited PII: names, addresses, receipts, and whatever people volunteer while frustrated. Building this pipeline meant deciding up front what would never leave the machine.

**The raw corpus never leaves gitignored temp.** The mbox and the SQLite database live in `tmp/`, are never committed, never uploaded, never attached to a PR. They're disposable and rebuildable, and after the analysis they were deleted. The CLI's help text says so at the top, so nobody has to infer it.

**Agents receive trimmed digests, not mail.** The classification prompt never sees an email. It sees a digest that is PII-free by construction - no addresses, no names, no headers, just direction, subject, and truncated cleaned bodies:

```ruby
# lib/support_mail/digester.rb
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

Long threads are compressed to the first four and last two messages. The truncation is a privacy control as much as a cost control: 700 characters of the opening message is enough to classify a support request and short enough to rarely reach the part where someone pastes a receipt.

**Only aggregates are retained.** Because the corpus was a one-time run, we needed the analysis to survive the data being deleted. So two artifacts were committed: a table of aggregate counts, and a row-level CSV with thread ids, bodies, and addresses stripped out - just theme, sentiment, resolution, year, and a PII-free one-line summary. Both were explicitly PII-scanned before commit. Future re-slicing needs no re-run and no re-ingestion.

**The findings language is anonymized.** Plans, PRs, and commit messages paraphrase ("a subscriber reported...") and never quote. Test fixtures use invented addresses.

The principle underneath all of it: **the sensitive artifact is disposable, and the durable artifact is de-identified.** Get that ordering right and the privacy story holds even if every other control fails.

## Results

The pipeline processed 33,377 messages with zero parse errors, resolving into 9,150 human threads that were classified end to end. Wall clock for the parse was under a minute; the full classification run was under half an hour.

What it produced:

- **A silent bug with a confirmed mechanism.** Approximately a hundred threads spanning 2017–2026, with a behavioral signature that named the responsible code path before the fix was designed. The fix is three surgical changes: refresh entitlement for existing members instead of only non-members, push the local lease forward whenever the billing service confirms an active subscription, and project a fresh lease from *now* rather than extrapolating from the original signup timestamp.
- **A post-release signal where no crash signal exists.** Because there's no crash event to watch, the success criterion is the corpus itself: those reports should trend toward zero.
- **Roadmap validation, mostly.** The corpus largely confirmed the existing plan rather than redirecting it - with one genuine gap that survived era-weighting, and several "top" historical asks that a redesign had already answered.
- **A reusable house voice.** Outbound replies were mined separately (year-stratified, weighted toward recent mail) into a `support-voice` skill - the canonical reference for customer-facing copy. Ten years of your own replies is a better style guide than anything you'd write from scratch.
- **A live counterpart.** The one-time corpus tool later grew a sibling that reads the live inbox and posts a weekly digest, so the same triage discipline applies going forward instead of only in retrospect.

## Lessons Learned

- **Crash reporters only see crashes.** Correctness failures, stale caches, and wrong-but-alive states produce no events. If your entire feedback loop is a crash dashboard, you are blind to an entire bug class - and it's the class users notice most.
- **Support email is telemetry you already collect.** It needs a parser and a schema, not a vendor. One SQLite file turns an unsearchable inbox into a queryable dataset.
- **The user's workaround names the code path.** This is the highest-leverage inference in the whole exercise. Ask what that action uniquely triggers, then look for what should have run earlier and didn't.
- **Prove the classification budget on a sample first.** A few hundred threads gives you a real tokens-per-item number *and* catches a wrong taxonomy before you've paid to apply it at scale.
- **Determinism makes the analysis defensible.** Seeded sampling, stable synthetic ids, and idempotent upserts mean any number can be reproduced instead of re-litigated.
- **Time-weight everything.** Lifetime totals in a long archive measure history, not demand. Report current-era counts and shares alongside them, or you'll prioritize solved problems.
- **Tag noise, don't delete it.** Labeling is reversible; deleting is not.
- **Decide the privacy boundary before you build.** Raw data disposable and local, agent inputs de-identified by construction, retained artifacts aggregate and scanned. Retrofitting that ordering is far harder than designing to it.

The broader point: agents are unusually good at reading a decade of unstructured human prose and turning it into a queryable table. That capability turns your support archive - which you already have, already consented to, and probably already ignore - into a second telemetry channel that covers precisely the failures your first one can't see.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and plan docs, then reviewed before publishing.
