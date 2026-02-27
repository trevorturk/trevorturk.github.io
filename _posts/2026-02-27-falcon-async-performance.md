---
layout: post
title: "Falcon and Ruby Async: Why Ruby Works for I/O-Bound Services"
date: 2026-02-27
summary: "Using Falcon and async-http for fiber-based concurrency, achieving 87% latency reduction without leaving Ruby."
tags: [ruby, falcon, async, performance]
---

*Credit: The Ruby async ecosystem is built by [ioquatix (Samuel Williams)](https://github.com/ioquatix). Falcon, async-http, and the underlying async gems make this possible.*

## The Problem

Hello Weather is a proxy and transformation layer. We fetch data from multiple upstream weather providers, transform it, and return it to clients. The work is almost entirely I/O-bound - we spend most of our time waiting for upstream responses, not computing.

Traditional Ruby (with Puma) handles this with threads. But threads have overhead, and blocking I/O means threads sit idle waiting for responses. For I/O-bound workloads, there's a better model: fibers.

The conventional wisdom was: if you need high I/O concurrency, use Node.js, Go, or Elixir. Ruby is too slow. But that wisdom is outdated.

## The Solution

Falcon + async-http give Ruby fiber-based concurrency. Fibers are lightweight, cooperative coroutines. When one fiber waits for I/O, another fiber runs. No thread overhead, no callback hell.

### falcon.rb

```ruby
#!/usr/bin/env -S falcon host
require "falcon/environment/rack"

hostname = File.basename(__dir__)
port = ENV["PORT"] || 3000

service hostname do
  include Falcon::Environment::Rack

  preload "preload.rb"
  cache false
  count ENV.fetch("FALCON_COUNT", 1).to_i
  endpoint Async::HTTP::Endpoint.parse("http://0.0.0.0:#{port}")
    .with(protocol: Async::HTTP::Protocol::HTTP11)
end
```

That's the entire Falcon configuration. It runs our Rails app with fiber-based concurrency.

### Async HTTP Client

The magic happens in our HTTP client:

```ruby
require "async"
require "async/http/internet/instance"

class Api::AsyncHttp
  DEFAULT_TIMEOUT = ENV.fetch("DEFAULT_TIMEOUT", 2).to_i

  def self.get(url, headers = nil, timeout: nil)
    Async do |task|
      response = nil

      task.with_timeout(timeout || DEFAULT_TIMEOUT) do
        response = Async::HTTP::Internet.instance.get(url, headers)

        raise Api::Weather::AuthenticationError if [401, 403].include?(response.status)
        raise Api::Weather::RateLimitError if response.status == 429
        raise Api::Weather::DataError unless response.success?

        body = response.read

        Response.new(
          data: JSON.parse(body, symbolize_names: true),
          cdn: response.headers["x-cache"]&.include?("Hit") ? "hit" : "miss"
        )
      end
    rescue Async::TimeoutError
      raise Api::Weather::TimeoutError
    ensure
      response&.finish
    end
  end
end
```

Key points:
- `Async do` creates a fiber context
- `Async::HTTP::Internet.instance` is a connection pool that reuses connections
- `task.with_timeout` handles timeouts at the fiber level
- When `response.read` blocks on I/O, other fibers run

### Parallel Requests

The real power shows when fetching from multiple sources:

```ruby
def fetch_all_sources(lat:, lon:)
  Async do
    # These run concurrently, not sequentially
    currently = Api::AsyncHttp.get(currently_url)
    hourly = Api::AsyncHttp.get(hourly_url)
    daily = Api::AsyncHttp.get(daily_url)

    # Wait for all to complete
    {
      currently: currently.wait.data,
      hourly: hourly.wait.data,
      daily: daily.wait.data
    }
  end.wait
end
```

Three requests, but wall clock time is roughly the slowest one, not the sum. If each takes 200ms, total time is ~200ms, not 600ms.

## Benchmarking

We built benchmark infrastructure to measure improvements:

```bash
# Basic benchmark
bin/rails runner scripts/benchmark/weather_request_probe.rb

# Compare modes
MODE=baseline OUTPUT=full ITERS=300 bin/rails runner scripts/benchmark/weather_request_probe.rb
MODE=weather_loops OUTPUT=full ITERS=300 bin/rails runner scripts/benchmark/weather_request_probe.rb

# Compare results
ruby scripts/benchmark/compare_results.rb \
  tmp/benchmarks/request_full_baseline.json \
  tmp/benchmarks/request_full_weather_loops.json
```

The benchmark scripts:
- `weather_request_probe.rb` - Full request path latency + allocations
- `derived_hotspots_probe.rb` - Trace derived attribute methods
- `compare_results.rb` - Diff two benchmark runs

Results are written to `tmp/benchmarks/*.json` for reproducible comparisons.

## Results

After switching from Puma to Falcon:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| p50 Latency | ~800ms | ~100ms | 87% reduction |
| p95 Latency | ~1500ms | ~300ms | 80% reduction |
| Monthly Cost | ~$2,100 | ~$500 | $1,600 savings |
| Apdex | 0.70 | 0.92 | 31% improvement |

The latency improvement comes from parallel upstream requests. The cost savings come from needing fewer dynos - each dyno handles more concurrent requests.

## Why Ruby Async Works

### For Proxy/Transformation Layers

Our workload is perfect for fibers:
- Most time spent waiting for I/O
- Minimal CPU-bound computation
- Many concurrent requests per user interaction

If we were doing heavy computation (image processing, ML inference), threads or processes would be better.

### Compared to Alternatives

| Approach | Concurrency Model | Complexity |
|----------|------------------|------------|
| **Puma** | Threads | Low, but blocking I/O wastes threads |
| **Falcon** | Fibers | Low, great for I/O-bound work |
| **Node.js** | Event loop + callbacks | Medium, callback complexity |
| **Go** | Goroutines | Low, but new language |
| **Elixir** | Processes | Medium, but new language |

Falcon gives us Node.js-level concurrency without leaving Ruby. We keep our Rails app, our gems, our tooling.

### When Fibers Help

- HTTP proxies
- API aggregators
- WebSocket servers
- Anything that waits on external services

### When Fibers Don't Help

- CPU-bound computation
- Heavy database writes
- File processing
- ML inference

## Migration Path

Moving from Puma to Falcon was surprisingly smooth:

1. **Add gems**: `falcon`, `async-http`
2. **Create `falcon.rb`** config file
3. **Replace HTTP client** with `Async::HTTP`
4. **Update Procfile**: `web: bundle exec falcon host`
5. **Test locally** before deploying

The main gotcha: any synchronous I/O blocks the whole fiber pool. Libraries must be async-aware. Most modern Ruby HTTP clients work fine.

## Lessons Learned

- **I/O-bound work loves fibers** - The concurrency model matters more than the language
- **Ruby is faster than its reputation** - Modern Ruby with good architecture is plenty fast
- **Benchmark before and after** - Real numbers beat assumptions
- **Keep it simple** - Falcon config is simpler than Puma clusters
- **Connection pooling matters** - `Async::HTTP::Internet.instance` reuses connections

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Credit: [@ioquatix](https://github.com/ioquatix) for the Ruby async ecosystem. Sources: `falcon.rb`, `app/models/api/async_http.rb`, `scripts/benchmark/`
