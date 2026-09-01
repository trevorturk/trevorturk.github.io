---
layout: post
title: "Reserving the Top of the Scale"
date: 2026-07-29 09:30:00 -0600
summary: "A hazard band for auto-ranked cards: reserve the top of a 0-1 salience scale for severity anchored to public authorities, cap editorial signals below the floor, require corroboration from noisy inputs, and ship the scores from the server."
tags: [api-design, ranking, ruby, ux]
---

## The Problem

[Hello Weather](https://helloweather.com) shows a grid of stat cards: wind, air quality, UV, visibility, pressure, and more. Only a few fit above the fold. Which one goes first?

The obvious approach is to score each stat from 0 to 1 and sort descending. Scoring is easy. Deciding what the numbers *mean* is where taste-based ranking falls apart:

1. **Everything wants to be first.** Every stat's author believes their stat matters. Left unconstrained, each one gets nudged upward until they all cluster near 1.0 and the ordering is noise.
2. **Editorial boosts crowd out real hazards.** Showing the moon card on the night of a full moon is a nice touch. So is showing the temperature card during the morning commute. Neither should ever outrank a dense fog advisory, but if both live on the same unstructured 0-1 scale, eventually one will.
3. **Rank contradicts the label.** A card that says **UV 1 - Low** has no business being second on screen. Users read placement as importance. When placement and text disagree, the text loses and the ranking looks broken.
4. **Noisy inputs cause flapping.** One forecast hour with an implausible feels-like spike should not promote a card to the top of the screen and demote it on the next refresh.
5. **Alert fatigue is permanent.** If the top slot is sometimes an urgent hazard and sometimes a pleasant fact about the moon, users stop reading it as urgent. No amount of tuning gets that back.

The last one is the real constraint. A ranking scale is a communication channel, and the top of it can only mean one thing.

## The Solution

Scores are computed on the server, one per stat, each a float from 0 to 1. The client sorts and fills its auto slots; user-pinned slots always win. Ordering changes deploy without an app release.

Five rules:

1. **Reserve the top band** (0.9-1.0) for severity anchored to external public authorities.
2. **Cap subjective signals** permanently below the band floor.
3. **Require corroboration** before a noisy input may enter the band.
4. **Give every stat a boring tier** so rank cannot contradict the card's own label.
5. **Ship scores server-side** so ordering is deployable and forward-compatible.

This is not the full ranking algorithm. It is the invariant that keeps the rest of the algorithm honest.

## The Reserved Band

The entire mechanism is four lines:

```ruby
HAZARD_FLOOR = 0.9

def hazard_score(val, advisory, warning)
  fraction = (val - advisory) / (warning - advisory)
  round_f(HAZARD_FLOOR + [fraction, 1.0].min * (1.0 - HAZARD_FLOOR), 3)
end
```

A stat enters the band only when it crosses an **advisory** threshold, and it lands at 0.9. It reaches 1.0 at the **warning** threshold, interpolating linearly in between. Past warning, it clamps.

The interpolation is what makes co-occurring hazards sortable. If wind and air quality are both in advisory territory on the same afternoon, they rank by how deep into severity each one is, not by which stat someone decided was more important in the abstract.

Each hazard-capable score gets exactly one new line in its existing guard chain:

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

Read top to bottom: missing data, boring tier, hazard band, normal range. Every score in the file has the same shape, so the invariant can be audited by eye.

One trap the band cannot catch on its own: a raw zero from a sensor. Zero visibility scores as maximal dense-fog hazard, because that is what zero visibility *is*. Sources that mean "unknown" must send null. Any hazard band with a low-is-bad direction needs this rule written down, because a source that helpfully defaults missing values to 0 will manufacture an emergency.

### Anchoring to outside authorities

The thresholds are not our taste. Every advisory/warning pair cites a public source:

| Signal | Advisory (0.9) | Warning (1.0) | Authority |
|---|---|---|---|
| Air quality index | 101 | 201 | EPA AQI category boundaries |
| Wind gusts | 46 mph | 58 mph | NWS Wind Advisory / High Wind Warning |
| Visibility | 1/4 mi | 1/8 mi | NWS Dense Fog Advisory |
| UV index | 11 | 14 | WHO/EPA UV index scale ("extreme") |

The citations do most of the work. When someone asks why air quality outranks the moon, the answer is not "we thought it should." It is "the EPA calls 101 unhealthy for sensitive groups." Arguments about thresholds become arguments about which authority to cite, which is a better argument to have. It also leaves an upgrade path: national defaults today, regional or product-specific severity tomorrow, same band.

### Capping the editorial tier

Reserving the band only works if the cap is enforced. Anything driven by editorial judgment or comfort rather than a published severity threshold must be provably unable to reach 0.9. Two things do that work.

First, explicit constants below the floor:

```ruby
def moon_score
  case next_full_moon
    when today    then night? ? 0.85 : 0.6
    when tomorrow then night? ? 0.6  : 0.3
  else
    FALLBACK_SCORES[:moon]
  end
end
```

0.85, not 0.9. A full moon on a clear night is worth surfacing. It is never worth outranking a wind advisory.

Second, per-stat dampeners that cap a whole normal range:

```ruby
SCORE_DAMPENERS = {
  precip:      0.89,  # probability alone can never band
  pollen:      0.6,
  pressure:    0.3,   # comfort tier by design
  cloud_cover: 0.1,
  # ...
}
```

The dampener multiplies the normalized value, so a stat's normal range has a hard ceiling regardless of input. Precipitation at 100% probability scores 0.89: one notch below the floor, close enough to sort above almost everything, and structurally unable to claim hazard status on probability alone.

A subtle failure this caught: a bare `uv_index / 11.0` normalization breaches 0.9 at UV 10, entering the band without any threshold being crossed. Dividing by a max is not a cap. Any expression that can reach the floor needs an explicit dampener, and a test that asserts the ceiling for every non-hazard path is worth writing.

Pressure is the clearest example of a permanently capped stat. It is scored as the forecast drop from the current reading to the 8-hour low. Rate of change, not absolute value, is the signal people feel:

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

A 5 mb drop scores 0.15; a 10 mb drop scores 0.30. No public authority issues pressure advisories, so pressure never gets to act like one. A falling barometer *feels* urgent enough to justify an exception. Pressure does not get one.

## Corroboration and the Boring Tier

### Corroboration

Some inputs are too noisy for a single reading to be trusted with the top of the scale. Feels-like temperature and precipitation rate both come from forecast hours that can disagree wildly between refreshes. So a noisy signal bands only when **at least two hours in the window** cross the advisory threshold. Ruby's `many?` says this cleanly:

```ruby
current = currently.apparent_temperature_normalized
forecast = hourly.data.first(8).map(&:apparent_temperature_normalized).compact

heat = current.to_f >= heat_advisory || forecast.many? { |val| val >= heat_advisory }
cold = current.to_f <= cold_advisory || forecast.many? { |val| val <= cold_advisory }

return hazard_score([current, forecast.max].compact.max, heat_advisory, heat_warning) if heat
return hazard_score([current, forecast.min].compact.min, cold_advisory, cold_warning) if cold
```

Notice the asymmetry. A *current observation* over the threshold bands immediately, because it is measured rather than predicted. Only the forecast needs a second opinion. A stat with no current-conditions equivalent, precipitation rate for instance, has no shortcut and always needs two hours. Once corroborated, depth comes from the window peak: corroboration gates entry and does not dampen severity.

This creates a cliff. A single 104°F hour scores the same as a calm day. That is a deliberate trade, false calm over false alarm, and a graduated "near-hazard" tier is the obvious follow-up when the cliff starts to bite.

### The boring tier

The last rule fixes the rank-contradicts-label problem. Every stat gets an explicit tier that means *this card has nothing to say*, and it scores as that stat's fallback value:

```ruby
FALLBACK_SCORES = {
  aqi:         0.09,
  uv:          0.07,
  visibility:  0.05,
  pressure:    0.01,
  # ...
}
```

The fallbacks are tiny and distinct, so they double as the tiebreak ordering on a quiet day. The grid keeps a sensible default arrangement when nothing is happening.

The boring thresholds are anchored the same way the hazard thresholds are, at the other end of the published scale. UV at or below 2 is the WHO/EPA "Low" category. AQI at or below 50 is EPA "Good". Visibility above 6 miles is unambiguously clear. Precipitation below 20% probability sits under the lowest forecast term anyone bothers to express.

```ruby
return FALLBACK_SCORES[:uv] if val <= low  # UV <= 2, "Low"
```

One line per stat, and the bug where an evening UV reading of 1 lands second on screen becomes structurally impossible.

The boring tier creates its own cliff at tier entry, visibility jumping from 0.05 to 0.32 as it crosses out of "Good", and that is intended. The cliff *is* the card becoming rankable.

## Results

- Ranking changes ship without a client release. Thresholds live in one server-side file, so if a stat gets chatty in a particular season, raising its entry constant is a one-line deploy rather than an app review cycle.
- New signals are forward-compatible by default. When pressure scoring shipped, no client read the key. The client struct does not declare fields it does not know, so unknown score keys are ignored. Scores go live, get observed in real responses, and get tuned before any client work starts, and old clients keep working.
- Severity is auditable. Every value in the band traces to a citable threshold, so reviewing a scoring change means checking the citation, not relitigating intuitions.
- The failure modes moved rather than vanished. What remains is known and bounded: cliffs at tier boundaries, and near-zero conditions occasionally rounding below their own fallback. That replaces the unbounded failure of a scale where everything drifts toward 1.0.

## Lessons Learned

- **Reserve the top of any priority scale for severity an outside authority already called serious.** "The EPA calls this unhealthy" ends a design argument. "This feels important" starts one.
- **Cap by structure, not convention.** A comment saying "keep this below 0.9" will be violated. A constant multiplied into the expression cannot be. Then test the ceiling.
- **Interpolate within the band.** A flat 1.0 for "hazard" throws away the information you need when two hazards co-occur.
- **Corroborate noisy inputs: two predictions, or one measurement.** One predicted outlier is not an emergency, and the flapping it causes costs more trust than the missed alert.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and code, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The title lost its subtitle, the raw-zero trap moved from Results into the hazard-band section as that mechanism's limit, Results became four bullets, and Lessons Learned went from seven rules to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
