---
layout: post
title: "Falcon and Ruby Async"
date: 2026-02-27 16:00:00 -0600
summary: "Fiber-based concurrency with Falcon and async-http cut p50 latency by 87% and dyno cost by $1,600 a month on an I/O-bound Rails proxy, without leaving Ruby."
tags: [ruby, falcon, async, performance]
---

*Credit: The Ruby async ecosystem is built by [ioquatix (Samuel Williams)](https://github.com/ioquatix). Falcon, async-http, and the underlying async gems make this possible.*

## The Problem

At p50, a request to [Hello Weather](https://helloweather.com) took about 800ms under Puma, and nearly all of that was time spent waiting. The app is a proxy and transformation layer. It fetches data from multiple upstream weather providers, transforms it, and returns it to clients. Almost none of the work is computation.

Puma handles concurrency with threads. A thread blocked on an upstream response does nothing until the response arrives, and threads have overhead of their own. For a workload that is nearly all waiting, most of that overhead buys idling.

The conventional answer is to move I/O-heavy services to Node.js, Go, or Elixir, on the grounds that Ruby is too slow. That advice is out of date.

## The Solution

Run the Rails app on Falcon and make upstream requests with async-http. Fibers are lightweight cooperative coroutines. When one fiber waits on I/O, another runs. There is no thread overhead and no callback style to adopt.

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

That is the entire Falcon configuration. It runs the Rails app with fiber-based concurrency.

### The HTTP client

Every upstream fetch goes through one client:

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

`Async do` opens a fiber context, and `task.with_timeout` applies the timeout at the fiber level. `Async::HTTP::Internet.instance` is a connection pool, so repeated calls reuse connections. Notice `response.read`: when it blocks on I/O, other fibers run.

### Fan-out across sources

One request needs several endpoints. Issuing them inside one `Async` block makes them concurrent:

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

We built benchmark scripts to measure improvements:

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

`weather_request_probe.rb` measures full request path latency and allocations, `derived_hotspots_probe.rb` traces derived attribute methods, and `compare_results.rb` diffs two runs. Each run writes to `tmp/benchmarks/*.json`, so comparisons are reproducible.

## Results

After switching from Puma to Falcon:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| p50 Latency | ~800ms | ~100ms | 87% reduction |
| p95 Latency | ~1500ms | ~300ms | 80% reduction |
| Monthly Cost | ~$2,100 | ~$500 | $1,600 savings |
| Apdex | 0.70 | 0.92 | 31% improvement |

The latency improvement comes from parallel upstream requests. The cost savings come from needing fewer dynos, because each dyno handles more concurrent requests.

## Why Ruby Async Works

The workload decides. Most of each request here is waiting on upstream services, with minimal CPU-bound computation and many concurrent requests per user interaction. The same shape covers HTTP proxies, API aggregators, WebSocket servers, and anything else that waits on external services. Fibers stop helping when the time goes into computation: CPU-bound work, heavy database writes, file processing, ML inference. Heavy computation such as image processing is better served by threads or processes.

### Compared to alternatives

| Approach | Concurrency Model | Complexity |
|----------|------------------|------------|
| **Puma** | Threads | Low, but blocking I/O wastes threads |
| **Falcon** | Fibers | Low, great for I/O-bound work |
| **Node.js** | Event loop + callbacks | Medium, callback complexity |
| **Go** | Goroutines | Low, but new language |
| **Elixir** | Processes | Medium, but new language |

Falcon gives Node.js-level concurrency without leaving Ruby. The Rails app, the gems, and the tooling all stay.

## Migration Path

Moving from Puma to Falcon took five steps:

1. **Add gems**: `falcon`, `async-http`
2. **Create `falcon.rb`** config file
3. **Replace HTTP client** with `Async::HTTP`
4. **Update Procfile**: `web: bundle exec falcon host`
5. **Test locally** before deploying

The one gotcha: any synchronous I/O blocks the whole fiber pool, so every library in the request path must be async-aware. Most modern Ruby HTTP clients are.

## Lessons Learned

- **Fibers only pay off when the wait is I/O.** If a request is mostly computation, threads or processes win.
- **Choose the concurrency model before the language.** Fibers in Ruby gave what a rewrite in Node.js, Go, or Elixir would have, with the app, gems, and tooling intact.
- **One blocking call stalls every fiber.** Audit every library in the request path for async support before switching servers.

---

## How This Post Was Made

**Prompt:** "Write 7+ in-depth blog posts documenting real engineering patterns from helloweather/web. These posts go deeper than the existing 'Skills and Scripts' overview, showing specific implementations."

Generated by Claude (Opus 4.5) using the blog-post-generator skill. Credit: [@ioquatix](https://github.com/ioquatix) for the Ruby async ecosystem. Sources: `falcon.rb`, `app/models/api/async_http.rb`, `scripts/benchmark/`

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the 800ms p50 and the waiting behind it, the "key points" list after the client code became one paragraph, the two "when fibers help" lists folded into a single boundary paragraph, and the generic lessons were replaced by three rules that transfer. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
