---
layout: post
title: "A Timeout That Knows What Time It Is"
date: 2026-09-01 13:00:00 -0600
summary: "Timeouts per source and per minute of the hour, picked by stepping up from a floor over CDN log samples until enough requests would have finished in time. Increases go straight into config; decreases have to hold run by run first."
tags: [ruby, async, falcon, timeouts, cloudfront, performance]
model: "Claude Fable 5.1"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Several of the sources behind [Hello Weather](https://helloweather.com) need twice the timeout in the eight minutes after the top of the hour that they need for the rest of it. In the current config, four sources sit at 1.5 seconds for most of the hour and 3.0 seconds in that window. The reason is our own traffic. Twice an hour a silent push wakes the install base, and every device refreshes within a few minutes. Our request volume spikes on a schedule we set, and so does every source's slow tail. [Heroku Capacity](/heroku-capacity/) covers what that does to dynos. This post is about what it does to the number we pass to `with_timeout`.

For three and a half years that number was a constant. In 2022 the client had a single 4-second timeout for every source. In November 2022 that became a 2-second default, with one source hardcoded to 5 seconds in its adapter. A single value is wrong in both directions. During the spike it's too tight, so calls that would have finished a few hundred milliseconds later raise `Async::TimeoutError` and take the error path. The rest of the hour it's too loose, so a slow source holds the request open long enough for the user to feel it.

The plan that started the work, written on February 24, 2026, recorded three facts from production log slices. Failures clustered around the 2.0-second boundary. Sources behaved differently from each other. The spike windows were predictable and repeated. That third fact matters most, because if we know in advance which minutes are expensive, the timeout can be a lookup instead of a guess.

## The Solution

Three parts, landed between February 26 and March 12, 2026:

- Timeouts live in config, keyed by source and by a window of the hour, and the fetch path looks them up at call time.
- A tool picks the number for each cell by searching over captured CDN log samples until a success target holds, and reports how well that number holds run by run.
- Applying a recommendation is asymmetric: increases go straight in, decreases have to earn it.

### A lookup keyed by source and minute

The alternative was a constant per source in each adapter, which is where the 5-second override had lived since 2022. That puts a tuning decision in code, one adapter at a time, and there's no way to vary it by time of day. With config keyed by source and window, every value sits in one file, and the recommendation tool can rewrite that file in place.

The runtime side is small. A host class loads the YAML at boot and answers two questions: which host serves a source, and what its timeout is right now. The code calls the window a mode.

```ruby
class Api::Host
  CONFIG = YAML.load_file(Rails.root.join("config/cloudfront.yml")).freeze

  def self.timeout_for(key)
    CONFIG.dig("timeouts", "sources", key, current_mode)
  end

  def self.current_mode
    case Time.now.utc.min
    when 0..7   then "spike_00"
    when 30..37 then "spike_30"
    else "normal"
    end
  end
end
```

The config has a defaults block and one entry per source with the three windows. This is the shape, with placeholder keys and only a few rows. The real file lists every source and its hosts, and we're not publishing that.

```yaml
timeouts:
  defaults:
    normal: 1.5
    spike_00: 1.5
    spike_30: 1.5
  sources:
    source_a:
      normal: 1.5
      spike_00: 3.0
      spike_30: 1.5
    source_b:
      normal: 2.7
      spike_00: 3.0
      spike_30: 3.0
```

The source base class asks for its own timeout on every fetch and hands it to the client. The client wraps the request in the reactor's cooperative timeout.

```ruby
class Api::Sources::Base
  def get(cache_level, url, headers = {}, parse: :json)
    Api::AsyncHttp.get(url, headers, timeout: timeout, parse: parse).wait.data
  end

  private

  def timeout
    Api::Host.timeout_for(key)
  end
end

class Api::AsyncHttp
  DEFAULT_TIMEOUT = ENV.fetch("DEFAULT_TIMEOUT", 2).to_i

  def self.get(url, headers = nil, timeout: nil, parse: :json)
    Async do |task|
      task.with_timeout(timeout || DEFAULT_TIMEOUT) do
        # request, status checks, body parse
      end
    rescue Async::TimeoutError
      raise Api::Weather::TimeoutError, URI(url).host
    end
  end
end
```

Both excerpts are trimmed to the timeout path. The mode is computed on every call, so a request that starts at minute 7 and one that starts at minute 8 get different budgets, and nothing is cached across the boundary. The windows are eight minutes long because that's our push schedule. A different app has different expensive minutes, and the case statement is the only place that knowledge lives. The lookup works alongside the fan-out in [Falcon and Ruby Async](/falcon-async-performance/): the barrier bounds the shape of a request, and the per-source timeout bounds each leg of it.

### Choosing the number

The obvious way to pick a timeout is to read a percentile off the latency distribution. The tool reports p95 and p99, but they aren't the decision. A percentile says how slow the tail is. The question we're asking is how much of the tail is worth waiting for. So the recommendation is a search instead: start at a floor, step up until enough of the samples would have finished within the candidate, and stop.

The samples come from CDN access logs. [CloudFront Logging](/cloudfront-logging/) covers the capture side. Each capture run is tagged with the window it fell in and stored as one row per request with how long it took. Given those rows for one source and one window, the recommendation is this, trimmed to the decision:

```ruby
def recommendation_for_source(latencies, target_success:, min_samples:, min_timeout:, max_timeout:, step:)
  sorted = latencies.sort
  return { recommended_timeout_sec: nil, issue: "insufficient_samples" } if sorted.length < min_samples

  timeout = min_timeout
  recommended = nil
  while timeout <= max_timeout
    rate = success_rate_at_timeout(sorted, timeout)
    if rate >= target_success
      recommended = timeout
      break
    end
    timeout = (timeout + step).round(10)
  end
  recommended ||= max_timeout

  { recommended_timeout_sec: recommended, estimated_success_rate: success_rate_at_timeout(sorted, recommended) }
end

def success_rate_at_timeout(sorted_values, timeout_sec)
  first_above = sorted_values.bsearch_index { |value| value > timeout_sec } || sorted_values.length
  first_above.to_f / sorted_values.length
end
```

The defaults are a 1.5-second floor, a 3.0-second cap, a 0.1-second step, at least 100 samples, and a target of 0.99 in the normal window or 0.995 in the spike windows. In practice the runbook passes 0.999. If the cap doesn't reach the target, the cap is the answer, and the estimated success rate says how far short it fell.

Pooled samples can hide a bad run. A source that met the target on twenty quiet days and missed it badly on one still looks fine in aggregate. So the stability variant groups the same samples by capture run, keeps runs with at least 100 samples, and reports how many of those runs individually meet the target at the recommended value, along with the worst and best run. The pooled number is the recommendation. The per-run counts tell us whether to trust it.

### Applying it asymmetrically

The writer command computes the recommendation for every source and window from the last 90 days of samples and rewrites the config in place. It doesn't decide whether the new values should stay. The operating rule, added on March 17, 2026 after the first full 90-day pass, is that we apply increases directly and we don't apply decreases without more evidence.

The reasoning is about what each mistake costs. A timeout set too high makes some users wait longer on a slow day. A timeout set too low drops requests that would have succeeded, in exactly the minutes when the app is under the most load. So before accepting a decrease, the runbook reads the stability JSON for that source and window. If few runs were considered, or the aggregate success is only just above target, we keep the prior value. The first pass kept two recommended reductions on that basis. The May 4 pass declined two more.

The floor and cap have their own evidence. Fixed-floor sweeps below 1.5 seconds increased dropped requests materially, mostly in the top-of-hour window, so 1.5 is the floor. The 3.0-second cap comes from response-time guidance rather than the logs. A weather refresh can tolerate more wait than tap feedback, but it still needs a bounded wait with visible progress.

One later pass shows what the search can't see on its own. On August 4, 2026 a source's plan change tripled the calls per fetch, from one to three concurrent origin calls, each racing the same timeout. The per-call drop rate at 1.5 seconds had risen from 0.04% to 0.26% in the normal window. That sounds small. But a fetch succeeds only if all three calls do, so the per-fetch drop rate is roughly the per-call rate times three: about 0.8% of normal-window fetches and about 4.5% in the top-of-hour window. The fix raised that source in all three windows, to 2.7, 3.0, and 3.0 seconds, and used the stricter 0.999 target, because per-fetch success is per-call success cubed.

The same pass drew a distinction the tool doesn't draw. Before the change, the samples over 1.5 seconds for that source were mostly CDN error rows averaging around 37 seconds. That's a dead origin, and no timeout value rescues it, so failing fast was right. After the change, errors had nearly vanished and the tail was real slow responses, mostly between 1.5 and 3.0 seconds. Only the second kind of tail is worth a longer timeout. The search only sees the distribution, so it would happily recommend 3.0 seconds for a dead origin. Someone has to look at what the tail is made of before raising a value.

## Results

- One constant became sixteen source entries with three windows each. Most rows sit at the 1.5-second floor in all three windows. Four sources are raised in the top-of-hour window only, three of them to the 3.0-second cap. One is raised in both spike windows, and two are raised in all three, one of them since August 2026.
- Four tuning passes between March 17 and August 4, 2026 changed nine values: seven increases and two decreases. Four more recommended decreases were declined on stability evidence.
- We didn't record a before-and-after drop rate for the system as a whole. The one measured change is the August raise, projected from the same samples to cut that source's normal-window drop rate from 0.26% to 0.09% and its top-of-hour rate from 1.52% to 0.60%.
- The cost is the logging. The original workflow turned CDN logging on for an investigation and off again. The rolling window needs it left on with 90-day retention, and the resulting store made the all-source dry run slow enough that we needed a source-scoped index and a per-source fallback.

## Lessons Learned

- Key timeouts by the minutes where your own load concentrates, not only by source. The expensive minutes are usually on your schedule, so they can be a lookup.
- Pick a timeout by searching up from a floor to a success target. A percentile describes the tail; the target says how much of it to wait for.
- Look at what the tail is made of before raising a value. A longer wait rescues slow responses. It doesn't rescue dead-origin errors, and failing fast is right for those.
- When a fetch fans out into N calls that must all succeed, set the per-call target to the per-fetch target to the power of 1/N. Otherwise the small percentages compound past what you meant to accept.
- Apply increases directly and make decreases show run-level evidence. The two mistakes cost different things, and only the cheap one should happen automatically.
