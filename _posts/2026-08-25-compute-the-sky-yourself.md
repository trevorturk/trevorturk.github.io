---
layout: post
title: "Compute the Sky Yourself"
date: 2026-08-25 08:10:00 -0600
summary: "Why a weather app computes its own sun and moon events, how backfilling only the missing fields keeps it cheap, and two performance reversals that pointed to a precomputed moon-phase grid."
tags: [ruby, performance, api-design, architecture]
---

## The Problem

A weather vendor's daily forecast almost always gives you two astronomical facts: sunrise and sunset. Ask for anything past that and the payload thins out fast. Solar noon, civil twilight (the softer light before sunrise and after sunset), the moon's phase and how illuminated it is, moonrise and moonset: some vendors return one or two, most return none.

[Hello Weather](https://helloweather.com) shows all of them. The moon panel needs a phase fraction and an illumination percentage. The sun panel wants civil twilight and solar noon, not just the two endpoints of the day. So the server cannot simply forward what a vendor sent. It has to compute the sky itself and fill in whatever the vendor left out.

That is a per-day, per-location astronomical calculation on the hot path of every forecast, for a one-person product on a fixed dyno budget. Under an async web server, request fibers share one Ruby CPU, so a few hundred microseconds of ephemeris math per forecast day competes with everything else the request is doing (see [Falcon and Ruby Async](/falcon-async-performance/) for that model). The path has to be correct, has to cover fields no vendor reliably supplies, and has to stay cheap. Getting all three took two failed optimizations.

## The Solution

Compute every sun and moon event in-house from a real astronomical engine, then reconcile it against the vendor field by field: the vendor's value when it exists, the computed one when it does not. The production engine is the `astronomy-engine` npm package, called from Ruby through a long-lived Node process; a pure-Ruby library stands in during development and test. Keeping that path off the CPU budget did not mean a general cache, because the two obvious caches both made things worse. What the measurements point to is narrower: precompute the one field that is identical for everyone on Earth, and interpolate it.

### Backfill only the field the vendor omitted

Vendors disagree about which astronomical fields they return (the same disagreement that drives the whole [multi-source adapter](/multi-source-api-adapter-pattern/)), so the reconciliation cannot be "vendor or engine" for the whole day. Swap the whole day to the engine whenever anything is missing and you discard accurate vendor values you were handed for free. Trust the vendor wholesale and you ship blank moon panels. It has to be per field.

The pattern is a single fallback expression per field, `day.x || backfill&.x`, with the vendor's parsed day on the left and the in-house engine on the right. Here is the shape, runnable, with a stubbed engine standing in for the real one:

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

The engine is lazy. `@backfill` is only touched when the left side is nil, so a day where the vendor supplied everything never runs it. The real reconciler adds one twist. A vendor that reports a named principal phase ("New", "First Quarter", "Full", "Last Quarter") is asserting an exact fraction, so a backfilled numeric phase that disagrees is clamped into that phase's range rather than overriding it. A computed value must never silently contradict a categorical one the source already committed to.

### Compute the sky in JavaScript, call it from Ruby

Astronomical calculations are a place where "roll your own" is a bad instinct. Rise and set times depend on atmospheric refraction and the apparent radius of the disk, twilight is an altitude search, and getting any of it subtly wrong is a bug you find from a customer screenshot months later. The most accurate, well-tested engine available is a JavaScript package, but the server is Ruby. So the engine runs in one long-lived Node process, and Ruby calls into it through a thin bridge that turns a JS function into a Ruby method. This mirrors production, trimmed to sunrise and sunset:

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

The detail to keep is the `null` returns. Above the Arctic Circle in summer the sun never sets, so `SearchRiseSet` returns nothing and every field reading through this can be nil. The reconciler treats a nil rise or set as a real answer. Polar regions are a supported location, not an edge case to crash on.

This bridge is also why the path costs what it does. The persistent Node process is warm memory, and the web server runs two Ruby worker processes per dyno, so there can be two warm JavaScript runtimes per dyno. That memory buys the engine's accuracy. It is also what made caching look irresistible.

### The cache that multiplied by worker count

The backfill is the same calculation over and over: one city recomputes the identical sky thousands of times an hour. That is the textbook case for a cache, and the pure-Ruby astronomy library ships one, a process-local least-recently-used cache, off by default. It was enabled in production with a conservative 1,000-entry cap, expecting lower CPU for a little memory.

Both halves were wrong, and it was rolled back within days. Ruby CPU still spiked under real traffic, because the cache's own bookkeeping and the surrounding Ruby work were themselves the cost, not just the astronomy math. And the memory was not a little. A process-local cache is not shared between worker processes, so its cost multiplies by how many you run. The cap held under a megabyte of live objects. After garbage collection, resident memory told a different story:

```ruby
ENTRIES        = 1_000   # LRU cap enabled in the production canary
LIVE_MB        = 0.9     # live ObjectSpace the entries actually held
AFTER_GC_MB    = 31.6    # resident-memory growth per worker, post-GC
FALCON_WORKERS = 2       # Ruby worker processes per dyno

puts "live objectspace:       #{LIVE_MB} MB"      # what you reason about
puts "after-GC RSS / process: #{AFTER_GC_MB} MB"  # what you actually pay
puts "after-GC RSS / dyno:    #{AFTER_GC_MB * FALCON_WORKERS} MB"  # ~63 MB
```

The 0.9 MB is the number you reason about. The ~63 MB per dyno is the number that bills you: resident memory after garbage collection, multiplied by every process that holds a copy. The library's cache stays off, and re-enabling it now requires a fresh canary.

A second optimization failed around the same time and cut the other way. To drop the Node runtime and reclaim its warm-process memory, the pure-Ruby library was tried as the production engine, and it regressed CPU. The two reversals bracket the design space: the JS engine is CPU-cheap but memory-expensive, the Ruby engine the reverse, and a general cache over either paid in the wrong currency. The way out was not to cache the calculation but to stop doing most of it.

### Precompute the one field that is the same everywhere

Sunrise depends on where you are standing. Moon phase does not. The moon shows the same illuminated fraction to everyone on Earth at a given instant, so a moon-phase value needs no location key at all, and it changes slowly and smoothly. A quantity that is global and slow-moving can be sampled onto a grid once and read by interpolation forever after.

So the direction for the moon-phase half is a precomputed grid: sample the phase and illumination every 60 minutes of UTC across the forecast horizon, store the grid, and answer any instant by interpolating between the two nearest samples. Today that grid lives as a benchmark probe with a dedicated rake task, not yet in the serving path; the live reconciler still calls the engine per request. The probe exists so the switch has to prove its speed and accuracy against the engine before it lands, and those are the numbers below.

The lookup key is a UTC instant (the forecast day's local middle-of-day, resolved to UTC), never a plain calendar date, because a calendar date is a different real moment in each time zone. This block generates a grid and interpolates it, printing both timings. The `full_moon_phase` stand-in is pure Ruby so the block runs; production computes each sample through the astronomy engine above, which is why the real gap is far wider than this one:

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

The one non-obvious line is the seam unwrap. Phase is cyclic: it runs up to 1.0 and wraps to 0.0 at each new moon, so a naive average of 0.98 and 0.01 lands near 0.5 (a full moon) instead of near 0.0 (a new moon). Adding 1.0 before interpolating, then folding the result back, keeps it on the short arc.

On the real engine the trade is decisive. A grid lookup measured 0.003 ms against 0.284 ms for the full location-specific calculation, roughly a hundredfold, and a 60-minute grid reproduces the engine's phase and illumination with a maximum illumination error around five millionths, invisible to a UI that renders a whole-number percentage. The benchmark task regenerates those numbers on demand, so a later change to the step or interpolation has to re-prove itself the same way.

The same precision argument settles the sun's cache key. Sun times shift about 2.1 seconds per kilometer east to west, so rounding a location key to one decimal place (about 11 km) moves a sunrise by at most half a minute. iOS renders times to the minute, so that rounding is imperceptible. Carrying five decimals of longitude into a key that resolves a minute-granular time shreds the hit rate for accuracy no one can see.

## Results

- The forecast shows solar noon, civil twilight, moon phase and illumination, and moonrise and moonset on top of the two fields most vendors return.
- Two optimizations were tried in production and reverted, with the reasons recorded. The process-local cache spiked CPU anyway and turned ~0.9 MB of live objects into ~31.6 MB resident per worker, ~63 MB per dyno. The pure-Ruby engine regressed CPU. Neither returns without a fresh canary.
- The cost that stays is the warm Node process, up to two per dyno, paid for the engine's accuracy.
- The 60-minute UTC moon-phase grid benchmarked at roughly 0.003 ms per lookup against 0.284 ms for the full calculation, with illumination error far below what a percentage display can show. It is the committed direction, and the benchmark task is the gate before it replaces the per-request call.

## Lessons Learned

- **Backfill per field, not per record.** Vendors disagree about which fields they return. A per-field `||` fallback keeps every accurate vendor value and computes only the gaps; swapping the whole record on any miss discards data you were handed for free.
- **Benchmark a cache on resident memory per dyno.** Live object size per process is the wrong number. The bill is resident memory after garbage collection, times every worker process that holds its own copy.
- **Precompute what is global and smooth; key what is local at render precision.** Moon phase is identical everywhere and nearly linear over an hour, so a sampled grid beats both recomputing and caching. A display that rounds to the minute cannot see a location key finer than ~11 km.
- **Write down why a reverted change was reverted.** "The cache spiked CPU and multiplied RSS by worker count" stops the next person re-enabling it. A silently reverted commit invites the retry.

---

## How This Post Was Made

**Prompt 1:** "please kick off a big batch to look through all skills looking for other topics that might be interesting to blog about. we could look at git history, but I think since we've been using claude/codex for the last ~year we should have most of the interesting stuff built into the skills by now. however, you can also look at the changelog view in the iOS repo for other highlights that might be worth dispatching research about. come back to me with a list of possible topics (that haven't already been covered in the blog) …"

**Prompt 2:** "lets do 4, 20, 21, 22 -- the others I think are not worth it"

Ten Claude agents mined the iOS, web, and Android skills, the iOS changelog, and the plan indexes for uncovered topics; the owner picked four from the ranked list. This post was researched and drafted by one agent from the cited skills, plans, and code, under the why-then-how voice and self-contained-code brief settled in the previous localization batch, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The title lost its subtitle, the bolded rules in the body moved into Lessons Learned, Results now holds only what changed and what it cost, and the caveat that the moon grid is not yet in the serving path stays. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
