---
layout: post
title: "CloudFront as an Infinite Cache"
date: 2026-02-27 14:00:00 -0600
summary: "Using CloudFront distributions as a caching layer between your app and upstream data providers, with careful cache key design."
tags: [cloudfront, aws, caching, architecture]
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

Each source's distribution is configured by host:

```bash
# Environment variables
ACCUWEATHER_CDN_HOST=d123abc.cloudfront.net
AERIS_CDN_HOST=d456def.cloudfront.net
PIRATEWEATHER_CDN_HOST=d789ghi.cloudfront.net
```

A problem at one origin stays inside one distribution and never touches the others.

## Implementation

### The Cache Key Problem

A CDN keys its cache on the URL plus the headers you tell it to vary on. For weather that is not quite enough. Two requests for the same lat/lon made at different times should land on the same cache entry, and a duration-based header does not give you that:

```ruby
# Different every request = always cache miss
headers["Cache-Control"] = "max-age=900"  # 15 minutes
```

Each request computes its own expiration from the moment it is made, so no two requests share a key and nothing is ever a hit.

### The Cache-Expires Solution

Instead of "how long to cache," send "when this expires," as a fixed boundary time:

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

Every time-based branch rounds down to a boundary and then adds one second past it. `:weekly` sends a version string instead of a time. Requests made at different times inside the same window now produce the same header:

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

The HTTP client refuses any CloudFront request that would key the cache wrong:

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

A new header added elsewhere would silently change the cache key. The allowlist turns that into an exception at the call site.

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

- Cache hit rates of 60-80% for frequently accessed locations.
- Fewer origin requests, so lower upstream costs.
- CloudFront shields the app from brief provider outages, and edge locations serve cached responses faster than the origin would.

## Lessons Learned

- **One distribution per origin.** Isolation is cheaper than debugging one source's bad responses showing up in another's cache.
- **Reject unapproved cache headers at the client.** A stray header is a silent miss forever; an exception is a bug you find in development.
- **Count hits and misses at the call site.** Without a tracker, the savings are invisible and so are regressions.
- **Set the TTL per data type.** The freshness the UI needs decides the boundary, and one policy cannot serve a nowcast and a moon phase.

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Architecture credit: [@nickyleach](https://github.com/nickyleach). Sources: `app/models/api/async_http.rb`, `.claude/skills/cdn-caching/SKILL.md`

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The Problem now opens on per-request pricing and shared locations as prose instead of a bullet list, each code sample is followed by what to notice rather than a restatement, and Lessons Learned drops the bullet that repeated the Cache-Expires section in favor of four rules that transfer. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
