---
layout: post
title: "Compute the Sky Yourself"
date: 2026-08-25 08:10:00 -0600
summary: "Why our weather app computes its own sun and moon events, how filling in only the fields the vendor left out keeps that cheap, and two failed optimizations that pointed us at a precomputed moon-phase grid."
tags: [ruby, performance, api-design, architecture]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
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
