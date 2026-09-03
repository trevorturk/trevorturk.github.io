---
layout: post
title: "Compute the Sky Yourself"
date: 2026-08-25 08:10:00 -0600
summary: "Why our weather app computes its own sun and moon events, how filling in only the fields the vendor left out keeps that cheap, and two failed optimizations that pointed us at a precomputed moon-phase grid."
tags: [ruby, performance, api-design, architecture]
---

## The Problem

A weather vendor's daily forecast almost always gives you sunrise and sunset. Ask for anything more and you get very little. Solar noon, civil twilight (the soft light before sunrise and after sunset), the moon's phase and how much of it is lit, moonrise and moonset: some vendors return one or two of these, and most return none.

[Hello Weather](https://helloweather.com) shows all of them. The moon panel needs a phase fraction and an illumination percentage. The sun panel shows civil twilight and solar noon as well as sunrise and sunset. So our server can't just forward what the vendor sent. It has to compute the missing fields itself.

That calculation runs for every day of every forecast, for every location, on a one-person product with a fixed server budget. Our web server is async, so all the requests in a worker share one Ruby CPU. A few hundred microseconds of astronomy per forecast day competes with everything else the request is doing (see [Falcon and Ruby Async](/falcon-async-performance/) for how that works). The calculation has to be right, it has to cover fields no vendor reliably sends, and it has to stay cheap. The first two optimizations we tried both failed, and the failures pointed at a narrower one.

## The Solution

We compute every sun and moon event ourselves with a real astronomy library, then merge the result with the vendor's data one field at a time. The vendor's value wins when it exists, and the computed one fills in when it doesn't. In production the library is the `astronomy-engine` npm package, and Ruby calls it through a Node process that stays running. In development and test, a pure-Ruby library stands in for it. We tried two obvious ways to make this cheaper: swapping in the lighter Ruby library, and putting a cache over it. Both made things worse. The measurements point to something narrower: precompute the one field that's the same for everyone on Earth, and interpolate it.

### Backfill only the field the vendor omitted

Vendors disagree about which astronomical fields they return. That's the same disagreement behind our [multi-source adapter](/multi-source-api-adapter-pattern/). So the merge can't be "vendor or engine" for the whole day. If we switched the whole day to the engine whenever one field was missing, we'd throw away accurate vendor values we already had. If we trusted the vendor alone, we'd ship blank moon panels. So the choice is made per field.

Each field is one fallback expression, `day.x || backfill&.x`. The vendor's parsed day is on the left and our engine is on the right. Here's the shape, runnable, with a stub standing in for the real engine:

```ruby
# A vendor's parsed daily payload. Most vendors fill sunrise/sunset and little
# else, so the remaining fields are frequently nil.
VendorDay = Struct.new(
  :sunrise, :sunset, :solar_noon, :sunrise_civil, :sunset_civil,
  :moon_phase, :moon_illumination, :moonrise, :moonset,
  keyword_init: true
)

# The in-house engine: computes any field from lat/lon/time. The real one calls
# an astronomy library; this stub returns fixed values so the block runs.
class Backfill
  def initialize(lat:, lon:, time:)
    @lat, @lon, @time = lat, lon, time
  end

  def moon_phase        = 0.42            # 0.0..1.0, global: no location needed
  def moon_illumination = 87
  def sunrise_civil     = @time - 1800
  def solar_noon        = @time + 21_600
  # ...one method per computed field
end

# Reconcile: prefer the vendor's value, fall back to the computed one.
class DailySunMoon
  def initialize(day:, backfill:)
    @day, @backfill = day, backfill
  end

  def sunrise           = @day.sunrise           || @backfill&.sunrise
  def solar_noon        = @day.solar_noon        || @backfill&.solar_noon
  def sunrise_civil     = @day.sunrise_civil     || @backfill&.sunrise_civil
  def moon_phase        = @day.moon_phase        || @backfill&.moon_phase
  def moon_illumination = @day.moon_illumination || @backfill&.moon_illumination
end

loc    = { lat: 41.87, lon: -87.62, time: Time.now }
vendor = VendorDay.new(sunrise: loc[:time], sunset: loc[:time] + 43_200)
day    = DailySunMoon.new(day: vendor, backfill: Backfill.new(**loc))

day.sunrise      # vendor value; the `||` short-circuits, engine never runs
day.moon_phase   # vendor nil, so this reads through to the backfill: 0.42
```

The engine only runs when it's needed. `@backfill` is only touched when the left side is nil, so a day where the vendor sent everything never calls it. The real code adds one twist. Some vendors report a named phase ("New", "First Quarter", "Full", "Last Quarter"). A name like that means a specific fraction, so when our computed phase disagrees with it, we clamp the computed number into that phase's range instead of overriding the name. We don't want a computed value to quietly contradict something the vendor stated outright.

### Compute the sky in JavaScript, call it from Ruby

Astronomy is a bad place to "roll your own". Rise and set times depend on how the atmosphere bends light and on the apparent size of the sun's disk. Twilight is a search for the moment the sun reaches a certain angle below the horizon. Get any of that slightly wrong and you find out months later from a customer's screenshot. The most accurate, best-tested library we found is a JavaScript package, but our server is Ruby. So the library runs in one Node process that stays up, and Ruby calls it through a thin bridge that makes a JS function look like a Ruby method. This is what production does, trimmed to sunrise and sunset:

```ruby
require "nodo"

# Bridge a JavaScript astronomy function into Ruby. Nodo boots one persistent
# Node process and exposes the JS function below as an ordinary Ruby method.
class Astro < Nodo::Core
  require Astronomy: "astronomy-engine"

  function :rise_set, <<~JS
    (lat, lon, startIso, days) => {
      const observer = new Astronomy.Observer(lat, lon, 0);
      const start    = new Astronomy.AstroTime(new Date(startIso));
      // SearchRiseSet applies refraction + disk radius (the civil definition
      // of sunrise: top of the disk crossing the horizon).
      const rise = Astronomy.SearchRiseSet('Sun', observer, +1, start, days);
      const set  = Astronomy.SearchRiseSet('Sun', observer, -1, start, days);
      return {
        sunrise: rise ? rise.date.toISOString() : null,  // null at polar day/night
        sunset:  set  ? set.date.toISOString()  : null,
      };
    }
  JS
end

astro = Astro.new
astro.rise_set(41.87, -87.62, Time.now.utc.strftime("%Y-%m-%dT00:00:00Z"), 1)
# => {"sunrise" => "...T10:52:...Z", "sunset" => "...T00:31:...Z"}
```

The `null` returns matter. Above the Arctic Circle in summer the sun doesn't set, so `SearchRiseSet` returns nothing, and every field that reads through this can be nil. The merge treats a nil rise or set as a real answer. Polar regions are places we support, not an edge case to crash on.

This bridge is also where the cost comes from. The Node process stays resident in memory, one per Ruby worker process. In May 2026 our web server ran two workers per dyno, so two JavaScript runtimes per dyno. Since then it runs one. That memory is the price of the library's accuracy, and it's what made dropping the runtime look so attractive.

### The cache that multiplied by worker count

The backfill does the same work over and over. One city recomputes the same sky thousands of times an hour. That's the textbook case for a cache, and the pure-Ruby astronomy library has one built in: a per-process least-recently-used cache, off by default. The two optimizations arrived together. In May 2026 we put the pure-Ruby library into production in place of the Node engine, to get the memory back. Ruby CPU went up. The same day, we turned on the library's cache with a cap of 1,000 entries to win that CPU back.

The cap came from a local benchmark that tried a range of cache sizes, each in its own process. The memory column is the part to keep. A per-process cache isn't shared between workers, so its cost multiplies by the number of workers you run. The 1,000-entry cache held under a megabyte of live objects. After garbage collection and compaction, resident memory (the memory the operating system has actually assigned to the process) told a different story:

```ruby
ENTRIES        = 1_000   # LRU cap enabled in the production canary
LIVE_MB        = 0.9     # live ObjectSpace freed by clearing the entries
AFTER_GC_MB    = 31.6    # resident-memory growth per worker vs. disabled, post-GC
FALCON_WORKERS = 2       # Ruby worker processes per dyno at the time

puts "live objectspace:       #{LIVE_MB} MB"      # what you reason about
puts "after-GC RSS / process: #{AFTER_GC_MB} MB"  # what you actually pay
puts "after-GC RSS / dyno:    #{AFTER_GC_MB * FALCON_WORKERS} MB"  # budgeted as ~64 MB
```

The 0.9 MB is the number you think about. The ~64 MB per dyno is the number you pay: resident memory after garbage collection, times every process that holds its own copy. We budgeted that against the two Node runtimes it replaced, and shipped.

The benchmark couldn't show us real traffic. With the cache on, Ruby time still spiked past our limits. Two days later we rolled both changes back: the Node engine went back into production and the pure-Ruby library went back to development and test. The library's cache stays off, and turning it on again means a fresh canary (a trial on one dyno, with CPU and memory watched). The two rollbacks show us where the edges are. The JS engine is cheap on CPU and expensive on memory. The Ruby engine is the reverse. A cache over the Ruby engine cost memory and didn't get enough CPU back. What was left to try was doing less of the calculation, instead of caching it.

### Precompute the one field that is the same everywhere

Sunrise depends on where you're standing. Moon phase doesn't. At any instant the moon shows the same lit fraction to everyone on Earth, so a moon-phase value doesn't need a location in its key. It also changes slowly and smoothly. A value like that can be sampled onto a grid once and read back by interpolation.

So the candidate for the moon side is a precomputed grid. Sample phase and illumination every 60 minutes of UTC across the forecast range, store the samples, and answer any instant by interpolating between the two nearest. Today that grid is a benchmark with its own rake task, not part of serving. The live merge still calls the engine on every request. We opened an implementation in May 2026 and closed it unmerged, because the CPU it would save wasn't expected to be enough to reduce the dyno count. So the idea is parked as a latency improvement, not a cost saving. If we bring it back, it has to beat the benchmark numbers below.

The lookup key is a UTC instant: the forecast day's local midday, converted to UTC. It's never a plain calendar date, because a calendar date is a different real moment in each time zone. This block builds a grid and interpolates it, and prints both timings. The `full_moon_phase` stand-in is pure Ruby so the block runs. Production computes each sample through the astronomy engine above, which is why the real gap is much wider than this one:

```ruby
require "benchmark"

SYNODIC  = 29.530588853 * 86_400          # mean lunar month, in seconds
NEW_MOON = Time.utc(2000, 1, 6, 18, 14)   # a reference new moon

# The "expensive" path. In production this crosses into the JS astronomy
# engine; here it is a pure-Ruby synodic approximation so the block runs.
def full_moon_phase(instant)
  ((instant - NEW_MOON) % SYNODIC) / SYNODIC        # 0.0..1.0
end

# Build the grid: one [epoch, phase] sample every `step` seconds.
def build_grid(from:, to:, step: 3600)
  (from.to_i..to.to_i).step(step).map { |e| [e, full_moon_phase(Time.at(e).utc)] }
end

# Answer any UTC instant by interpolating between the two nearest samples.
def phase_at(grid, instant)
  target = instant.to_i
  i = grid.bsearch_index { |epoch, _| epoch >= target } || grid.length - 1
  i = 1 if i.zero?
  (e0, p0), (e1, p1) = grid[i - 1], grid[i]
  ratio = (target - e0).to_f / (e1 - e0)
  p1 += 1.0 if p0 > 0.75 && p1 < 0.25               # unwrap across the new-moon seam
  result = p0 + (p1 - p0) * ratio
  result >= 1.0 ? result - 1.0 : result
end

grid  = build_grid(from: Time.utc(2026, 8, 1), to: Time.utc(2026, 9, 30))
probe = Time.utc(2026, 8, 25, 12, 0)

n = 100_000
Benchmark.bm(12) do |bench|
  bench.report("grid lookup") { n.times { phase_at(grid, probe) } }
  bench.report("full calc")   { n.times { full_moon_phase(probe) } }
end
```

The one non-obvious line is the seam unwrap. Phase is cyclic. It runs up to 1.0 and wraps to 0.0 at each new moon, so a naive average of 0.98 and 0.01 lands near 0.5 (a full moon) instead of near 0.0 (a new moon). Adding 1.0 before interpolating, then folding the result back, keeps it on the short way around the circle.

On the real engine the difference is large. A grid lookup measured 0.003 ms, against 0.284 ms for the full calculation, so roughly a hundred times faster. A 60-minute grid matches the engine's phase and illumination with a maximum illumination error around five millionths. The UI shows a whole-number percentage, so that error is invisible. The benchmark task regenerates those numbers on demand, so any later change to the step or the interpolation has to prove itself the same way.

The same precision argument settles the sun's cache key. Sun times shift about 2.1 seconds per kilometer east to west, so rounding a location key to one decimal place (about 11 km) moves a sunrise by at most half a minute. iOS shows times to the minute, so the rounding is invisible. Keeping five decimals of longitude in the key would wreck the hit rate for accuracy nobody can see.

## Results

- The forecast shows solar noon, civil twilight, moon phase and illumination, and moonrise and moonset, on top of the two fields most vendors return.
- We tried two optimizations in production in May 2026 and reverted both two days later, and wrote down why. The pure-Ruby engine raised CPU, and its per-process cache didn't win enough back. The benchmark before rollout had already priced that cache at ~0.9 MB of live objects but ~31.6 MB resident per worker, ~64 MB per dyno. Neither comes back without a fresh canary.
- The cost that stays is the resident Node process, one per Ruby worker: two per dyno then, one since the worker count dropped to one. That's the price of the engine's accuracy.
- The 60-minute UTC moon-phase grid benchmarked at roughly 0.003 ms per lookup against 0.284 ms for the full calculation, with illumination error far below what a percentage display can show. We closed its implementation unmerged in May 2026 because the CPU win was too small to justify a dyno trial. The benchmark task is still the gate if we revive it.

## Lessons Learned

- **Backfill per field, not per record.** Vendors disagree about which fields they return. A per-field `||` fallback keeps every vendor value and computes only the gaps. Swapping the whole record on any miss throws away data you already had.
- **Benchmark a cache on resident memory per dyno.** Live object size per process is the wrong number. The bill is resident memory after garbage collection, times every worker that holds its own copy.
- **Precompute what's global and smooth, and key what's local at the precision the display shows.** Moon phase is the same everywhere and close to linear over an hour, so a sampled grid beats both recomputing and caching. A display that rounds to the minute can't see a location key finer than ~11 km.
- **Write down why you reverted a change.** "The cache spiked CPU and multiplied RSS by worker count" stops the next person from turning it back on. A revert with no explanation invites a retry.

---

## How This Post Was Made

**Prompt 1:** "please kick off a big batch to look through all skills looking for other topics that might be interesting to blog about. we could look at git history, but I think since we've been using claude/codex for the last ~year we should have most of the interesting stuff built into the skills by now. however, you can also look at the changelog view in the iOS repo for other highlights that might be worth dispatching research about. come back to me with a list of possible topics (that haven't already been covered in the blog) …"

**Prompt 2:** "lets do 4, 20, 21, 22 -- the others I think are not worth it"

Ten Claude agents mined the iOS, web, and Android skills, the iOS changelog, and the plan indexes for uncovered topics; the owner picked four from the ranked list. This post was researched and drafted by one agent from the cited skills, plans, and code, under the why-then-how voice and self-contained-code brief settled in the previous localization batch, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The title lost its subtitle, the bolded rules in the body moved into Lessons Learned, Results now holds only what changed and what it cost, and the caveat that the moon grid is not yet in the serving path stays. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The cache section was resequenced to match the commits: the pure-Ruby engine replaced the Node one first (May 4, 2026), its cache was enabled the same day to recover CPU, and both were rolled back two days later; the 0.9 MB and 31.6 MB figures came from the pre-rollout benchmark and were budgeted, not discovered in production, so "both halves were wrong" and the unsupported "cache bookkeeping" explanation were removed, and the per-dyno figure now reads ~64 MB as the plan recorded it. The worker count is dated: two per dyno during the experiment, one since May 2026. The moon grid is no longer called the committed direction: its implementation PR was closed unmerged on May 11, 2026 as too small a CPU win, and the work is parked, so the caveat now says that.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 3, run after batch 2 (#69) merged. The prose now says "we" and uses contractions, "reconcile" and "reconciler" became "merge" throughout, resident memory and canary are defined where they first appear, and the two "for free" flourishes are gone. Judgment calls: the idiom "roll your own" stayed in quotes because it's the ordinary phrase; the closing line of the cache section ("The remaining direction was not to cache the calculation but to stop doing most of it") was reworded into a plain statement of what was left to try; and the Lessons bullet "Precompute what is global and smooth; key what is local at render precision" was reworded, since it read as a maxim, without changing its rule. Code blocks, numbers, quoted text, links, and headings are unchanged. Prompts, verbatim:

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
