---
layout: post
title: "Reserving the Top of the Scale"
date: 2026-07-29 09:30:00 -0600
summary: "How we keep the top of a 0-1 ranking scale for real hazards: thresholds from public agencies, everything else capped below the band, two forecast hours before a noisy input counts, and scores computed on the server."
tags: [api-design, ranking, ruby, ux]
---

## The Problem

[Hello Weather](https://helloweather.com) shows a grid of stat cards: wind, air quality, UV, visibility, pressure, and more. Only a few fit above the fold. Which one goes first?

The obvious approach is to score each stat from 0 to 1 and sort. Scoring is easy. The hard part is deciding what the numbers *mean*, and that's where a ranking built on taste falls apart:

1. **Everything wants to be first.** Whoever wrote a stat thinks it matters. Without a rule, each score gets nudged up until they all sit near 1.0 and the order is noise.
2. **Nice touches crowd out real hazards.** Showing the moon card on the night of a full moon is a nice touch. So is showing the temperature card during the morning commute. Neither should ever outrank a dense fog advisory, but if they share one unstructured 0-1 scale, one day one will.
3. **Rank contradicts the label.** A card that says **UV 1 - Low** has no business being second on screen. People read position as importance. When the position and the text disagree, the text loses and the ranking looks broken.
4. **Noisy inputs cause flapping.** One forecast hour with an implausible feels-like spike shouldn't push a card to the top of the screen and drop it again on the next refresh.
5. **Alert fatigue is permanent.** If the top slot is sometimes an urgent hazard and sometimes a pleasant fact about the moon, people stop reading it as urgent. No amount of tuning gets that back.

The last one is the real constraint, so the top of the scale has to mean one thing only.

## The Solution

The server computes one score per stat, a float from 0 to 1. The client sorts and fills its automatic slots, and slots the user pinned always win. Because the scores come from the server, we can change the order without an app release.

Five rules:

1. **Reserve the top band** (0.9-1.0) for severity that a public agency has already defined.
2. **Cap subjective signals** permanently below the band floor.
3. **Require corroboration** before a noisy input can enter the band.
4. **Give every stat a boring tier** so rank can't contradict the card's own label.
5. **Ship scores from the server** so ordering is deployable and old clients keep working.

This isn't the whole ranking algorithm. It's the rule that keeps the rest of it honest.

## The Reserved Band

The whole mechanism is four lines:

```ruby
HAZARD_FLOOR = 0.9

def hazard_score(val, advisory, warning)
  fraction = (val - advisory) / (warning - advisory)
  round_f(HAZARD_FLOOR + [fraction, 1.0].min * (1.0 - HAZARD_FLOOR), 3)
end
```

A stat enters the band when it crosses an **advisory** threshold, and it lands at 0.9. It reaches 1.0 at the **warning** threshold, and in between it's a straight line (linear interpolation). Past warning, it stays at 1.0.

The straight line matters when two hazards happen at once. If wind and air quality are both past advisory on the same afternoon, they rank by how far into severity each one is, not by which stat someone decided was more important in the abstract.

Each score that can reach the band gets one new line in its existing chain of guards:

```ruby
def aqi_score
  max = 300.0
  good = 50.0
  advisory = 101.0
  warning = 201.0

  val = currently.aqi || daily.data.first.aqi

  return DATA_MISSING_SCORE if val.nil?
  return FALLBACK_SCORES[:aqi] if val <= good
  return hazard_score(val, advisory, warning) if val >= advisory
  val = [val, max].min

  round_f(val / max)
end
```

Read top to bottom: missing data, boring tier, hazard band, normal range. Every score in the file has the same shape, so you can check the rule by eye.

One trap the band can't catch on its own is a raw zero from a sensor. Zero visibility scores as the worst possible fog, because that's what zero visibility *is*. A source that means "unknown" has to send null. Any band where low is bad needs this rule written down, because a source that defaults missing values to 0 will invent an emergency.

### Anchoring to outside authorities

The thresholds aren't our taste. Every advisory/warning pair cites a public source:

| Signal | Advisory (0.9) | Warning (1.0) | Authority |
|---|---|---|---|
| Air quality index | 101 | 201 | EPA AQI category boundaries |
| Wind gusts | 46 mph | 58 mph | NWS Wind Advisory / High Wind Warning |
| Visibility | 1/4 mi | 1/8 mi | NWS Dense Fog Advisory |
| UV index | 11 | 14 | WHO/EPA UV index scale ("extreme") |

The citations do most of the work. When someone asks why air quality outranks the moon, the answer isn't "we thought it should." It's "the EPA calls 101 unhealthy for sensitive groups." Arguments about thresholds become arguments about which authority to cite, and that's a better argument to have. It also leaves room to grow: national defaults today, regional or product-specific thresholds tomorrow, same band.

### Capping the editorial tier

Reserving the band only works if the cap holds. Anything driven by editorial judgment or comfort rather than a published threshold has to be unable to reach 0.9, and we want the code to prove it. Two things do that.

First, explicit constants below the floor:

```ruby
def moon_score
  case astronomy.next_full_moon
    when timezone.now.to_date      then night?(timezone.now) ? 0.85 : 0.6
    when timezone.tomorrow.to_date then night?(timezone.now) ? 0.6 : 0.3
  else
    FALLBACK_SCORES[:moon]
  end
end
```

0.85, not 0.9. A full moon on a clear night is worth showing. It's never worth outranking a wind advisory.

Second, a per-stat dampener that caps a whole normal range:

```ruby
SCORE_DAMPENERS = {
  precip:      0.89,  # probability alone can never band
  pollen:      0.6,
  pressure:    0.3,   # comfort tier by design
  cloud_cover: 0.1,
  # ...
}
```

The dampener multiplies the normalized value, so a stat's normal range has a hard ceiling whatever the input. Precipitation at 100% probability scores 0.89: one notch below the floor, high enough to sort above almost everything, and unable to claim hazard status on probability alone.

This caught one subtle bug. A bare `uv_index / 11.0` passes 0.9 at UV 10, so it entered the band without crossing any threshold. Dividing by a maximum isn't a cap. Any expression that can reach the floor needs an explicit dampener, and a test that asserts the ceiling for every non-hazard path is worth writing.

Pressure is the clearest example of a permanently capped stat. We score it as the forecast drop from the current reading to the lowest of the next 8 hours. People feel the rate of change, not the absolute value:

```ruby
def pressure_score
  max = 10.0
  migraine = 5.0

  base = currently.pressure_normalized
  low = hourly.data.first(8).map(&:pressure_normalized).compact.min

  return DATA_MISSING_SCORE if base.nil? || low.nil?
  val = base - low

  return FALLBACK_SCORES[:pressure] if val < migraine
  val = [val, max].min * SCORE_DAMPENERS[:pressure]

  round_f(val / max)
end
```

A 5 mb drop scores 0.15; a 10 mb drop scores 0.30. No public authority issues pressure advisories, so pressure never gets to act like one. A falling barometer *feels* urgent enough to deserve an exception. It doesn't get one.

## Corroboration and the Boring Tier

### Corroboration

Some inputs are too noisy to trust a single reading with the top of the scale. Feels-like temperature and precipitation rate both come from forecast hours that can change a lot between refreshes. So a noisy signal enters the band only when **at least two hours in the window** cross the advisory threshold. ActiveSupport's `many?` says this in one word:

```ruby
current = currently.apparent_temperature_normalized
forecast = hourly.data.first(8).map(&:apparent_temperature_normalized).compact

heat = current.to_f >= heat_advisory || forecast.many? { |val| val >= heat_advisory }
cold = current.to_f <= cold_advisory || forecast.many? { |val| val <= cold_advisory }

return hazard_score([current, forecast.max].compact.max, heat_advisory, heat_warning) if heat
return hazard_score([current, forecast.min].compact.min, cold_advisory, cold_warning) if cold
```

Notice the asymmetry. A *current observation* over the threshold enters the band on its own, because it was measured rather than predicted. Only the forecast needs a second opinion. A stat with no current-conditions reading, precipitation rate for instance, always needs two hours. Once a signal is in, its depth comes from the peak of the window, so the two-hour rule decides entry and doesn't soften severity.

This creates a cliff. A single 104°F hour scores the same as a calm day. We chose that on purpose, false calm over false alarm, and a graduated "near-hazard" tier is the obvious follow-up when the cliff starts to hurt.

### The boring tier

The last rule fixes the rank-contradicts-label problem. Every stat gets a tier that means *this card has nothing to say*, and that tier scores as the stat's fallback value:

```ruby
FALLBACK_SCORES = {
  aqi:         0.09,
  uv:          0.07,
  visibility:  0.05,
  pressure:    0.01,
  # ...
}
```

The fallbacks are tiny and all different, so they also decide the order on a quiet day. The grid keeps a sensible default arrangement when nothing is happening.

The boring thresholds come from the same published scales as the hazard thresholds, at the other end. UV at or below 2 is the WHO/EPA "Low" category. AQI at or below 50 is EPA "Good". Visibility above 6 miles is clear by any standard. Precipitation below 20% probability is under the lowest forecast term anyone bothers to say.

```ruby
return FALLBACK_SCORES[:uv] if val <= low  # UV <= 2, "Low"
```

One line per stat, and an evening UV reading of 1 can no longer land second on screen.

The boring tier has its own cliff at its edge. Visibility jumps from 0.05 to 0.32 as it leaves "Good", and that's intended: the jump is the card becoming rankable.

## Results

- Ranking changes ship without a client release. The thresholds live in one server-side file, so if a stat gets chatty in a particular season, raising its entry constant is a one-line deploy instead of an app review cycle.
- New signals work with old clients by default. When pressure scoring shipped, no client read the key. The client struct doesn't declare fields it doesn't know, so it ignores unknown score keys. A score goes live, we watch it in real responses and tune it, and only then does client work start.
- Severity can be audited. Every value in the band traces to a citable threshold, so reviewing a scoring change means checking the citation, not arguing about intuitions again.
- The failure modes moved rather than vanished. What's left is known and bounded: cliffs at tier boundaries, and near-zero conditions sometimes rounding below their own fallback. That replaces the unbounded failure of a scale where everything drifts toward 1.0.

## Lessons Learned

- **Reserve the top of any priority scale for severity an outside authority already called serious.** "The EPA calls this unhealthy" ends a design argument. "This feels important" starts one.
- **Cap by structure, not convention.** A comment saying "keep this below 0.9" will be broken. A constant multiplied into the expression can't be. Then test the ceiling.
- **Interpolate within the band.** A flat 1.0 for "hazard" throws away the information you need when two hazards happen at once.
- **Corroborate noisy inputs: two predictions, or one measurement.** One predicted outlier isn't an emergency, and the flapping it causes costs more trust than the missed alert.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and code, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The title lost its subtitle, the raw-zero trap moved from Results into the hazard-band section as that mechanism's limit, Results became four bullets, and Lessons Learned went from seven rules to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. Every constant, threshold, citation, and score excerpt matched the server's scoring file and its skill; two small corrections: `many?` is ActiveSupport's, not core Ruby's, and the `moon_score` excerpt now uses the real code's date comparisons (`timezone.now.to_date`, `night?(timezone.now)`) instead of undefined `today`/`tomorrow` helpers, with the astronomy source named generically.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 3, run after batch 2 (#69) merged. The summary and every prose paragraph were rewritten in plain register: "the invariant" became "the rule", "interpolating linearly" got a plain definition at first use, "forward-compatible" became "old clients keep working", "worth surfacing" became "worth showing", third-person phrasing moved to "we", and contractions were added where a person would use them. Judgment calls: the closing line of The Problem ("A ranking scale is a communication channel...") was dropped as an aphorism and its point folded into the sentence before it; the quoted pair in the first Lessons Learned bullet stayed because it is quoted text. Prompts, verbatim:

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
