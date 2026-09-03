---
layout: post
title: "Find the Hotspot, Then Cut the Dynos"
date: 2026-09-01 09:00:00 -0600
summary: "In January a profile said the API was fast, about 8ms a request. It had timed the wrong output. The output three quarters of traffic asks for cost 126ms. A one-file memoization cut that by 88%, and we walked production from 40 web dynos to 8 in an evening, checking logs against limits after each step, then to 6."
tags: [ruby, performance, heroku, falcon, benchmarking]
---

## The Problem

In January 2026 we profiled the [Hello Weather](https://helloweather.com) API and concluded it was already efficient. A request cost about 8ms of CPU against a mock data source. Most of that was JSON serialization, and it grew with the size of the response. The write-up said the remaining options all had trade-offs, and the cheapest was swapping the JSON library. Meanwhile production was running 40 Standard-2X web dynos on Heroku. It had been 4 before the current version of the app shipped in late 2025 with sun and moon calculations, a larger JSON payload, scores, and chart attributes. Six days later a cost analysis compared that fleet against moving to another cloud and found that code changes would pay back ten times faster. So the plan was to trim the web view path, where a 10ms request could lose 7.

The profile had timed the wrong output. The server builds several output shapes from one weather object, and the January numbers came from a shape most clients don't ask for. About three quarters of production requests want the format the current iOS app reads. That format carries the derived attributes (values computed from the raw forecast, like pressure trends and scores) that the app shows. Nobody had timed that path on its own. On February 12 a local run put it at about 126ms per request. Roughly 70ms of that was in the current-conditions derived attributes and another 19ms in scores. Astronomy, which the January plan had ranked as the top optimization, was 3.6ms.

The server runs on Falcon with fibers, so waiting on vendors overlaps and a dyno runs out of CPU before anything else (see [Falcon and Ruby Async](/falcon-async-performance/)). At 126ms per request on the dominant path, the one three quarters of traffic takes, that serialization was most of the fleet. We only found it by benchmarking that path.

## The Solution

Four steps, each with a measurement in front of it:

- Benchmark the dominant path locally, with no network involved
- Memoize the one accessor that redid its work on every read, so it remembers the answer
- Take the smaller wins the benchmark could still see, and stop when it couldn't
- Cut the dyno count one step at a time, checking logs against limits after each step

### Benchmark the path traffic actually takes

Production metrics tell you how long a request took, not which Ruby method took it. They also include time waiting on vendors, which no server change can move. So the probe that found the hotspot runs the full request path in a `rails runner` process against a mock source, and every millisecond is our Ruby. It stubs out the hit-tracking write, counts allocations alongside wall time, and starts the garbage collector on a schedule so its pauses land the same way in every run. Trimmed to the measuring loop:

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

A second probe answers the next question: which method? It wraps the same call in a `TracePoint` on `:call` and `:return`, filtered to the three files that hold derived attributes, scores, and chart values. Then it ranks methods by total time:

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

The call count column pointed at the fix. Methods like the pressure trend ran several times per object per request: once for the serializer and again for each sibling attribute built from them. No single method was slow. The same method was just running again for every reader.

These scripts lived in `/tmp` while we did the work. On February 24, after the results were in, we moved them into the repo with rake wrappers and a JSON file per run so two runs can be diffed. The limit is the file filter. It's a guess about where the hotspot lives, and a hotspot outside those three files wouldn't show up.

### Memoize the accessor everything reads through

The weather objects keep their attributes behind a lazy accessor. A value can be stored directly or as a proc, and a proc runs when the attribute is first read. That keeps building the object cheap when a caller only wants a few fields. The alternative, computing every attribute up front, would charge every request for fields it never reads. But the lazy accessor didn't remember the answer. Every read called the proc again. The serializer reads a trend attribute directly, then the attribute that names it, then the one that phrases it, and each of those reads the trend again.

The fix replaces the proc with its result on the first read. Here's the accessor after the change, in full:

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

The nested-callable check is the one line that isn't about speed. Before the change, a proc that returned a proc got called once, and the inner proc went back to the caller. After the change, the inner proc would be stored as the memoized value and called on the next read, with no warning. Raising makes the rule plain: a callable runs once, and its result is the value. Nil and false get memoized too, because the check asks whether the stored object responds to `call`, not whether it's truthy.

Before merging, we checked the change against the fiber model, because memoizing on an object under a concurrent server is how you get stale reads. Three facts made it safe. Every request builds its own weather object, so nothing is shared between requests. The vendor fetches run as sibling fibers under a barrier that waits for all of them before serialization starts, so by the time a lazy attribute is read, the request is back in plain sequential code. And the procs are pure computation over data already fetched. We grepped them for HTTP and fetch calls to confirm that, so a read never hands control to the scheduler partway through. If two fibers ever did read one object, the worst case is the same value computed twice.

On February 19 the change took the dominant path from 132ms to 16ms per request locally, an 88% cut. A stricter rerun at 360 iterations, comparing commits five days apart, put the before and after at 131.51ms and 16.34ms mean. One test broke. It built a weather object once and checked the icon under two frozen clock times, and the second check now saw the first answer. The fix was to build a fresh object for each time, which is what a real request does.

### Take the smaller wins, then stop

With the big cost gone, the same probe ranked what was left. Each candidate had to beat a baseline run interleaved with it, so drift on the machine didn't count as a win. Three shipped over the next five days. The web view adapter used to serialize to a hash and then wrap it. Replacing that with direct delegation cut the web view path 20%, from 11.29ms to 9.04ms, though web view paths were a small share of traffic by then. A process-wide cache for translation keys with no interpolation took another 8.9% off the dominant path, 18.77ms to 17.11ms, and saved 4,800 objects per request:

```ruby
I18N_KEY_CACHE = Concurrent::Map.new

def t(key, **options)
  locale = units.language.to_sym
  return I18n.t(key, locale: locale, **options) unless options.empty?

  locale_cache = I18N_KEY_CACHE.compute_if_absent(locale) { Concurrent::Map.new }
  locale_cache.compute_if_absent(key) { I18n.t(key, locale: locale).freeze }
end
```

Keys with interpolation go straight to the translation library, because a correct cache key would have to include the options, and a follow-up probe found that caching those too made the path 3% slower. The third change moved the timezone and the "now" cutoff out of the hourly and daily loops, so they're computed once per block.

We stopped because of a report on February 24, not because we ran out of ideas. Memoizing the trend methods one by one measured at plus 0.19% against the baseline, inside noise. Aggregating the score traversals measured at plus 1.76%. Encoding a prebuilt hash took 0.06ms of a 17ms request, so swapping the JSON library couldn't matter. The largest lever left was translation, and stubbing it out entirely put its ceiling at about 3%. The report closed the track and wrote down when to reopen it, tied to pressure in production. One more memoization did land in April. It measured 3.6%, and we justified it by the trace showing the repeated calls, not by the request mean.

### Walk the dynos down under guardrails

The memoization deployed on February 19. On February 23 the Heroku dashboard still showed 40 web dynos at a one-minute load average of 0.04. The count was the cost, and the question was how far it could drop before latency or errors moved. We rejected one big cut on a hunch, because the traffic has a shape. A silent push wakes thousands of devices at the top and bottom of every hour, so a fleet that looks fine at 9:34 can fall over at the next spike (see [Heroku Capacity](/heroku-capacity/) for the tooling that grew out of this).

Each step ran the same loop. Scale down, wait three minutes for the new fleet to settle, pull the router and runtime logs several times twenty seconds apart, drop duplicate request ids, and summarize. The summary had p50, p95, and p99 service time by path, status counts, one-minute load, memory, and swap. A step held only if every limit held, and we wrote the limits down before the first cut:

- Router p95 under 700ms, p99 under 1200ms, and the app API p95 under 750ms
- 5xx rate under 0.2% of sampled requests
- Load p95 under 0.70 in a normal window and under 1.00 in a spike window, never above 1.00 for five minutes
- Memory p95 under 860MB of the 1GB quota, swap p95 under 50MB
- No sustained router timeout or connection errors

The sequence was 40 to 28 to 20 to 18 to 14 to 10 to 8, all on the evening of February 23 into the early hours of February 24 UTC. We checked the last step against a top-of-hour spike. Two moments broke the rhythm. At 20 dynos a short sample showed two 504s, a 0.27% error rate against the 0.2% line. Both were widget requests to one vendor that took two seconds, so a vendor timeout and not a saturated dyno, and a second, larger sample came back clean. At 10 dynos the first sample had a 502 and a 504. The router codes said one was a client disconnect and the other a vendor failure, and again we took a larger sample before the next cut. The rule was that a missed limit with a known outside cause earns a bigger sample, not a pass.

Two days later another capture showed steady headroom, and the fleet went to 6, where it has stayed. In May we tried 5 and rolled it back. A full spike capture passed the hard limits but left too little margin on p95 latency.

## Results

- Local CPU on the dominant path: 131.51ms to 16.34ms mean per request, measured at 360 iterations against a mock source, comparing commits five days apart. One file accounts for nearly all of it.
- Web dynos: 40 to 8 in one evening on February 23 and 24, 2026, then 6 on February 26. That's 85% fewer than in January, and the monthly dyno bill fell by the same share.
- Two measured follow-ups took 20% and 8.9% off their paths. A third wasn't benchmarked on its own. We closed the track on February 24 with the remaining candidates inside noise.
- What it cost: the accessor now raises on a nested callable, and tests that reused one object across two times have to build a fresh one.

## Lessons Learned

- Profile the output your traffic actually asks for. A profile of the wrong path can call the system efficient and rank the wrong fix first.
- Count calls, not just time. A method that's cheap once but runs on every read shows up in the call column before it shows up in the mean.
- Before memoizing on an object under a fiber or thread server, check three things: the object belongs to one request, a read can't yield to the scheduler partway through, and the memoized function is pure.
- Stop when candidates measure inside noise against an interleaved baseline, and write down what would reopen the work.
- Cut capacity in steps, with limits written before the first cut. Treat a miss with a known outside cause as a reason to take a bigger sample, not as a pass.

---

## How This Post Was Made

**Prompt 1:** "did we have a post about our major CPU savings work we did a while back? IIRC we cut our Heroku bill by a huge amount w/ Claude doing local benchmarks and finding hotspots"

**Prompt 2:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

Generated by Claude Fable 5.1 using the blog-post-generator skill. The first prompt found that the CPU pass was mentioned in two existing posts but had none of its own. Sources: the January 2026 CPU plans and profile results, the February 12 plan that re-measured the app output path, the February 19 memoization PR and its concurrency audit, the direct-delegation and translation-cache PRs of February 23, the February 24 report that closed the track, the dyno step-down plan with its per-window log samples, the February 25 closeout summary, `scripts/benchmark/weather_request_probe.rb`, `scripts/benchmark/derived_hotspots_probe.rb`, `app/models/api/concerns/lazy_attr_accessor.rb`, and `config/heroku/formation.yml` history. Judgment calls: dollar figures recorded in the plans are expressed here as dyno counts and percentages, the product-specific output name is described rather than named, the vendor behind the 504s is not named, and the "three quarters of traffic" figure is the plan's approximate number as of February 2026.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 3, run after batch 2 (#69) merged. Rewrote the prose from an ELI5 of the post: first person and contractions throughout, "derived attributes", "memoize", and "interleaved baseline" defined at first use, one name each for the dominant path, the vendors, and the fleet, and the sentences that packed three facts split into one fact each. Judgment calls: "the contract" and "the whole diff" on the accessor became "the rule" and "the accessor after the change, in full", and "formation" became "fleet" so the dyno count has one name; code, headings, numbers, and the guardrail bullets are unchanged. Prompts, verbatim:

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
