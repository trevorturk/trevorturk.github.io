---
layout: post
title: "CloudFront as an Infinite Cache"
date: 2026-02-27 14:00:00 -0600
summary: "We put a CloudFront distribution in front of each weather provider so repeated requests come from cache instead of the provider. The cache key is the part that took care to get right."
tags: [cloudfront, aws, caching, architecture]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

*Credit: This architecture idea came from [nickyleach](https://github.com/nickyleach).*

## The Problem

Most weather providers charge per request, and thousands of [Hello Weather](https://helloweather.com) users ask about the same places at the same times. When we called the providers directly, we paid for every one of those requests. Cost wasn't the only problem. Each provider has its own rate limits, and a busy hour could trip them. Each response took as long as the provider took, so some requests were fast and some were slow. When a provider went down, our users saw it right away.

We needed a cache that could take that load without us running more infrastructure.

## The Solution

We put CloudFront, Amazon's CDN, between the app and each provider and treat it as an "infinite cache": we don't run it, size it, or store anything ourselves. Each provider gets its own distribution, which is CloudFront's word for one cache with one server behind it. The app calls CloudFront. If CloudFront has the response, it answers from cache. If not, it calls the provider (the "origin," in CloudFront's terms), stores the answer, and returns it.

### Architecture

```
Client Request
    └─> Hello Weather API
        └─> Source Adapter
            └─> get(cache_level, url)
                └─> CloudFront Distribution (per source)
                    ├─> Cache HIT → Return cached response
                    └─> Cache MISS → Origin (upstream API)
                            └─> Cache response
                            └─> Return to client
```

One config file lists each provider's distribution and origin. A `CDN_ENABLED` flag decides which of the two hosts the adapter calls. The hosts below are made up:

```yaml
# config/cloudfront.yml
sources:
  accuweather:
    distribution_id: E1EXAMPLE
    cdn_host: https://d123abc.cloudfront.net
    origin_host: https://api.accuweather.example
  aeris_weather:
    distribution_id: E2EXAMPLE
    cdn_host: https://d456def.cloudfront.net
    origin_host: https://api.aeris.example
```

```ruby
class Api::Host
  CONFIG = YAML.load_file(Rails.root.join("config/cloudfront.yml")).freeze

  def self.host_for(key)
    CONFIG.dig("sources", key, cdn_enabled? ? "cdn_host" : "origin_host")
  end

  def self.cdn_enabled?
    ENV["CDN_ENABLED"] == "true"
  end
end
```

In development we leave `CDN_ENABLED` unset, so the app talks to the providers directly. Because each provider has its own distribution, a problem at one provider can't reach another provider's cache.

## Implementation

### The Cache Key Problem

CloudFront decides whether two requests are the same by looking at the URL plus any headers you tell it to include. That combination is the cache key. The providers' own `Cache-Control` headers don't help. Many providers send none, and the rest set expiry times too short to be worth caching. So we send our own expiry header and tell CloudFront to include it in the key. Our first version was wrong:

```ruby
# Different every request = always cache miss
headers["Cache-Expires"] = 1.hour.from_now.to_s
```

Each request computes its expiry from the moment it's made, so no two requests share a key. Nothing is ever a hit.

### The Cache-Expires Solution

Instead of "how long to cache," we send "when this expires," and we round that time to a boundary so every request in the same window agrees on it:

```ruby
CDN_USER_AGENT = "weathermachine.io"

def cdn_headers_for(cache_level, url = nil)
  cache_expires = case cache_level
  when :currently, :alerts
    Time.now.utc.beginning_of_hour + cdn_15_minutes + 1.second
  when :minutely
    Time.now.utc.beginning_of_minute + 1.minute + 1.second
  when :hourly
    Time.now.utc.beginning_of_hour + 1.hour + 1.second
  when :daily
    Time.now.utc.beginning_of_day + cdn_3_hours + 1.second
  when :astronomy, :pollen
    Time.now.utc.beginning_of_day + 1.day + 1.second  # next UTC date boundary
  when :weekly
    "v1"  # CDN min/max/default TTL is 1.week, but we send a version number
  else
    raise Api::Weather::NotImplementedError, cache_level
  end

  {
    "Cache-Env" => ENV.fetch("CDN_ENV", Rails.env),
    "Cache-Expires" => cache_expires.to_s,
    "Cache-Id" => account&.uuid,
    "User-Agent" => CDN_USER_AGENT,
  }.compact
end
```

Every time-based branch rounds down to a boundary, then adds one second. The extra second matters for a request made exactly on the boundary: it keys to the next window instead of the one that just closed. `:weekly` sends a version string instead of a time. CloudFront's TTL on every distribution is one week, so `v1` stays cached for that week, and we replace it by bumping the version number. `:astronomy` and `:pollen` were added in June 2026, after this post first went up, for data that changes once per calendar date. Requests made at different times inside the same window now send the same header:

```ruby
# Request at 01:05 UTC → Cache-Expires: 02:00 UTC (miss)
# Request at 01:59 UTC → Cache-Expires: 02:00 UTC (hit!)
# Request at 02:15 UTC → Cache-Expires: 03:00 UTC (miss)
# Request at 02:30 UTC → Cache-Expires: 03:00 UTC (hit!)
```

### Cache Levels

Each kind of data goes stale at its own rate, so each gets its own level:

| Level | Boundary | Max Cache Time | Use Case |
|-------|----------|----------------|----------|
| `:currently` | 15 min | ~15 min | Current conditions |
| `:minutely` | 1 min | ~1 min | Precipitation nowcast |
| `:hourly` | 1 hour | ~1 hour | Hourly forecast |
| `:daily` | 3 hours | ~3 hours | Daily forecast |
| `:alerts` | 15 min | ~15 min | Weather alerts |
| `:astronomy` / `:pollen` | next UTC date | ~24 hours | Sun/moon, pollen |
| `:weekly` | version | 1 week | URL-stable only: location keys, geocoding |

Sun and moon data ran on `:daily` until June 2026, and now has its own level. It was never a fit for `:weekly`, because the fixed `v1` key would serve the same moon phases all week. Anything tied to a calendar date needs a level that rolls over at the date boundary.

### 15-Minute Buckets

The `:currently` and `:alerts` levels use 15-minute windows. This helper picks the end of the current one:

```ruby
def cdn_15_minutes
  case Time.now.utc.min
  when 0..14  then 15.minutes
  when 15..29 then 30.minutes
  when 30..44 then 45.minutes
  when 45..59 then 60.minutes
  end
end
```

### The get() Helper

A source adapter, our wrapper around one provider's API, passes a cache level and a URL, and nothing else:

```ruby
def currently_data
  response = get(:currently, "https://#{host}/current?lat=#{lat}&lon=#{lon}")
  build_currently(response)
end
```

`get()` adds the cache headers and counts hits and misses:

```ruby
def get(cache_level, url, headers = {})
  headers.merge!(cdn_headers_for(cache_level, url)) if cdn_enabled?

  Api::AsyncHttp.get(url, headers, timeout: timeout).wait.tap do |response|
    case response.cdn
    when "hit"  then hit_tracker&.cache_hit
    when "miss" then hit_tracker&.source_hit
    end
  end.data
end
```

### Api::AsyncHttp

The HTTP client refuses any request to CloudFront that would get the cache key wrong:

```ruby
class Api::AsyncHttp
  CDN_NAME = "cloudfront"
  CDN_HEADER_NAME = "Cache-Expires"
  CDN_HEADERS_ALLOWED = [
    "Accept",
    "Authorization",
    "Cache-Env",
    "Cache-Expires",
    "Cache-Id",
    "User-Agent",
  ]

  def self.get(url, headers = nil, timeout: nil)
    validate_cdn_headers_if_cdn_host!(url, headers)
    # ... make request
  end

  # Since the CDN has a long default expiration, be careful when calling
  # Async::HTTP directly, making sure we don't hit CloudFront without the
  # Cache-Expires header, and if we're sending new headers, make sure to
  # include those in the CloudFront config.
  def self.validate_cdn_headers_if_cdn_host!(url, headers)
    return unless url.include?(CDN_NAME)

    if headers.keys.exclude?(CDN_HEADER_NAME)
      raise ArgumentError, "can't hit CloudFront without the cache busting header"
    end

    if headers.keys.any? { |header| CDN_HEADERS_ALLOWED.exclude?(header) }
      raise ArgumentError, "can't hit CloudFront with headers that aren't in the CloudFront config"
    end
  end
end
```

CloudFront forwards only the headers named in its config and drops the rest before the provider sees them. So a header added somewhere else in the code would disappear without an error. If someone then added it to the CloudFront config to fix that, it would become part of the cache key and split the cache. The allowlist in the client matches the config, so a header missing from either one raises where it's called. `Accept` and `Authorization` are on the list because a couple of providers need them.

### Additional Cache Key Headers

Besides `Cache-Expires`, we send three more headers that go into the key:

| Header | Purpose |
|--------|---------|
| `Cache-Env` | Prevent staging/production mixing |
| `Cache-Id` | Per-account isolation |
| `User-Agent` | Consistent across requests |

## Debugging Cache Behavior

Every CloudFront response says whether it came from cache:

```bash
curl -I "https://d123abc.cloudfront.net/endpoint?lat=41.87&lon=-87.62"

# Look for:
X-Cache: Hit from cloudfront   # Served from cache
X-Cache: Miss from cloudfront  # Fetched from origin
Age: 123                       # Seconds since cached
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Always miss | Different `Cache-Expires` each request | Use boundary times |
| Stale data | TTL too long | Reduce cache level |
| Account leakage | Missing `Cache-Id` | Include account UUID |
| Env mixing | Missing `Cache-Env` | Include environment |

## Related: Timeout Tuning

Each provider answers at its own speed, so each one needs its own timeout. [CloudFront Logging: Time-Boxed Investigations](/cloudfront-logging/) covers how we use CloudFront logs to set those timeouts.

## Results

- As of February 2026, 60-80% of requests for popular locations came from cache.
- Fewer requests reach the providers, so we pay them less.
- A short provider outage doesn't reach the app while the cache holds. Cached responses also come back faster than the provider would answer, because CloudFront serves them from its edge locations.

## Lessons Learned

- **One distribution per provider.** It's cheaper than tracking down one provider's bad responses in another's cache.
- **Reject unlisted cache headers in the client.** A stray header gets dropped or splits the key, and neither shows up as an error. An exception shows up in development.
- **Count hits and misses where you make the request.** Without a counter you can't see the savings, and you can't see when they drop.
- **Pick the cache window per data type.** How fresh the UI needs the data decides the boundary. One window can't serve both a rain nowcast and a moon phase.
