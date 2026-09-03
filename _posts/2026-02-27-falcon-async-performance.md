---
layout: post
title: "Falcon and Ruby Async"
date: 2026-02-27 16:00:00 -0600
summary: "How we run a Rails proxy that mostly waits on upstream servers on Falcon and async-http, without leaving Ruby: the server config, the HTTP client, the barrier fan-out, and where the dyno savings came from."
tags: [ruby, falcon, async, performance]
model: "Claude Opus 4.5"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

*Credit: The Ruby async ecosystem is built by [ioquatix (Samuel Williams)](https://github.com/ioquatix). Falcon, async-http, and the underlying async gems make this possible.*

## The Problem

Under Puma, nearly all of a request's time at [Hello Weather](https://helloweather.com) went to waiting. The app is a proxy. It fetches data from several upstream weather providers, reshapes it, and returns it to the apps. At the time, almost none of that work was computation.

Puma runs requests on threads. A thread that's waiting on an upstream response does nothing until it arrives, but it still has a cost of its own. When the work is nearly all waiting, we were paying for threads that mostly sat idle.

The usual advice is to move a service like this to Node.js, Go, or Elixir because Ruby is too slow. That advice is out of date.

## The Solution

We run the Rails app on Falcon and make upstream requests with async-http. Both are built on fibers. A fiber is a lightweight unit of work that hands off control when it waits, so while one fiber is waiting on the network, another runs. There's no thread cost and no callbacks to write.

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

That file is all of our Falcon config. The line-length override is the only thing we've added since 2021. It landed in July 2026, after Heroku's router forwarded request lines longer than Falcon's 8KB default.

### The HTTP client

Every upstream fetch goes through one class. Here it is, with the request logging and CDN header checks trimmed out:

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

`Async do` runs the block in a fiber, and `task.with_timeout` puts the timeout on that fiber. `Async::HTTP::Internet.instance` is a shared connection pool, so repeated calls reuse connections. The line to notice is `response.read`. While it waits on the network, other fibers run. The `parse:` option arrived in July 2026 for endpoints that return something other than JSON. Everything else has been the same since we wrote the client.

### Fan-out across sources

A single provider can need several endpoints. Each source class lists them, and a `preload_each` helper on the base class runs them together under an `Async::Barrier`, which starts a set of tasks and waits for all of them:

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

Each `*_data` method calls `get`, shown here without its CDN header and hit-tracking bookkeeping. `get` waits on one `Api::AsyncHttp` task. The barrier runs the five as sibling fibers, so the whole batch takes about as long as the slowest request, not the sum. If each takes 200ms, the batch takes ~200ms, not a second.

## Benchmarking

We have no benchmarks from the Puma-to-Falcon switch, because the benchmark scripts came later. Once the waiting overlapped, CPU became the limit, and we added the scripts in February 2026 to chase that:

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

`weather_request_probe.rb` measures latency and allocations for the full request path. `derived_hotspots_probe.rb` traces the derived attribute methods. `compare_results.rb` diffs two runs. Each run writes a timestamped file to `tmp/benchmarks/`, so we can compare runs later. The pass that produced those scripts, and the dyno cut that followed, is in [Find the Hotspot, Then Cut the Dynos](/find-the-hotspot-then-cut-the-dynos/).

## Results

We switched from Puma to Falcon in September 2021 and didn't record a benchmark, a latency figure, or a cost figure. The one dyno cut the repository does record came later, and Falcon made it possible rather than causing it. In February 2026 a CPU pass on the serialization path took the web fleet from 40 Standard-2X dynos to 8 in one evening, then to 6. That work is in [Find the Hotspot, Then Cut the Dynos](/find-the-hotspot-then-cut-the-dynos/). The plans put the cost this way:

| Web dynos | Before the CPU pass (January 2026) | After (February 25, 2026) | Change |
|-----------|-----------------------------------|---------------------------|--------|
| Count | 40 | 8 | 32 fewer |
| Approximate monthly cost | ~$2,100 | ~$500 | ~$1,600 savings |

Falcon's part in that was to make CPU the only limit. With the upstream waiting overlapped, CPU was the only thing a dyno ran out of, so cutting CPU per request meant fewer dynos. An earlier version of this post credited the cost row to the Falcon switch and carried p50, p95, and Apdex before-and-after figures. None of those has a source in the repository, so we removed them.

## Why Ruby Async Works

It depends on the workload. When we switched, most of each request was waiting on upstream services, with little computation, and one user action could fan out into many requests at once. HTTP proxies, API aggregators, and WebSocket servers have the same shape. Fibers stop helping when the time goes into computation: CPU-bound work, heavy database writes, file processing, ML inference. That kind of work is better served by threads or processes.

### Compared to alternatives

| Approach | Concurrency Model | Complexity |
|----------|------------------|------------|
| **Puma** | Threads | Low, but blocking I/O wastes threads |
| **Falcon** | Fibers | Low, great for I/O-bound work |
| **Node.js** | Event loop + callbacks | Medium, callback complexity |
| **Go** | Goroutines | Low, but new language |
| **Elixir** | Processes | Medium, but new language |

Falcon gives us Node.js-level concurrency without leaving Ruby. The Rails app, the gems, and the tooling all stay.

## Migration Path

Moving from Puma to Falcon took five steps, all in one pull request in September 2021:

1. **Add gems**: `falcon`, `async-http`
2. **Create `falcon.rb`** config file
3. **Replace HTTP client** with `Async::HTTP`
4. **Update Procfile**: `web: bundle exec falcon host`
5. **Test locally** before deploying

The one catch is that any synchronous I/O blocks the whole fiber pool, so every library in the request path has to be async-aware. Most modern Ruby HTTP clients are.

## Lessons Learned

- **Fibers only help when the time goes to waiting.** If a request is mostly computation, threads or processes win.
- **Pick the concurrency model before you pick a language.** Fibers in Ruby gave us what a rewrite in Node.js, Go, or Elixir would have, and we kept the app, the gems, and the tooling.
- **One blocking call stalls every fiber.** Check every library in the request path for async support before you switch servers.
