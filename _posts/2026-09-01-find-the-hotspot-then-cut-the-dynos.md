---
layout: post
title: "Find the Hotspot, Then Cut the Dynos"
date: 2026-09-01 09:00:00 -0600
summary: "A January profile said the API was efficient at 8ms a request. It had measured the wrong output. Profiling the one three quarters of traffic used found 126ms, a one-file memoization cut that by 88%, and production walked from 40 web dynos to 8 in an evening under log-sampled guardrails, then to 6."
tags: [ruby, performance, heroku, falcon, benchmarking]
---

## The Problem

In January 2026 a profiling pass on the [Hello Weather](https://helloweather.com) API concluded that it was already efficient. A request cost about 8ms of CPU with a mock data source, most of it JSON serialization proportional to the size of the response, and the write-up said so: the remaining options all had trade-offs, and the cheapest was swapping the JSON library. Production was running 40 Standard-2X web dynos on Heroku, up from 4 before the current version of the app shipped in late 2025 with sun and moon calculations, a larger JSON payload, scores, and chart attributes. A cost analysis six days later compared that fleet against a cloud migration and concluded that code changes would pay back ten times faster than moving providers. The plan was to trim the web view path, where a 10ms request could lose 7.

The profile had measured the wrong output. The server renders several output shapes from one weather object, and the January numbers came from a different shape than the one most clients ask for. About three quarters of production requests asked for the format the current iOS app consumes, which carries the derived attributes the app shows: pressure trends with names and phrases, temperature trends, level labels, scores. Nobody had timed that path on its own. On February 12 a local run put it at about 126ms per request, with roughly 70ms inside the current-conditions derived attributes and another 19ms in scores. Astronomy, which the January plan had ranked as the top optimization, was 3.6ms of that.

The server is Falcon with fiber-based concurrency, so the waiting on upstream vendors overlaps and CPU is what a dyno runs out of (see [Falcon and Ruby Async](/falcon-async-performance/)). A 126ms serialization per request, on the dominant path, is most of the fleet. Finding it took a benchmark that ran the path traffic actually took.

## The Solution

Four steps, each gated by a measurement:

- Benchmark the dominant path locally, with the network out of the loop
- Memoize the one accessor that was re-running work on every read
- Take the smaller wins the benchmark could still see, and stop when it could not
- Walk the dyno count down under guardrails, one step per sampled window

### Benchmark the path traffic actually takes

Production metrics say how long a request took, not which Ruby method took it, and they include upstream latency that no server change can move. The probe that found the hotspot runs the full request path in a `rails runner` process against a mock source, so every millisecond is Ruby. The hit-tracking write is stubbed out, allocations are counted alongside wall time, and the garbage collector is started on a schedule so its pauses land in every run the same way. Trimmed to the measuring loop:

```ruby
# scripts/benchmark/weather_request_probe.rb (trimmed)
iters  = ENV.fetch("ITERS", "220").to_i
warmup = ENV.fetch("WARMUP", "45").to_i
params = { source: "mock", output: ENV.fetch("OUTPUT", "app"), lat: 41.87, lon: -87.62 }

samples_ms = []
allocs = []

(warmup + iters).times do |i|
  GC.start if (i % 25).zero?
  before_alloc = GC.stat(:total_allocated_objects)
  t0 = Process.clock_gettime(Process::CLOCK_MONOTONIC)

  Weather.new(params).to_json

  elapsed_ms = (Process.clock_gettime(Process::CLOCK_MONOTONIC) - t0) * 1000.0
  next if i < warmup
  samples_ms << elapsed_ms
  allocs << (GC.stat(:total_allocated_objects) - before_alloc)
end

sorted = samples_ms.sort
puts JSON.pretty_generate(
  mean_ms: (samples_ms.sum / samples_ms.length).round(3),
  p95_ms: sorted[(sorted.length * 0.95).floor].round(3),
  allocs_per_req: (allocs.sum.to_f / allocs.length).round(1),
)
```

A second probe answers the next question, which method. It wraps the same call in a `TracePoint` on `:call` and `:return`, filtered to the three files that hold derived attributes, scores, and chart values, and ranks methods by cumulative time:

```ruby
# scripts/benchmark/derived_hotspots_probe.rb (trimmed)
target_suffixes = %w[derived_attributes.rb derived_scores.rb derived_chart_attributes.rb]
stats = Hash.new { |h, k| h[k] = { calls: 0, total_s: 0.0 } }
stack = []

trace = TracePoint.new(:call, :return) do |tp|
  next unless target_suffixes.any? { |suffix| tp.path.end_with?(suffix) }

  if tp.event == :call
    stack << ["#{File.basename(tp.path)}##{tp.method_id}", Process.clock_gettime(Process::CLOCK_MONOTONIC)]
  elsif (top = stack.pop)
    key, t0 = top
    stats[key][:calls] += 1
    stats[key][:total_s] += Process.clock_gettime(Process::CLOCK_MONOTONIC) - t0
  end
end

warmup.times { work.call }
trace.enable
n.times { work.call }
trace.disable

ranked = stats.map { |key, v| { method: key, calls: v[:calls], total_ms: (v[:total_s] * 1000).round(3) } }
puts JSON.pretty_generate(ranked.sort_by { |row| -row[:total_ms] }.first(40))
```

The call count column is what pointed at the fix. Methods like the pressure trend were being called several times per object per request, once by the serializer and again by each sibling attribute that derived from them. The cost was not any one method being slow. It was the same method being run again for every reader.

These scripts lived in `/tmp` while the work was being done and were promoted into the repo on February 24, after the results were in, with rake wrappers and a JSON artifact per run so two runs can be diffed. Where the hotspot lives cannot be enforced by a script; the file filter is a guess, and a hotspot outside those three files would not appear.

### Memoize the accessor everything reads through

The weather objects hold their attributes behind a lazy accessor. A value can be stored directly or as a proc, and a proc is evaluated when the attribute is first read, so building the object stays cheap when a caller only wants a few fields. The alternative, computing every attribute at construction, would charge every request for fields it never reads. What the lazy accessor did not do was remember the answer. Every read called the proc again, and the serializer reads a trend attribute directly, then reads the attribute that names it, then the one that phrases it, each of which reads the trend again.

The fix replaces the proc with its result on first read. It is the whole diff to the accessor:

```ruby
module LazyAttrAccessor
  extend ActiveSupport::Concern

  class_methods do
    def lazy_attr_accessor(names)
      names.each do |name|
        attr_writer(name)

        define_method(name) do
          val = instance_variable_get("@#{name}")
          if val.respond_to?(:call)
            result = val.call
            if result.respond_to?(:call)
              raise ArgumentError, "lazy_attr_accessor does not support nested callables for :#{name}"
            end
            instance_variable_set("@#{name}", result)
            result
          else
            val
          end
        end
      end
    end
  end
end
```

The nested-callable check is the one line that is not about speed. Before the change, a proc that returned a proc would have been called once and the inner proc handed back to the caller. After it, the inner proc would be stored as the memoized value and called on the next read, silently. Raising makes the contract explicit: a callable is evaluated once, and its result is the value. Nil and false memoize too, since the check is on the stored object responding to `call`, not on truthiness.

The change was audited for the fiber model before merging, because caching on an object under a concurrent server is where stale reads come from. Three facts made it safe. Every request builds its own weather object, so there is no cross-request sharing to worry about. The upstream fetches run as sibling fibers under a barrier that waits for all of them before serialization starts, so by the time a lazy attribute is read, the request is in single-fiber sequential code. And the procs are pure computation over already-fetched data, checked by grepping them for HTTP and fetch calls, so a read never yields to the scheduler mid-evaluation. The worst case, if two fibers ever did read one object, is the same value computed twice.

Locally the change took the dominant path from 132ms to 16ms per request, an 88% cut, on February 19. A stricter rerun at 360 iterations, against commits five days apart, put the before and after at 131.51ms and 16.34ms mean. One test broke: it built a weather object once and asserted the icon under two frozen clock times, and the second assertion now saw the first answer. The fix was to build a fresh object per time, which is what a request does.

### Take the smaller wins, then stop

With the big term gone, the same probe ranked what was left, and each candidate had to beat an interleaved baseline. Three shipped over the next five days. Replacing the serialize-to-hash-then-wrap path in the web view adapter with direct delegation cut that path 20%, from 11.29ms to 9.04ms, though the web view paths were a minority of traffic by then. A process-wide cache for translation keys with no interpolation took another 8.9% off the dominant path, 18.77ms to 17.11ms, and 4,800 fewer objects per request:

```ruby
I18N_KEY_CACHE = Concurrent::Map.new

def t(key, **options)
  locale = units.language.to_sym
  return I18n.t(key, locale: locale, **options) unless options.empty?

  locale_cache = I18N_KEY_CACHE.compute_if_absent(locale) { Concurrent::Map.new }
  locale_cache.compute_if_absent(key) { I18n.t(key, locale: locale).freeze }
end
```

Interpolated keys go straight to the translation library, since the cache key would have to include the options to be correct, and a follow-up probe found that caching those too made the path 3% slower. The third change hoisted the timezone and the "now" cutoff out of the hourly and daily loops so they were computed once per block.

The stop came from a report on February 24, not from running out of ideas. Memoizing the trend methods individually measured at plus 0.19% against the baseline, inside noise. Aggregating the score traversals measured at plus 1.76%. Encoding a prebuilt hash was 0.06ms of a 17ms request, so swapping the JSON library could not matter. The largest remaining lever, translation, had an upper bound of about 3% measured by stubbing it out entirely. The report closed the track with re-open criteria tied to production pressure, and the later memoization that did land, in April, measured 3.6% and was justified by the trace showing the calls, not by the request mean.

### Walk the dynos down under guardrails

The memoization deployed on February 19. On February 23 the Heroku dashboard still showed 40 web dynos at a one-minute load average of 0.04. The cost lever was the count, and the question was how far it could drop before latency or errors moved. The alternative, one large cut on a hunch, was rejected because the traffic has a shape: a silent-push refresh wakes thousands of devices at the top and bottom of every hour, so a formation that looks fine at 9:34 can fail at the next spike (see [Heroku Capacity](/heroku-capacity/) for the tooling that grew out of this).

Each step was the same loop. Scale down, wait three minutes for the formation to converge, pull the router and runtime logs several times twenty seconds apart, deduplicate by request id, and summarize: p50, p95, p99 service time by path, status counts, one-minute load, memory and swap. A step held only if every guardrail held. The thresholds were written before the first cut:

- Router p95 under 700ms, p99 under 1200ms, and the app API p95 under 750ms
- 5xx rate under 0.2% of sampled requests
- Load p95 under 0.70 in a normal window and under 1.00 in a spike window, never above 1.00 for five minutes
- Memory p95 under 860MB of the 1GB quota, swap p95 under 50MB
- No sustained router timeout or connection errors

The sequence was 40 to 28 to 20 to 18 to 14 to 10 to 8, all on the evening of February 23 into the early hours of February 24 UTC, with the last step validated against a top-of-hour spike. Two moments broke the rhythm. At 20 dynos a short sample showed two 504s, a 0.27% error rate against the 0.2% line. Both were widget requests to one upstream source at two seconds of service time, a vendor timeout rather than dyno saturation, and a second larger sample came back clean. At 10 dynos the first sample carried a 502 and a 504, classified from the router codes as a client disconnect and an upstream failure, and again a larger repeat sample was required before the next cut. The rule was that a guardrail miss with a known non-saturation cause buys a bigger sample, not a pass.

Two days later a further capture showed sustained headroom and the formation went to 6, where it has stayed. A probe to 5, tried in May, was rolled back when a full spike capture passed the hard thresholds but left too thin a margin on p95 latency.

## Results

- Local CPU on the dominant path: 131.51ms to 16.34ms mean per request, measured at 360 iterations against a mock source, at commits five days apart. One file changed for nearly all of it.
- Web dynos: 40 to 8 across one evening on February 23 and 24, 2026, then 6 on February 26. The count is 85% lower than in January, and the monthly dyno bill fell in proportion.
- Two measured follow-ups took 20% and 8.9% off their paths, a third was not benchmarked on its own, and the track was closed on February 24 with the remaining candidates inside noise.
- The cost that stayed: a nested-callable contract on the accessor, and tests that reuse one object across time now have to build a fresh one.

## Lessons Learned

- Profile the output your traffic mix actually requests. A profile of the wrong path can conclude the system is efficient and rank the wrong optimization first.
- Count calls, not just time. A method that is cheap once and called on every read shows up in the call column before it shows up in the mean.
- Before memoizing on an object under a fiber or thread server, check three things: the object is request-local, no scheduler yield can happen mid-read, and the cached function is pure.
- Stop when candidates measure inside noise against an interleaved baseline, and write the stop down with re-open criteria.
- Cut capacity in steps sized to the evidence, with thresholds written before the first cut, and treat a miss with a known external cause as a reason for a larger sample rather than a pass.

---

## How This Post Was Made

**Prompt 1:** "did we have a post about our major CPU savings work we did a while back? IIRC we cut our Heroku bill by a huge amount w/ Claude doing local benchmarks and finding hotspots"

**Prompt 2:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

Generated by Claude Fable 5.1 using the blog-post-generator skill. The first prompt found that the CPU pass was mentioned in two existing posts but had none of its own. Sources: the January 2026 CPU plans and profile results, the February 12 plan that re-measured the app output path, the February 19 memoization PR and its concurrency audit, the direct-delegation and translation-cache PRs of February 23, the February 24 report that closed the track, the dyno step-down plan with its per-window log samples, the February 25 closeout summary, `scripts/benchmark/weather_request_probe.rb`, `scripts/benchmark/derived_hotspots_probe.rb`, `app/models/api/concerns/lazy_attr_accessor.rb`, and `config/heroku/formation.yml` history. Judgment calls: dollar figures recorded in the plans are expressed here as dyno counts and percentages, the product-specific output name is described rather than named, the vendor behind the 504s is not named, and the "three quarters of traffic" figure is the plan's approximate number as of February 2026.
