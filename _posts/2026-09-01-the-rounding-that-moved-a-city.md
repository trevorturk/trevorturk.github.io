---
layout: post
title: "The Rounding That Moved a City"
date: 2026-09-01 15:00:00 -0600
summary: "Truncating request coordinates from three decimals to two looked like a free CDN cache win. An offline sweep flipped a timezone by an hour, and a live sweep against a gazetteer-based provider moved a resolved location 357 kilometers. Neither change shipped, and the rule that replaced them is: never quantize a coordinate that is also an input."
tags: [caching, cdn, performance, failure-story, ruby]
---

## The Problem

Every weather request at [Hello Weather](https://helloweather.com) carries a latitude and longitude, and since June 2022 the server has truncated both to three decimal places before doing anything with them. Three decimals is about 111 meters. The truncated values go into the upstream request URL, and because the CDN in front of every vendor keys its cache on the full URL (see [CloudFront as an Infinite Cache](/cloudfront-cdn-architecture/)), that precision is also the size of a cache cell. Two people a block apart in a dense city are two cells, two origin fetches, and two billable vendor calls.

Truncating to two decimals instead makes each cell about 1.1 kilometers on a side, which collapses roughly a hundred fine cells into one coarse one. The change is one character on each of two lines. The cost plans ranked it among the higher-leverage levers available, on the reasoning that forecast grids are generally coarser than 111 meters anyway, so a 1.1 kilometer cell usually still lands in the same effective forecast cell. That reasoning is true for grid-based sources. It was tested twice in July 2026, and both tests came back against the change.

```ruby
class WeatherRequest
  def initialize(lat:, lon:)
    # Three decimals since 2022. The proposal was truncate(2) on both lines.
    @lat = lat.to_f.truncate(3)
    @lon = lon.to_f.truncate(3)
  end
end
```

The two lines matter beyond the cache because the same truncated pair feeds three consumers: the upstream weather request, the timezone lookup, and the sun and moon backfill described in [Compute the Sky Yourself](/compute-the-sky-yourself/). The plan for the change noted that the blast radius was therefore wider than cache keys. It did not yet say how wide.

## First Sweep: The Timezone Flip

The global version of the change went up as a draft PR on July 13, 2026, and the draft carried its own measurement. The question was whether the two non-cache consumers could tolerate a coordinate that moved up to 1.1 kilometers, so the check ran the real timezone and astronomy code paths offline against 16 hand-picked points: timezone borders in the US, the 49th parallel, dense metros on three continents, high latitudes, and a mountain city. For each point it truncated to three decimals and to two, resolved the timezone for each, and differenced the absolute sunrise and sunset instants using the same reference zone for both so only the coordinate effect showed.

The astronomy result was harmless. The largest sunrise shift was 8 seconds and the largest sunset shift was 11 seconds, both at Reykjavik, and the maximum surface move was 1,146 meters, as expected for a 0.01 degree cell. A whole-number percentage or a clock time cannot show that.

The timezone result was not. Two of the 16 points resolved to a different zone after truncation. One was cosmetic: a point on the Washington side of the 49th parallel moved from the Vancouver identifier to the Los Angeles identifier, with the same wall-clock offset on the reference date. The other was material: a point 733 meters inside Arizona, which does not observe daylight saving time, truncated across the state line into Denver's zone. In summer that is a 60 minute shift on every local time the user sees, sunrise and sunset included, for a location that never moved.

The draft's recommendation was that this needed a human accuracy decision, and it offered the decoupling: keep two decimals for the source URLs only, and keep feeding three-decimal coordinates to the timezone and astronomy paths. That would have removed the flip entirely. The draft stayed open with a warning in its title, and no one reached the decision, because thirteen days later a narrower version of the same idea was tested against live data and failed worse.

## Second Sweep: The Gazetteer

The narrower version came out of a support report, not a cost plan. Manual refresh was failing intermittently for one source in Chicago and working immediately after switching sources. The investigation found the mechanism: that source is the only one whose data calls block on a first-hop location lookup, cached weekly, and a manual refresh with live GPS lands on a fresh three-decimal cell often enough that the cold lookup occasionally exceeds the per-call timeout and takes the whole request with it. A local sweep of 20 Chicago cells also noticed that adjacent 111 meter cells frequently returned the same resolved location key.

From there the inference looked obvious. If neighbors share a key, three decimals is pure cache fragmentation on that lookup, and asking the lookup for two decimals would warm the cell for everyone nearby. A PR that did exactly that, on the location lookup only, was opened on July 26 and closed ten minutes later.

The premise was wrong in a specific way. Sharing a key with a neighbor 111 meters away says nothing about surviving a displacement of a kilometer, because this provider does not resolve coordinates on a grid at all. It snaps them to a discrete gazetteer of named places, at a resolution rank that in Chicago means neighborhoods rather than the city. Truncating to two decimals changed the resolved key for 2 of 5 Chicago test coordinates: one downtown point resolved to a different neighborhood, and one on the north side resolved to the adjacent one.

The adversarial sweep that followed used 133 coordinate pairs across timezone lines, country borders, dense metros, coastlines, and both hemispheres, 372 live calls with no errors. Each pair was the same point at three decimals and at two, and the comparison was on the resolved key, the timezone name, the timezone offset, and the resolved location's own coordinates.

```ruby
# Illustrative shape of the pair sweep, trimmed to the comparison.
require "json"
require "net/http"

Point = Struct.new(:name, :lat, :lon)

POINTS = [
  Point.new("Adelaide, western edge", -34.928, 138.600),
  Point.new("Blanc-Sablon, Quebec", 51.427, -57.131),
  Point.new("Chicago Loop", 41.878, -87.629),
  # ... points chosen to sit near lines the rounding might cross
]

def resolve(lat, lon)
  uri = URI("https://provider.example/locations?q=#{lat},#{lon}")
  body = JSON.parse(Net::HTTP.get(uri))
  {
    id: body["id"],
    zone: body.dig("timezone", "name"),
    offset: body.dig("timezone", "offset_hours"),
    at: [body.dig("position", "lat"), body.dig("position", "lon")]
  }
end

def km_between(a, b)
  rad = Math::PI / 180
  dlat = (b[0] - a[0]) * rad
  dlon = (b[1] - a[1]) * rad
  h = Math.sin(dlat / 2)**2 + Math.cos(a[0] * rad) * Math.cos(b[0] * rad) * Math.sin(dlon / 2)**2
  2 * 6371 * Math.asin(Math.sqrt(h))
end

POINTS.each do |p|
  fine = resolve(p.lat.truncate(3), p.lon.truncate(3))
  coarse = resolve(p.lat.truncate(2), p.lon.truncate(2))
  puts [
    p.name,
    fine[:id] == coarse[:id] ? "same id" : "ID CHANGED",
    fine[:zone] == coarse[:zone] ? "same zone" : "ZONE CHANGED",
    format("%+.1fh", coarse[:offset] - fine[:offset]),
    format("moved %.2f km", km_between(fine[:at], coarse[:at]))
  ].join("  ")
end
```

The fourth column is the one to watch. A grid provider moves the resolved point by at most the rounding distance, so the number stays under 1.2 kilometers by construction. A gazetteer provider can return any place in its index, so the number has no upper bound from the query at all.

The results:

- 10.8 percent of the adversarial points changed timezone name, and 5.4 percent changed the actual wall-clock offset, by between half an hour and two and a half hours. A point on the western edge of Adelaide resolved to Perth after truncation, an hour and a half earlier and 357 kilometers away. A point at Blanc-Sablon, Quebec, gained two and a half hours. Other flips landed on the South Australia and New South Wales line, on the Sweden and Finland border, and on the Estonia and Russia border.
- The error was not bounded by the rounding distance. A query displacement of at most 1.1 kilometers moved the resolved location by up to 5.93 kilometers.
- The damage was not confined to boundaries. An unbiased sample of 40 points inside major metros saw no timezone changes, but half of them changed resolved key, and about 15 percent of those returned materially different weather. In one city the day-one forecast went from showers to intermittent clouds.
- The US timezone lines were clean on every metric, 15 of 15. The risk was international, and it was specific to how this provider resolves a point.
- Truncation always displaces toward the equator and the prime meridian, so the two coordinates of a pair are never rounded in a neutral direction. Rounding to nearest instead of truncating cut the error roughly in half at an identical cache benefit. A milder 0.005 degree quantization cut timezone differences from 10 to 1 and key differences from 32 to 11 while still collapsing cells about 25 fold.

The timezone flips were the disqualifying line. The forecast's timezone is read from this same lookup payload, and the lookup is cached weekly, so a confidently wrong zone would mis-bucket every day of the forecast and stay wrong for days. The fallback added the previous day for a missing or unparseable zone name offered no protection, because a valid name for the wrong place passes straight through.

The milder quantization and the round-instead-of-truncate variant were both strictly worse than the option that replaced them, so neither was pursued.

## What Shipped Instead

The same lookup URL carried two query parameters that provably did not affect its response: a units flag the location endpoint does not document, and a language parameter that changes only the response language. Recorded fixtures at one coordinate in two languages were byte-identical on every field the app reads. Dropping both collapsed the lookup's cache entries from one per language and unit combination to one per coordinate, at zero fidelity cost, because the queried coordinate never moves. That shipped on July 26, and the support reports did not recur through the watch window.

A local cache of the coordinate-to-key mapping was considered and rejected the same day. Keyed at the same granularity, it has the same miss rate as the CDN, and misses were the failure. It would only have made warm hits cheaper.

The rule went into the CDN caching skill that agents load before touching cache keys: never quantize request coordinates to share cache cells, because at least one provider resolves through a gazetteer rather than a grid, and the timezone it returns is cached for a week. The only lever that would make quantization safe on that provider is asking it for city-level rather than neighborhood-level resolution, which is a deliberate content change needing source QA, not a cache tweak.

## Results

- Neither rounding change shipped. The vendor-specific PR lived ten minutes. The global draft is still open with a do-not-merge title and a status note pointing at the refutation.
- The cost lever the plans ranked highly remains unclaimed, and the plan now says that if it is ever revisited it must be evaluated per source, with gazetteer-based providers excluded or gated on city-level resolution.
- The parameter drop shipped in the same PR that recorded the refutation, and it is the only change from the episode that raised the cache hit rate.
- The cost was two sweeps and a support investigation. The offline sweep of 16 points ran the real timezone and astronomy paths; the live sweep spent 372 origin calls.

## Lessons Learned

- A cache-key rounding is only safe when the upstream resolves on a grid at least as coarse as the rounding. If it snaps to a gazetteer, the error has no bound from your query.
- Before rounding a value, list every consumer of it. A coordinate that also picks a timezone or drives an ephemeris has a blast radius the cache never sees.
- Round versus truncate is a correctness choice. Truncation displaces every point in the same direction, and it roughly doubles the error for the same cache benefit.
- Sharing behavior between adjacent cells does not extrapolate. Measure the actual displacement you propose, on the actual provider, with points chosen to sit near the lines you might cross.
- Look for the parameter that does not change the response before you coarsen the parameter that does.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent surveyed the web repository and proposed this post among a batch of candidates; a second agent wrote it. Sources in the web repository: `plans/cache-optimization-coordinate-rounding.md`, `plans/accuweather-refresh-investigation.md` ("Coordinate Rounding: Refuted", 2026-07-26), the CDN caching skill's "Coordinate Precision" rule, `app/models/api/weather.rb`, the draft coordinate-rounding PR of 2026-07-13 and its impact document (16-point offline sweep, reference date 2026-07-08), the vendor-specific PR opened and closed on 2026-07-26, and the merged investigation PR of the same day. The three-decimal truncation dates to June 2022 per `git log -S`. The iOS client's cache key at three decimals was confirmed in its `Location` model.

Judgment calls: the provider is described as "a gazetteer-based provider" and is not named, its resolution rank and query parameters are described in words, and the second source in the support report is not named. City and neighborhood names are public geography and kept; the per-city forecast values from the metro sample are reduced to one qualitative example so they do not read as a vendor accuracy claim. The sweep script in the repository history was not preserved as a file, so the code block is an illustrative reconstruction of its shape and is labeled as such. The truncation excerpt is trimmed to the two lines and renamed.
