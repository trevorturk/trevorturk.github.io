---
layout: post
title: "Reserving the Top of the Scale"
date: 2026-07-29 09:30:00 -0600
summary: "How we keep the top of a 0-1 ranking scale for real hazards: thresholds from public agencies, everything else capped below the band, two forecast hours before a noisy input counts, and scores computed on the server."
tags: [api-design, ranking, ruby, ux]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
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
