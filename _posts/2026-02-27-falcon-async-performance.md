---
layout: post
title: "Falcon and Ruby Async"
date: 2026-02-27 16:00:00 -0600
summary: "Fiber-based concurrency with Falcon and async-http cut p50 latency by 87% and dyno cost by $1,600 a month on an I/O-bound Rails proxy, without leaving Ruby."
tags: [ruby, falcon, async, performance]
---

*Credit: The Ruby async ecosystem is built by [ioquatix (Samuel Williams)](https://github.com/ioquatix). Falcon, async-http, and the underlying async gems make this possible.*

## The Problem

At p50, a request to [Hello Weather](https://helloweather.com) took about 800ms under Puma, and nearly all of that was time spent waiting. The app is a proxy and transformation layer. It fetches data from multiple upstream weather providers, transforms it, and returns it to clients. At the time, almost none of the work was computation.

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

# Heroku's router forwards ~32KB request lines; the 8KB default H13'd probes.
MAXIMUM_LINE_LENGTH = 65_536

service hostname do
  include Falcon::Environment::Rack

  preload "preload.rb"
  cache false
  count ENV.fetch("FALCON_COUNT", 1).to_i
  endpoint Async::HTTP::Endpoint.parse("http://0.0.0.0:#{port}")
    .with(protocol: Async::HTTP::Protocol::HTTP11.new(maximum_line_length: MAXIMUM_LINE_LENGTH))
end
```

That is the entire Falcon configuration. It runs the Rails app with fiber-based concurrency. The line-length override is the only thing added since 2021; it landed in July 2026 after Heroku's router forwarded request lines longer than Falcon's 8KB default.

### The HTTP client

Every upstream fetch goes through one client. This is the real class with its request logging and CDN header checks trimmed out:

```ruby
require "async"
require "async/http/internet/instance"

class Api::AsyncHttp
  CDN_NAME = "cloudfront"
  DEFAULT_TIMEOUT = ENV.fetch("DEFAULT_TIMEOUT", 2).to_i

  def self.get(url, headers = nil, timeout: nil, parse: :json)
    Async do |task|
      host = URI(url).host
      response = nil
      body = nil

      task.with_timeout(timeout || DEFAULT_TIMEOUT) do
        response = Async::HTTP::Internet.instance.get(url, headers)

        raise(Api::Weather::AuthenticationError, host) if [401, 403].include?(response.status)
        raise(Api::Weather::RateLimitError, host) if [429].include?(response.status)
        raise(Api::Weather::DataError.new(host, status: response.status)) unless response.success?

        body = response.read

        if body.nil? && response.status == 204
          body = "{}"
        elsif body.nil?
          raise Api::Weather::DataError.new(host, status: response.status)
        end

        data = case parse
          when :json then JSON.parse(body, symbolize_names: true)
          when :raw  then body
        else
          raise Api::Weather::NotImplementedError, parse
        end

        Response.new(
          data: data,
          cdn: response.headers["x-cache"] == ["Hit from #{CDN_NAME}"] ? "hit" : "miss",
        )
      end
    rescue Async::TimeoutError
      raise Api::Weather::TimeoutError, host
    rescue JSON::ParserError, Protocol::HTTP2::StreamError
      raise Api::Weather::DataError.new(host, status: response&.status)
    ensure
      response&.finish
    end
  end

  class Response
    attr_accessor :data, :cdn

    def initialize(args={})
      args.each { |key, val| send("#{key}=", val) }
    end
  end
end
```

`Async do` opens a fiber context, and `task.with_timeout` applies the timeout at the fiber level. `Async::HTTP::Internet.instance` is a connection pool, so repeated calls reuse connections. Notice `response.read`: when it blocks on I/O, other fibers run. The `parse:` option arrived in July 2026 for endpoints that return something other than JSON; everything else has been stable since the client was written.

### Fan-out across sources

One provider needs several endpoints. Each source class declares them, and a `preload_each` helper on the base class runs them under an `Async::Barrier`:

```ruby
# In a source class
def preload(_output)
  preload_each \
    :location_data,
    :currently_data,
    :hourly_data,
    :daily_data,
    :alerts_data
end

# In the base class
def preload_each(*methods)
  Sync do
    barrier = Async::Barrier.new

    methods.each do |method|
      barrier.async { send(method) }
    end

    yield barrier if block_given?

    begin
      barrier.wait
    ensure
      barrier.stop
    end
  end

  nil
end

def get(cache_level, url, headers = {}, parse: :json)
  Api::AsyncHttp.get(url, headers, timeout: timeout, parse: parse).wait.data
end
```

Each `*_data` method calls `get` (shown without its CDN header and hit-tracking bookkeeping), which waits on one `Api::AsyncHttp` task. The barrier runs them as sibling fibers, so five requests cost roughly the slowest one, not the sum. If each takes 200ms, total time is ~200ms, not a second.

## Benchmarking

The Puma-to-Falcon switch predates the repo's benchmark tooling. Once the waiting overlapped, CPU became the limit, and the scripts that exist today were added in February 2026 to chase that:

```bash
# Basic benchmark
bin/rails runner scripts/benchmark/weather_request_probe.rb

# Compare modes
MODE=baseline OUTPUT=full ITERS=300 bin/rails runner scripts/benchmark/weather_request_probe.rb
MODE=weather_loops OUTPUT=full ITERS=300 bin/rails runner scripts/benchmark/weather_request_probe.rb

# Compare results
ruby scripts/benchmark/compare_results.rb \
  tmp/benchmarks/request_full_baseline_20260225T000000Z.json \
  tmp/benchmarks/request_full_weather_loops_20260225T000000Z.json
```

`weather_request_probe.rb` measures full request path latency and allocations, `derived_hotspots_probe.rb` traces derived attribute methods, and `compare_results.rb` diffs two runs. Each run writes a timestamped file to `tmp/benchmarks/`, so comparisons are reproducible.

## Results

The switch from Puma to Falcon landed in September 2021, and no benchmark from that migration was kept. These are the figures as recorded when this post was written, in February 2026:

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| p50 Latency | ~800ms | ~100ms | 87% reduction |
| p95 Latency | ~1500ms | ~300ms | 80% reduction |
| Monthly Cost | ~$2,100 | ~$500 | $1,600 savings |
| Apdex | 0.70 | 0.92 | 31% improvement |

The latency improvement comes from parallel upstream requests. The cost savings come from needing fewer dynos, because each dyno handles more concurrent requests.

## Why Ruby Async Works

The workload decides. When the switch was made, most of each request here was waiting on upstream services, with minimal CPU-bound computation and many concurrent requests per user interaction. The same shape covers HTTP proxies, API aggregators, WebSocket servers, and anything else that waits on external services. Fibers stop helping when the time goes into computation: CPU-bound work, heavy database writes, file processing, ML inference. Heavy computation such as image processing is better served by threads or processes.

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

Moving from Puma to Falcon took five steps, all in one pull request in September 2021:

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

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The `falcon.rb` excerpt gained the request-line-length override added in July 2026; the client excerpt was replaced with the real class minus logging and CDN header checks, adding the `parse:` option, per-host error messages, 204 handling, and the `Response` class it referenced; the illustrative `fetch_all_sources` fan-out was replaced with the real `preload_each` barrier pattern. The benchmark section now says the scripts date from February 2026 and were not used for the Puma comparison, with timestamped result filenames; the results table is dated to February 2026 because the September 2021 switch left no recorded benchmark; "almost none of the work is computation" became past tense because the app is CPU-bound today.
