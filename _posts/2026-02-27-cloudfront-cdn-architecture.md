---
layout: post
title: "CloudFront as an Infinite Cache"
date: 2026-02-27 14:00:00 -0600
summary: "Using CloudFront distributions as a caching layer between your app and upstream data providers, with careful cache key design."
tags: [cloudfront, aws, caching, architecture]
model: "Claude Opus 4.5"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

*Credit: This architecture idea came from [nickyleach](https://github.com/nickyleach).*

## The Problem

Most weather providers charge per request, and thousands of [Hello Weather](https://helloweather.com) users ask about the same places at the same times. Calling the providers directly meant paying for every one of those requests, and cost was not the only problem. Each provider has its own rate limits, so a busy hour risked tripping them. Each response took as long as the upstream took, so latency varied from request to request. When a provider went down, the app felt it immediately.

We needed a caching layer that could absorb that load without complex infrastructure.

## The Solution

Put CloudFront between the app and each provider as an "infinite cache." Every data source gets its own distribution. The app calls CloudFront, and CloudFront calls the origin only on a miss.

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

Each source's distribution and origin live in one config file, and a `CDN_ENABLED` flag picks which host the adapter calls (hosts are illustrative):

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

Development leaves `CDN_ENABLED` unset and talks to origins directly. A problem at one origin stays inside one distribution and never touches the others.

## Implementation

### The Cache Key Problem

A CDN keys its cache on the URL plus the headers you tell it to vary on. The providers' own `Cache-Control` headers are no help: many sources send none, and the rest set TTLs too short to be worth caching. So we send our own expiry header and put it in the cache key. The obvious first version is wrong:

```ruby
# Different every request = always cache miss
headers["Cache-Expires"] = 1.hour.from_now.to_s
```

Each request computes its own expiration from the moment it is made, so no two requests share a key and nothing is ever a hit.

### The Cache-Expires Solution

Instead of "how long to cache," send "when this expires," as a fixed boundary time:

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

Every time-based branch rounds down to a boundary and then adds one second past it, so a request made exactly on the boundary keys to the next window rather than the one that just closed. `:weekly` sends a version string instead of a time; CloudFront's TTL on every distribution is one week, so `v1` rides that TTL and only a bumped version evicts it. `:astronomy` and `:pollen` landed in June 2026, after this post was first published, for data that changes once per calendar date. Requests made at different times inside the same window now produce the same header:

```ruby
# Request at 01:05 UTC → Cache-Expires: 02:00 UTC (miss)
# Request at 01:59 UTC → Cache-Expires: 02:00 UTC (hit!)
# Request at 02:15 UTC → Cache-Expires: 03:00 UTC (miss)
# Request at 02:30 UTC → Cache-Expires: 03:00 UTC (hit!)
```

### Cache Levels

Different data types need different freshness:

| Level | Boundary | Max Cache Time | Use Case |
|-------|----------|----------------|----------|
| `:currently` | 15 min | ~15 min | Current conditions |
| `:minutely` | 1 min | ~1 min | Precipitation nowcast |
| `:hourly` | 1 hour | ~1 hour | Hourly forecast |
| `:daily` | 3 hours | ~3 hours | Daily forecast |
| `:alerts` | 15 min | ~15 min | Weather alerts |
| `:astronomy` / `:pollen` | next UTC date | ~24 hours | Sun/moon, pollen |
| `:weekly` | version | 1 week | URL-stable only: location keys, geocoding |

Sun/moon data ran on `:daily` until June 2026 and now has its own level. It never belonged on `:weekly`: the static `v1` key would serve the same phases all week. Anything keyed to a calendar date needs a level that rolls at the date boundary.

### 15-Minute Buckets

Current conditions bucket into 15-minute windows:

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

Source adapters name the cache level and the URL, and nothing else:

```ruby
def currently_data
  response = get(:currently, "https://#{host}/current?lat=#{lat}&lon=#{lon}")
  build_currently(response)
end
```

Under the hood, `get()` adds the cache headers and counts hits and misses:

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

The HTTP client refuses any CloudFront request that would key the cache wrong:

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

CloudFront forwards only the headers its config names and drops the rest before the origin sees them. A new header added elsewhere would either vanish silently or, once added to the config, fragment the cache key. The allowlist mirrors the config, so a header that is not in both raises at the call site. `Accept` and `Authorization` are there because a couple of origins need them on the request.

### Additional Cache Key Headers

Beyond `Cache-Expires`, we include:

| Header | Purpose |
|--------|---------|
| `Cache-Env` | Prevent staging/production mixing |
| `Cache-Id` | Per-account isolation |
| `User-Agent` | Consistent across requests |

## Debugging Cache Behavior

CloudFront reports what happened on every response:

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

Each source has its own latency profile, so this architecture creates a need for per-source timeout tuning. [CloudFront Logging: Time-Boxed Investigations](/cloudfront-logging/) covers how CloudFront logs drive those timeouts.

## Results

- Cache hit rates of 60-80% for frequently accessed locations, as of February 2026.
- Fewer origin requests, so lower upstream costs.
- CloudFront shields the app from brief provider outages, and edge locations serve cached responses faster than the origin would.

## Lessons Learned

- **One distribution per origin.** Isolation is cheaper than debugging one source's bad responses showing up in another's cache.
- **Reject unapproved cache headers at the client.** A stray header is silently dropped or silently fragments the key; an exception is a bug you find in development.
- **Count hits and misses at the call site.** Without a tracker, the savings are invisible and so are regressions.
- **Set the TTL per data type.** The freshness the UI needs decides the boundary, and one policy cannot serve a nowcast and a moon phase.
