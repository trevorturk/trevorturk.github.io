---
layout: post
title: "The Rounding That Moved a City"
date: 2026-09-01 15:00:00 -0600
summary: "Truncating request coordinates from three decimals to two looked like an easy CDN cache win. An offline check moved one timezone by an hour, and a live check against a provider that snaps coordinates to named places moved a location 357 kilometers. Neither change shipped. The rule we kept instead: don't round a coordinate that something else also reads."
tags: [caching, cdn, performance, failure-story, ruby]
---

## The Problem

Every weather request at [Hello Weather](https://helloweather.com) carries a latitude and longitude. Since June 2022 our server has truncated both to three decimal places (dropping the extra digits, not rounding) before it does anything else with them. Three decimals is about 111 meters on the ground. The truncated values go into the URL we send upstream, and the CDN in front of every vendor keys its cache on the full URL (see [CloudFront as an Infinite Cache](/cloudfront-cdn-architecture/)). So 111 meters is also the size of a cache cell. Two people a block apart in a dense city are two cells, two origin fetches, and two vendor calls we pay for.

Truncating to two decimals instead makes each cell about 1.1 kilometers on a side, so roughly a hundred of the fine cells collapse into one. The change is one character on each of two lines. Our cost plans ranked it near the top of the levers we had. The reasoning was that forecast grids are usually coarser than 111 meters anyway, so a 1.1 kilometer cell would still land in the same forecast cell most of the time. That's true for sources that work on a grid. We tested the change twice in July 2026, and both tests came back against it.

```ruby
class WeatherRequest
  def initialize(lat:, lon:)
    # Three decimals since 2022. The proposal was truncate(2) on both lines.
    @lat = lat.to_f.truncate(3)
    @lon = lon.to_f.truncate(3)
  end
end
```

Those two lines matter beyond the cache because the same truncated pair feeds three things: the upstream weather request, the timezone lookup, and the sun and moon fields we compute ourselves, described in [Compute the Sky Yourself](/compute-the-sky-yourself/). The plan for the change noted that the effect would reach further than cache keys. It didn't yet say how far.

## First Sweep: The Timezone Flip

The version that changed every source went up as a draft PR on July 13, 2026, with its own test attached. The question was whether the timezone lookup and the sun and moon math could handle a coordinate that moved up to 1.1 kilometers. So the test ran the real code for both, offline, against 16 hand-picked points: timezone borders in the US, the 49th parallel, dense cities on three continents, high latitudes, and a mountain city. For each point it truncated to three decimals and to two, looked up the timezone for each, and compared the sunrise and sunset instants. Both were computed in the same reference zone, so the only difference was the coordinate.

The sun and moon side was fine. The largest sunrise shift was 8 seconds and the largest sunset shift was 11 seconds, both at Reykjavik. The largest move on the ground was 1,146 meters, which is what a 0.01 degree cell should give. A clock time or a whole-number percentage can't show a shift that small.

The timezone side was not fine. Two of the 16 points came back in a different zone after truncation. One didn't matter: a point on the Washington side of the 49th parallel moved from the Vancouver zone name to the Los Angeles one, and both have the same clock offset on the reference date. The other did. A point 733 meters inside Arizona, which doesn't observe daylight saving time, truncated across the state line into Denver's zone. In summer that's a 60 minute shift on every local time the user sees, sunrise and sunset included, for a location that never moved.

The draft said this needed a person to decide how much accuracy to give up, and it offered a way to split the difference: use two decimals in the source URLs only, and keep sending three-decimal coordinates to the timezone lookup and the sun and moon math. That would have removed the flip. The draft stayed open with a warning in its title. Nobody made the decision, because thirteen days later a narrower version of the same idea was tested against live data and failed worse.

## Second Sweep: The Gazetteer

The narrower version came out of a support report, not a cost plan. Manual refresh was failing now and then for one source in Chicago, and it worked right away after switching sources. We found the cause. That source is the only one that has to look up a location key before it can fetch weather, and we cache that lookup for a week. A manual refresh with live GPS lands on a fresh three-decimal cell often enough that the cold lookup sometimes runs past the per-call timeout and takes the whole request down with it. While checking 20 Chicago cells locally, we also noticed that cells 111 meters apart often came back with the same location key.

From there the next step looked obvious. If neighbors share a key, three decimals only splits that lookup's cache for no reason, and asking it for two decimals would warm the cell for everyone nearby. A PR that did that, on the location lookup only, was opened on July 26 and closed ten minutes later.

The premise was wrong in a specific way. Sharing a key with a neighbor 111 meters away says nothing about what happens when a point moves a kilometer, because this provider doesn't put coordinates on a grid at all. It snaps them to a gazetteer, a fixed list of named places, and at the level we ask for, that means neighborhoods in Chicago rather than the city. Truncating to two decimals changed the key for 2 of 5 Chicago test coordinates. One downtown point landed in a different neighborhood, and one on the north side landed in the one next door.

The sweep that followed was built to find failures. It used 133 coordinate pairs near timezone lines, country borders, dense cities, and coastlines, in both hemispheres, 372 live calls with no errors. Each pair was the same point at three decimals and at two. For each pair we compared the location key, the timezone name, the timezone offset, and the coordinates the provider returned for the place it picked.

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

The fourth column is the one to watch. A grid provider moves the returned point by at most the truncation distance, so that number stays under 1.2 kilometers no matter what. A gazetteer provider can return any place in its list, so nothing about the query puts a ceiling on that number.

The results:

- 10.8 percent of the points changed timezone name, and 5.4 percent changed the clock offset itself, by between half an hour and two and a half hours. A point on the western edge of Adelaide came back as Perth after truncation, an hour and a half earlier and 357 kilometers away. A point at Blanc-Sablon, Quebec, gained two and a half hours. Other flips landed on the South Australia and New South Wales line, on the Sweden and Finland border, and on the Estonia and Russia border.
- The error was bigger than the truncation. Moving the query by at most 1.1 kilometers moved the returned location by up to 5.93 kilometers.
- The damage wasn't only at borders. A sample of 40 points inside big cities, picked without regard to borders, saw no timezone changes. But half of them changed location key, and about 15 percent of those came back with noticeably different weather. In one city the first day's forecast went from showers to intermittent clouds.
- The US timezone lines passed every measure, 15 of 15. The risk was outside the US, and it came from how this provider picks a place.
- Truncation always moves a point toward the equator and the prime meridian, so the error never averages out across the two coordinates. Rounding to the nearest value instead cut the error roughly in half with the same cache benefit. A milder step of 0.005 degrees cut timezone differences from 10 to 1 and key differences from 32 to 11 while still merging about 25 cells into one.

The timezone flips killed the idea. The forecast's timezone comes from this same lookup response, and we cache the lookup for a week, so a wrong zone would put every day of the forecast in the wrong bucket and stay wrong for days. The fallback we'd added the day before, for a missing or unreadable zone name, didn't help, because a valid name for the wrong place passes straight through.

The milder step and the round-instead-of-truncate variant were both worse on every count than what we shipped instead, so we dropped them.

## What Shipped Instead

The same lookup URL carried two query parameters that we could show didn't change its response: a units flag the location endpoint doesn't document, and a language parameter that changes only the language of the text. Recorded responses at one coordinate in two languages were the same bytes on every field the app reads. Dropping both took the lookup's cache from one entry per language and unit combination to one per coordinate. Nothing in the response changed, because the coordinate we send stays the same. That shipped on July 26, and the support reports didn't come back during the time we watched.

We considered a local cache of coordinate to key and rejected it the same day. Keyed at the same precision, it misses as often as the CDN does, and misses were the problem. It would only have made hits cheaper.

The rule went into the CDN caching skill, which agents load before they touch cache keys: don't round request coordinates to share cache cells, because at least one provider uses a gazetteer rather than a grid, and we cache the timezone it returns for a week. The one thing that would make rounding safe on that provider is asking it for city-level rather than neighborhood-level places. That changes what the user sees and needs QA on the source, so it's a content change, not a cache tweak.

## Results

- Neither truncation change shipped. The one-provider PR lived ten minutes. The all-sources draft is still open with a do-not-merge title and a status note pointing at the write-up that refuted it.
- The saving the plans ranked highly is still on the table. The plan now says that if anyone revisits it, they test it one source at a time, and gazetteer providers are left out unless we switch them to city-level places.
- Dropping the two parameters shipped in the same PR that recorded the refutation, and it's the only change from this episode that raised the cache hit rate.
- The cost was two sweeps and a support investigation. The offline sweep ran 16 points through the real timezone and sun and moon code. The live sweep spent 372 calls to the provider.

## Lessons Learned

- Rounding a cache key is only safe when the upstream works on a grid at least as coarse as the rounding. If it snaps to a list of named places, your query puts no limit on the error.
- Before rounding a value, list everything that reads it. A coordinate that also picks a timezone or feeds sunrise math can break things the cache never sees.
- Round versus truncate is a correctness choice. Truncation moves every point in the same direction, and it roughly doubles the error for the same cache benefit.
- What neighboring cells do tells you nothing about a bigger move. Measure the move you're proposing, on the provider you use, with points chosen to sit near the lines you might cross.
- Before you round the parameter that changes the response, look for one that doesn't and drop it.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent surveyed the web repository and proposed this post among a batch of candidates; a second agent wrote it. Sources in the web repository: `plans/cache-optimization-coordinate-rounding.md`, `plans/accuweather-refresh-investigation.md` ("Coordinate Rounding: Refuted", 2026-07-26), the CDN caching skill's "Coordinate Precision" rule, `app/models/api/weather.rb`, the draft coordinate-rounding PR of 2026-07-13 and its impact document (16-point offline sweep, reference date 2026-07-08), the vendor-specific PR opened and closed on 2026-07-26, and the merged investigation PR of the same day. The three-decimal truncation dates to June 2022 per `git log -S`. The iOS client's cache key at three decimals was confirmed in its `Location` model.

Judgment calls: the provider is described as "a gazetteer-based provider" and is not named, its resolution rank and query parameters are described in words, and the second source in the support report is not named. City and neighborhood names are public geography and kept; the per-city forecast values from the metro sample are reduced to one qualitative example so they do not read as a vendor accuracy claim. The sweep script in the repository history was not preserved as a file, so the code block is an illustrative reconstruction of its shape and is labeled as such. The truncation excerpt is trimmed to the two lines and renamed.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 1, run after the pilot (#67) merged. The prose moved to "we" and contractions, "gazetteer" is defined where it first appears, the live sweep is no longer called adversarial, "byte-identical" became "the same bytes", and "quantize", "displacement", "resolved", and "blast radius" became "round", "move", "came back as", and plain descriptions of what breaks. Judgment calls: "truncate" stays as the one name for the operation because the round-versus-truncate distinction is one of the post's facts, and "unbiased sample" became "a sample picked without regard to borders", which is what it meant. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"
