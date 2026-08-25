---
layout: post
title: "One Tag That Tenfolded the Metrics Bill: Cardinality Guardrails and Observability as a Postgres Table"
date: 2026-08-25 08:00:00 -0600
summary: "How a single version tag multiplied a weather API's custom-metric spend roughly tenfold, the allowlist that stopped it, and the append-only Postgres table that replaces the metrics vendor entirely."
tags: [ruby, observability, architecture, workflow]
---

## The Problem

[Hello Weather](https://helloweather.com) is a one-person weather API. It answers a few hundred thousand requests a day and, like any service that fans out to several upstream providers, it needs to see itself: how many requests came in, from which client context, resolved by which source, and with what status. That visibility was a handful of custom metrics shipped to a monitoring vendor. Two of them carried most of the volume: an `internal` counter tagged with context, paid status, source, and result status, and a legacy `request` counter tagged with platform.

Custom metrics on that kind of vendor are not priced per data point. They are priced per *series* — per unique combination of tag values. A counter tagged with five contexts and six statuses is thirty series, and thirty series is nothing. The trap is that series count is the product of the tag cardinalities, not the sum, so a new tag does not add to the bill, it multiplies it.

One tag did exactly that. To compare how a new app build behaved against the previous one, a `version` tag was added to both counters. It seemed free: version is just one more label. But every shipped build minted a fresh value, and that value multiplied across every other dimension already on the counter. Custom-metric spend rose roughly tenfold over three months — from a few hundred series to a few thousand — before anyone connected the climbing invoice to the innocent-looking tag. Nothing broke, no alert fired, no query got slower. The only symptom was a bill that had quietly changed shape from a flat line into a curve.

## The Solution

The immediate fix was to drop the `version` tag from both high-volume counters, which collapsed the potential series count by more than an order of magnitude. The durable fix is a rule and a destination. The rule: normalize every dimension against an allowlist and bucket anything unrecognized as `"unknown"` *before* it is emitted, so no future tag can leak unbounded values into the series count. The destination: stop paying a vendor per series at all, and move the counters into an append-only Postgres table the service already runs, with hourly rollups and an in-app query page. Separately, the one genuinely expensive observability feature — distributed tracing — stays pinned off by default and is switched on only inside a named, time-boxed window.

### One tag turned a flat cost into a curve

The reason this is worth a blog post is that the failure is invisible in code review. A tag is a string; adding one looks like a one-line change with no blast radius. The blast radius only exists in the arithmetic, so the first guardrail is to make the arithmetic explicit. Here is the whole cost model as a runnable Ruby snippet — it multiplies the dimension sizes and prints the series count with and without the extra tag:

```ruby
DIMENSIONS = {
  context: 5,   # app, widget, complication, watch, ping
  paid:    2,   # true / false
  source:  12,  # upstream providers
  status:  6,   # ok plus five error classes
}

def series(dimensions)
  dimensions.values.reduce(1, :*)
end

before = series(DIMENSIONS.merge(version: 11)) # one value per shipped build
after  = series(DIMENSIONS)

puts "with version tag:    #{before}"    # 7920
puts "without version tag: #{after}"     # 720
puts "ratio: #{before / after}x"         # 11
```

Notice what the `version: 11` line does: eleven shipped builds do not add eleven series, they multiply the existing 720 into 7,920. And 11 is only today's count. Every future release increments it, so the series count grows for as long as the app ships, which is why a version tag on a high-volume counter is not a large fixed cost but an unbounded one. The legacy `request` counter had the same shape — its own set of dimensions times a growing version count — and dropping the tag there cut its potential series by a similar factor. Run this snippet before adding any tag to a metric that is already the product of several dimensions; if the new tag's cardinality is unbounded, so is the metric.

### Normalize before you emit, not at query time

The version tag was the acute failure, but it was a symptom of a missing gate: nothing sat between a raw request value and the tag that got emitted. Any dimension sourced from client input — a `source` string, a `context`, a request path — can arrive misspelled, renamed by an old client, or outright garbage, and each novel value is a new series whether or not anyone ever queries it. Cleaning up at dashboard-query time does not help, because the cost was already paid at ingest. The fix has to run before the metric is written.

So every dimension goes through an allowlist, and anything not on its list becomes `"unknown"`. Booleans that were once free-text tags (a timeout flag, a fallback flag) collapse to `"true"`/`"false"`. Version is simply not allowed as a high-volume dimension anymore. The helper is small, and its test is the actual specification — the point of the whole exercise is the second case, where an unexpected value is forced into a single bucket instead of minting a series:

```ruby
module MetricNormalizer
  UNKNOWN = "unknown"

  ALLOWLISTS = {
    context: %w[app widget complication watch ping],
    status:  %w[ok bad_request unauthorized too_many_requests bad_gateway gateway_timeout],
    source:  %w[a b c d e f g h i j k l], # the finite set of supported providers
  }.freeze

  def self.normalize(dimension, value)
    allowed = ALLOWLISTS.fetch(dimension)
    allowed.include?(value) ? value : UNKNOWN
  end
end
```

```ruby
require "minitest/autorun"

class MetricNormalizerTest < Minitest::Test
  def test_allowed_values_pass_through
    assert_equal "widget", MetricNormalizer.normalize(:context, "widget")
  end

  def test_unrecognized_values_are_bucketed
    assert_equal "unknown", MetricNormalizer.normalize(:context, "5.3.1")
  end

  def test_nil_is_bucketed
    assert_equal "unknown", MetricNormalizer.normalize(:context, nil)
  end
end
```

The `"5.3.1"` in the second test is the whole story in miniature: if a build number ever reaches a dimension again — through a renamed client, a new tag, a copy-paste — it lands in `"unknown"` instead of becoming series number 7,921. The allowlist also fails loud in the good way: `fetch` raises if you try to normalize a dimension nobody has declared, so adding a new tag forces you to state its finite set of values up front. That is the moment the cost model gets a chance to stop you.

### Move the metrics into a table you already run

Even with cardinality under control, there is a structural mismatch: a vendor that bills per series charges you for *shape* — for having dimensions at all — while the same counters in Postgres cost only *rows*, and rows are cheap and already on the invoice. The service runs a database. Counting is what a database is for. So the design retires the vendor's custom metrics and writes each event to an append-only table, generic enough to hold every counter:

```ruby
class CreateEvents < ActiveRecord::Migration[8.0]
  def change
    create_table :events do |t|
      t.string   :metric,     null: false
      t.datetime :created_at, null: false
      t.string   :source
      t.string   :status
      t.string   :context
      t.boolean  :paid
      t.integer  :value, default: 1, null: false
    end

    add_index :events, [:metric, :created_at]
  end
end
```

Writes are fire-and-forget on the request's async reactor and rescue their own exceptions to Sentry, so a metric write can never fail a weather response — the same pattern the request-tracking path already uses. Raw rows are useful for a couple of hours of live debugging and then pure weight, so a job that runs every few minutes folds everything older than two hours into hourly buckets and deletes the raw rows. The rollup is one upsert:

```sql
INSERT INTO event_rollups (metric, hour, source, status, context, paid, count)
SELECT metric,
       date_trunc('hour', created_at) AS hour,
       source, status, context, paid,
       SUM(value)
FROM events
WHERE created_at < NOW() - INTERVAL '2 hours'
GROUP BY metric, hour, source, status, context, paid
ON CONFLICT (metric, hour, source, status, context, paid)
DO UPDATE SET count = event_rollups.count + EXCLUDED.count;
```

Notice the retention split: raw events live two hours, rollups live thirty days. At a few hundred thousand events a day the raw table never holds more than a few tens of thousands of rows, and a month of hourly rollups is a couple of million — call it a few hundred megabytes, negligible next to the weather data the same database already stores. The `1h` dashboard queries read raw events for real-time detail; the `24h` dashboard sums the rollups. Both are plain `GROUP BY` queries behind an admin-only page, which means the query language is SQL you already know instead of a vendor's proprietary metric syntax, and there is no per-series meter running underneath.

The one guardrail this design must respect is that Postgres is not free either: trading a metrics bill for a database that falls over is not a win. So the plan gates itself behind the existing database-load baseline and keeps the normalization rule from the previous section — cardinality has to stay bounded whether the series lands in a vendor or in your own table, because an unbounded `GROUP BY` key bloats the rollup index just as surely as it bloated the invoice.

### Keep the expensive layer off by default

Custom metrics were the visible cost, but the more dangerous one is distributed tracing and span events, because tracing emits a record per span and dominates ingest the moment it is on. The rule that came out of this is blunt: tracing stays pinned off in config as a hard default, not a maybe, and is re-enabled only inside a named, time-boxed window with consumption checked before and after. Treating "on" as the exceptional state, with an explicit exit, is what keeps a debugging session from silently becoming a standing cost. The window is a four-step checklist a person actually follows:

```bash
# 1. BEFORE: record the tracing ingest baseline, so "after" has a number to beat.
#    NRQL: SELECT sum(BytesIngested) FROM NrConsumption
#          WHERE usageMetric = 'TracingBytes' SINCE 1 hour ago

# 2. ENABLE for the agreed window only — announce start and stop times first.
export TRACING_ENABLED=true
export SPAN_EVENTS_ENABLED=true
#    deploy, reproduce the issue, capture the trace...

# 3. DISABLE — flip both back to the pinned default the instant the window closes.
unset TRACING_ENABLED SPAN_EVENTS_ENABLED
#    deploy again.

# 4. AFTER: confirm tracing ingest fell back to ~0. If it did not, tracing is
#    still on somewhere — find it before you close the window.
```

The reason step 4 exists is that a window you open and forget is indistinguishable from never turning it off, and the whole point is that tracing is the layer you cannot afford to leave running. The consumption query before and after turns "I think I disabled it" into a number.

This sits inside a wider division of labor: two monitoring tools each own exactly one layer and never cross. One tool owns Postgres internals — connections, load, I/O, blocking queries, index hit rate. The other owns request- and job-level database call counts and durations. You do not ask the request-tracing tool about connection saturation, and you do not ask the database tool about which endpoint is slow. Keeping the boundary sharp is what lets you keep the ingest-heavy features of each tool off without losing coverage, because nothing is trying to be your one pane of glass for everything.

## Results

- The shipped mitigation dropped the `version` tag from both high-volume counters, cutting the `internal` counter's potential series from 7,920 to 720 and the legacy `request` counter's by a similar factor. Custom-metric spend, which had climbed roughly tenfold across three months, was designed to fall back toward its original baseline on the following billing cycle.
- The invisible-failure class is now caught in review: cardinality is a runnable arithmetic model anyone can paste and run before adding a tag, and an allowlist with an `"unknown"` bucket makes it structurally impossible for an unbounded value like a build number to mint new series again.
- The replacement design moves every counter into an append-only Postgres table with hourly rollups and an admin query page, retiring the per-series metrics vendor. Storage is a few hundred megabytes for a month of history, and the dashboards become ordinary SQL instead of proprietary metric queries.
- Tracing and span events stay off by default with a four-step enable-and-disable runbook, and the two monitoring tools hold a clean split — one for Postgres internals, one for request and job database duration — so the expensive layers of each stay dark until a named window opens them.

## Lessons Learned

- **A new metric tag multiplies the bill, it does not add to it.** Series count is the product of tag cardinalities, so before adding any dimension to a busy counter, compute the new product — an unbounded tag makes an unbounded cost.
- **Never tag a metric with a value that grows without end.** Build numbers, user IDs, and request paths all look like ordinary labels and all mint a fresh series per value, which is why version tagging turned a flat cost into a curve nobody noticed for three months.
- **Normalize dimensions before you emit, not at query time.** The series is created and paid for at ingest, so an allowlist that buckets unrecognized values as `"unknown"` is the only place the guardrail actually holds.
- **Count in the database you already run.** Per-series pricing charges you for having dimensions at all, while rows in Postgres cost only rows, so an append-only table with rollups replaces the vendor for the cost of some cheap storage.
- **Make the expensive feature off the default and on the exception.** Distributed tracing dominates ingest, so it lives pinned off with a time-boxed window and a before/after consumption check, because a window you forget to close is a standing bill.

---

## How This Post Was Made

**Prompt 1:** "please kick off a big batch to look through all skills looking for other topics that might be interesting to blog about. we could look at git history, but I think since we've been using claude/codex for the last ~year we should have most of the interesting stuff built into the skills by now. however, you can also look at the changelog view in the iOS repo for other highlights that might be worth dispatching research about. come back to me with a list of possible topics (that haven't already been covered in the blog) …"

**Prompt 2:** "lets do 4, 20, 21, 22 -- the others I think are not worth it"

Ten Claude agents mined the iOS, web, and Android skills, the iOS changelog, and the plan indexes for uncovered topics; the owner picked four from the ranked list. This post was researched and drafted by one agent from the cited skills, plans, and code, under the why-then-how voice and self-contained-code brief settled in the previous localization batch, then reviewed before publishing.
