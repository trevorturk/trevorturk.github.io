---
layout: post
title: "Reserving the Top of the Scale: A Hazard Band for Auto-Ranked Cards"
date: 2026-07-29 09:30:00 -0600
summary: "A ranking pattern that reserves the top of a 0-1 salience scale for externally-anchored severity, caps editorial signals below the floor, and ships ordering from the server."
tags: [api-design, ranking, ruby, ux]
---

## The Problem

[Hello Weather](https://helloweather.com) shows a grid of stat cards - wind, air quality, UV, visibility, pressure, and more. Only a few fit above the fold. Which one goes first?

The obvious approach is to score each stat 0 to 1 and sort descending. That part is easy. The hard part is deciding what the numbers *mean*, and this is where taste-based ranking quietly falls apart:

1. **Everything wants to be first.** Every stat's author believes their stat matters. Left unconstrained, each one gets nudged upward until they're all clustered near 1.0 and the ordering is noise.
2. **Editorial boosts crowd out real hazards.** "Show the moon card on the night of a full moon" is a nice touch. "Show the temperature card during the morning commute" is a nice touch. Neither should ever outrank a dense fog advisory - but if both live on the same unstructured 0-1 scale, eventually one will.
3. **Rank contradicts the label.** A card that says **UV 1 - Low** has no business being the second card on screen. Users read placement as importance; if placement and text disagree, the text loses and the ranking looks broken.
4. **Noisy inputs cause flapping.** One forecast hour with an implausible feels-like spike shouldn't promote a card to the top of the screen, then demote it on the next refresh.
5. **Alert fatigue is permanent damage.** If the top slot is sometimes an urgent hazard and sometimes a pleasant fact about the moon, users stop reading the top slot as urgent. That's not recoverable by tuning.

The last one is the real constraint. A ranking scale is a communication channel, and the top of it can only mean one thing.

## The Solution

Scores are computed on the server, one per stat, each a float from 0 to 1. The client sorts and fills its auto slots (user-pinned slots always win). Ordering changes deploy without an app release.

The design has five rules:

1. **Reserve the top band** (0.9-1.0) for severity anchored to external public authorities.
2. **Cap subjective signals** permanently below the band floor.
3. **Require corroboration** before a noisy input may enter the band.
4. **Give every stat a boring tier** so rank can't contradict the card's own label.
5. **Ship scores server-side** so ordering is deployable and forward-compatible.

None of this is the full ranking algorithm - it's the invariant that keeps the rest of the algorithm honest.

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

That linear interpolation is what makes co-occurring hazards sortable. If wind and air quality are both in advisory territory on the same afternoon, they rank by *how deep into severity* they are, not by which stat someone decided was more important in the abstract.

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

Read top to bottom: missing data, boring tier, hazard band, normal range. Every score in the file has the same shape, which makes the invariant auditable by eye.

### Anchoring to outside authorities

The thresholds are not our taste. Every advisory/warning pair cites a public source:

| Signal | Advisory (0.9) | Warning (1.0) | Authority |
|---|---|---|---|
| Air quality index | 101 | 201 | EPA AQI category boundaries |
| Wind gusts | 46 mph | 58 mph | NWS Wind Advisory / High Wind Warning |
| Visibility | 1/4 mi | 1/8 mi | NWS Dense Fog Advisory |
| UV index | 11 | 14 | WHO/EPA UV index scale ("extreme") |

This is the load-bearing part of the pattern. When someone asks "why does air quality outrank the moon?", the answer isn't "we thought it should" - it's "the EPA calls 101 unhealthy for sensitive groups." Arguments about thresholds become arguments about which authority to cite, which is a much better argument to have. It also gives you an upgrade path: national defaults today, regional or product-specific severity tomorrow, same band.

### Capping the editorial tier

The complement of reserving the band is enforcing the cap. Anything driven by editorial judgment or comfort - rather than a published severity threshold - must be provably unable to reach 0.9.

Two things do that work. First, explicit constants below the floor:

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

0.85, not 0.9. A full moon on a clear night is genuinely worth surfacing; it is never worth outranking a wind advisory.

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

The dampener multiplies the normalized value, so a stat's normal range has a hard ceiling regardless of input. Precipitation at 100% probability scores 0.89 - one notch below the floor, close enough to sort above almost everything, structurally incapable of claiming hazard status on probability alone.

A subtle failure this caught: a bare `uv_index / 11.0` normalization breaches 0.9 at UV 10, quietly entering the band without any threshold being crossed. Dividing by a max isn't a cap. Any expression that can reach the floor needs an explicit dampener, and it's worth writing a test that asserts the ceiling for every non-hazard path.

Pressure is the clearest example of a permanently-capped stat. It's scored as the forecast drop from the current reading to the 8-hour low - rate of change, not absolute value, since that's the signal people actually feel:

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

A 5 mb drop scores 0.15; a 10 mb drop scores 0.30. No public authority issues pressure advisories, so pressure never gets to act like one - even though a falling barometer is exactly the kind of signal that *feels* urgent enough to justify an exception. Declining that exception is the whole discipline.

## Corroboration and the Boring Tier

### Corroboration

Some inputs are noisy enough that a single reading shouldn't be trusted with the top of the scale. Feels-like temperature and precipitation rate both come from forecast hours that can disagree wildly between refreshes.

The rule: a noisy signal bands only when **at least two hours in the window** cross the advisory threshold. Ruby's `many?` says this cleanly:

```ruby
current = currently.apparent_temperature_normalized
forecast = hourly.data.first(8).map(&:apparent_temperature_normalized).compact

heat = current.to_f >= heat_advisory || forecast.many? { |val| val >= heat_advisory }
cold = current.to_f <= cold_advisory || forecast.many? { |val| val <= cold_advisory }

return hazard_score([current, forecast.max].compact.max, heat_advisory, heat_warning) if heat
return hazard_score([current, forecast.min].compact.min, cold_advisory, cold_warning) if cold
```

Note the asymmetry: a *current observation* over the threshold bands immediately, because it's measured rather than predicted. Only the forecast needs a second opinion. A stat with no current-conditions equivalent - precipitation rate, for instance - has no such shortcut and always needs two hours.

Once corroborated, depth comes from the window peak. Corroboration gates entry; it doesn't dampen severity.

This does create a cliff: a single 104°F hour scores the same as a calm day. That's a deliberate trade - false calm over false alarm - and a graduated "near-hazard" tier is the obvious follow-up when the cliff starts to bite.

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

The fallbacks are tiny and distinct, so they double as the tiebreak ordering on a quiet day - the grid still has a sensible default arrangement when nothing is happening.

The boring thresholds are anchored the same way the hazard thresholds are, just at the other end of the published scale: UV at or below 2 is the WHO/EPA "Low" category. AQI at or below 50 is EPA "Good". Visibility above 6 miles is unambiguously clear. Precipitation below 20% probability sits under the lowest forecast term anyone bothers to express.

```ruby
return FALLBACK_SCORES[:uv] if val <= low  # UV <= 2, "Low"
```

One line per stat, and the class of bug where an evening UV reading of 1 lands second on screen becomes structurally impossible.

The boring tier creates its own cliff at tier entry - visibility jumping from 0.05 to 0.32 as it crosses out of "Good" - and that's the point. The cliff *is* the card becoming rankable.

## Results

**Ranking changes ship without a client release.** Thresholds live in one server-side file. If a stat gets chatty in a particular season, raising its entry constant is a one-line deploy, not an app review cycle.

**New signals are forward-compatible by default.** When pressure scoring shipped, no client read the key. The client struct simply doesn't declare fields it doesn't know, so unknown score keys are ignored. Server-side scores can go live, be observed in real responses, and be tuned before any client work starts - and old clients keep working forever.

**Severity is auditable.** Every value in the band traces to a citable threshold. Reviewing a scoring change means checking the citation, not relitigating intuitions.

**The failure modes moved.** The remaining rough edges are known and bounded - cliffs at tier boundaries, near-zero conditions occasionally rounding below their own fallback - rather than the unbounded failure of a scale where everything drifts toward 1.0.

**One trap worth naming:** a raw zero from a sensor. Zero visibility scores as maximal dense-fog hazard, because that's what zero visibility *is*. Sources that mean "unknown" must send null. Any hazard band with a low-is-bad direction needs this rule written down, because a source that helpfully defaults missing values to 0 will manufacture an emergency.

## Lessons Learned

- **Reserve the top of any priority scale for externally-anchored severity.** The moment editorial choices can reach the top, the top stops meaning anything. Escalation is a finite resource; spend it only on things an outside authority already called serious.

- **Cite an authority, don't pick a number.** "The EPA calls this unhealthy" ends a design argument. "This feels important" starts one. Anchored thresholds also localize and evolve gracefully - swap the authority, keep the structure.

- **Cap subjective signals structurally, not by convention.** A comment saying "keep this below 0.9" will be violated. A dampener constant multiplied into the expression cannot be. Then test the ceiling.

- **Interpolate within the band.** A flat 1.0 for "hazard" throws away the information you need when two hazards co-occur. Linear positioning between advisory and warning makes severity sortable for free.

- **Require corroboration from noisy inputs.** Two hours over threshold, or one measured observation. One predicted outlier is not an emergency, and the flapping it causes costs more trust than the missed alert.

- **Give every metric a quiet tier.** If a card can say "Low" while ranking second, your ranking has a bug that no amount of tuning fixes. An explicit boring tier that scores as the fallback makes rank and label agree by construction.

- **Ship the scores, not the ordering.** Server-computed scores plus clients that ignore unknown keys means ranking is a deploy, new signals roll out invisibly, and old clients never break.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and code, then reviewed before publishing.
