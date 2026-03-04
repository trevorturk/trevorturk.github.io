---
layout: post
title: "CloudFront as an Infinite Cache"
date: 2026-02-27 14:00:00 -0600
summary: "Using CloudFront distributions as a caching layer between your app and upstream data providers, with careful cache key design."
tags: [cloudfront, aws, caching, architecture]
---

*Credit: This architecture idea came from [nickyleach](https://github.com/nickyleach).*

## The Problem

Hello Weather aggregates data from multiple upstream weather providers. Each provider has rate limits, latency variations, and occasional outages. Calling them directly from our app means:

- **Rate limit risk** - Thousands of users hitting the same endpoints
- **Latency variance** - Each request depends on upstream response time
- **Cascading failures** - If a provider goes down, our app feels it immediately
- **Cost** - Most providers charge per request

We needed a caching layer that could handle our scale without complex infrastructure.

## The Solution

CloudFront as an "infinite cache" sitting between our app and each upstream provider. Each data source gets its own CloudFront distribution. Our app hits CloudFront, CloudFront hits the origin only when needed.

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

**Each source has its own distribution:**

```bash
# Environment variables
ACCUWEATHER_CDN_HOST=d123abc.cloudfront.net
AERIS_CDN_HOST=d456def.cloudfront.net
PIRATEWEATHER_CDN_HOST=d789ghi.cloudfront.net
```

This isolation means issues with one source don't affect others.

## Implementation

### The Cache Key Problem

Standard CDN caching uses URL + headers as the cache key. But weather requests have a problem: we want multiple requests for "weather at lat/lon" to hit the same cache entry, even if made at different times.

Naive approach:

```ruby
# Different every request = always cache miss
headers["Cache-Control"] = "max-age=900"  # 15 minutes
```

The problem is each request generates a different expiration time, creating different cache keys.

### The Cache-Expires Solution

Instead of "how long to cache," we send "when does this expire" - a fixed boundary time:

```ruby
def cdn_headers_for(cache_level)
  cache_expires = case cache_level
  when :currently, :alerts
    Time.now.utc.beginning_of_hour + cdn_15_minutes + 1.second
  when :minutely
    Time.now.utc.beginning_of_minute + 1.minute + 1.second
  when :hourly
    Time.now.utc.beginning_of_hour + 1.hour + 1.second
  when :daily
    Time.now.utc.beginning_of_day + cdn_3_hours + 1.second
  when :weekly
    "v1"  # Static versioned cache
  end

  { "Cache-Expires" => cache_expires.to_s }
end
```

Now requests at different times produce the same header:

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
| `:weekly` | version | 1 week | Moon phases, static |

### 15-Minute Buckets

For current conditions, we bucket into 15-minute windows:

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

Source adapters use a simple interface:

```ruby
def currently_data
  response = get(:currently, "https://#{host}/current?lat=#{lat}&lon=#{lon}")
  build_currently(response)
end
```

Under the hood, `get()` adds the cache headers and tracks hits:

```ruby
def get(cache_level, url, headers = {})
  headers.merge!(cdn_headers_for(cache_level)) if cdn_enabled?

  Api::AsyncHttp.get(url, headers, timeout: timeout).wait.tap do |response|
    case response.cdn
    when "hit"  then hit_tracker&.cache_hit
    when "miss" then hit_tracker&.source_hit
    end
  end.data
end
```

### Api::AsyncHttp

The HTTP client validates CDN usage:

```ruby
class Api::AsyncHttp
  CDN_HEADERS_ALLOWED = [
    "Cache-Env", "Cache-Expires", "Cache-Id", "User-Agent"
  ]

  def self.get(url, headers, timeout:)
    validate_cdn_headers_if_cdn_host!(url, headers)
    # ... make request
  end

  def self.validate_cdn_headers_if_cdn_host!(url, headers)
    return unless url.include?("cloudfront")

    # Must have Cache-Expires
    unless headers.key?("Cache-Expires")
      raise ArgumentError, "can't hit CloudFront without cache busting header"
    end

    # Only allowed headers
    if headers.keys.any? { |h| CDN_HEADERS_ALLOWED.exclude?(h) }
      raise ArgumentError, "can't hit CloudFront with unapproved headers"
    end
  end
end
```

This prevents accidental cache pollution from new headers.

### Additional Cache Key Headers

Beyond `Cache-Expires`, we include:

| Header | Purpose |
|--------|---------|
| `Cache-Env` | Prevent staging/production mixing |
| `Cache-Id` | Per-account isolation |
| `User-Agent` | Consistent across requests |

## Debugging Cache Behavior

CloudFront tells you what happened:

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

This CDN architecture creates the need for per-source timeout tuning, which we cover in [CloudFront Logging: Time-Boxed Investigations](/2026/02/27/cloudfront-logging.html). Each source has different latency characteristics, and CloudFront logs help us tune timeouts appropriately.

## Results

- **Cache hit rates 60-80%** for frequently accessed locations
- **Upstream costs reduced** - Fewer origin requests
- **Reliability improved** - CloudFront shields us from brief outages
- **Latency reduced** - Edge locations serve cached responses faster

## Lessons Learned

- **Boundary times, not durations** - The key insight that makes caching work
- **Per-source distributions** - Isolation prevents cross-contamination
- **Validate CDN headers** - Runtime checks prevent cache pollution
- **Track cache hits** - Visibility into savings and behavior
- **Different data, different TTLs** - One cache policy doesn't fit all

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Architecture credit: [@nickyleach](https://github.com/nickyleach). Sources: `app/models/api/async_http.rb`, `.claude/skills/cdn-caching/SKILL.md`
