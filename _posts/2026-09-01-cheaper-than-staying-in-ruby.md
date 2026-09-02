---
layout: post
title: "Cheaper Than Staying in Ruby"
date: 2026-09-01 12:00:00 -0600
summary: "Nodo runs one long-lived Node process next to Ruby and makes an npm function look like a Ruby method. On a fiber server the socket hop costs a tenth of a millisecond and parks one fiber, while the pure-Ruby port cost 15 ms and stalled every fiber. What it costs is memory, and the rules that keep it safe."
tags: [ruby, nodo, node, falcon, async, performance]
---

## The Problem

In May 2026 the pure-Ruby astronomy library replaced the JavaScript one in production, and each sun-and-moon calculation went from 0.28 ms to 15.4 ms and 203,328 Ruby allocations. Two days later it was rolled back. The JavaScript engine had not been running in Ruby at all. It ran in a separate Node process, reached over a Unix socket, and the round trip through that socket was cheaper than doing the arithmetic in Ruby.

[Hello Weather](https://helloweather.com) computes sunrise, sunset, civil twilight, solar noon, moon phase, illumination, moonrise, and moonset for every forecast day, because most vendors supply only the first two. [Compute the Sky Yourself](/compute-the-sky-yourself/) covers why the engine has to be a real ephemeris and why a cache over the Ruby port did not help. This post is about the bridge underneath: how a Rails app on Falcon calls three npm packages as if they were Ruby, why the process boundary turned out to be the cheap part, what the bridge costs in memory, and the operating rules that make a child process safe on a request path.

The constraint that decides everything is the server model. Under Falcon, every in-flight request in a worker is a fiber on one reactor, and the one resource those fibers cannot share is Ruby CPU. [Falcon and Ruby Async](/falcon-async-performance/) explains the model, and [Find the Hotspot, Then Cut the Dynos](/find-the-hotspot-then-cut-the-dynos/) shows how directly CPU per request converts into dynos. Fifteen milliseconds of pure-Ruby math per forecast day, on the hot path of nearly every request, is not a latency problem. It is a fleet-size problem.

## The Solution

Keep the computation in JavaScript, where the best implementations live, and pay for a process boundary instead of a port. The [Nodo](https://github.com/mtgrosser/nodo) gem does the plumbing: it spawns one Node process, loads the packages once, and turns each declared JavaScript function into a Ruby method. The parts that matter:

- A JavaScript function as a Ruby method
- One coarse call, not many fine ones
- Why the hop beats in-process on a fiber server
- What it costs, and the flags that did not help
- The rules for a child process on the request path
- One child per process, or per dyno

### A JavaScript function as a Ruby method

The alternatives were tried or costed. A pure-Ruby port is the measurement above. A remote lookup service for the timezone and country half was written up as the removal path and parked. An embedded JavaScript engine was spiked and folded into the same parked plan. What shipped in April 2025, first for timezone and country lookups and then in July 2025 for astronomy, is a subclass of `Nodo::Core`:

```ruby
# app/models/api/sources/secondary/astronomy_engine.rb (trimmed)
class AstronomyEngine
  class Node < Nodo::Core
    require Astronomy: "astronomy-engine"

    function :data, <<~JS
      (lat, lon, localDayMiddleStr, localDayStartStr, localDayLengthDays) => {
        const localDayMiddle = new Astronomy.AstroTime(new Date(localDayMiddleStr));
        const localDayStart = new Astronomy.AstroTime(new Date(localDayStartStr));
        const results = {};

        results.moonPhase = Astronomy.MoonPhase(localDayMiddle);
        results.moonIllumination = Astronomy.Illumination('Moon', localDayMiddle).phase_fraction;
        if (lat == null || lon == null) return results;

        const observer = new Astronomy.Observer(lat, lon, 0);
        const sunrise = Astronomy.SearchRiseSet('Sun', observer, +1, localDayStart, localDayLengthDays);
        const sunset  = Astronomy.SearchRiseSet('Sun', observer, -1, localDayStart, localDayLengthDays);
        // ... civil twilight, moonrise, moonset, and solar noon follow the same shape
        results.sunrise = sunrise ? sunrise.date.toISOString() : null;
        results.sunset  = sunset  ? sunset.date.toISOString()  : null;
        return results;
      }
    JS
  end

  def self.node
    @_node ||= Node.new
  end

  def data
    @_data ||= self.class.node.data(
      lat, lon, time.middle_of_day.iso8601, time.beginning_of_day.iso8601, day_length_days
    ).symbolize_keys
  end
end
```

`require Astronomy: "astronomy-engine"` becomes a `const Astronomy = require("astronomy-engine")` inside the Node process. `function :data` defines a Ruby method of the same name. Calling it serializes the arguments to JSON, posts them over a Unix socket to a tiny HTTP server that Nodo runs inside Node, and parses the JSON that comes back. The second adapter is the same shape with two packages, `geo-tz` and `@rapideditor/country-coder`, and a `script` block that disables geo-tz's internal cache. Three packages total, and `package.json` lists nothing else.

The line to notice is `@_node ||= Node.new`. Nodo creates a JavaScript-side context for every Ruby instance and registers a finalizer to garbage-collect it. An adapter that instantiated its `Node` per request would churn contexts on both sides and lean on Ruby finalizers to clean up. One class-level singleton holds the bridge, and per-request state lives in the outer adapter object.

### One coarse call, not many fine ones

Every call is a fresh HTTP request over the socket, JSON in both directions. That fixes the design of the boundary: the cost is per call plus per byte, so make few calls and keep the payloads small. The astronomy function computes nine fields in one round trip and returns a flat hash of scalars, rather than exposing nine functions the Ruby side would call one by one.

Measured on a laptop today, with a no-op function and a 5,000-object array echoed back:

| Call | Mean | p95 |
|------|-----:|----:|
| No-op round trip | 0.088 ms | 0.150 ms |
| Echo 5,000 small objects | 1.877 ms | 2.094 ms |

So the bridge's floor is under a tenth of a millisecond, and of the ~0.3 to 0.4 ms an astronomy call takes locally, most is V8 doing real work. Twenty times that floor is what a careless payload costs. The limit is that the coarse boundary has to be designed up front. There is no batching layer, and a JavaScript function that returns a large structure pays the JSON tax on every call.

### Why the hop beats in-process on a fiber server

The intuition that a process boundary must be slower than a library call is right for a threaded server and wrong for this one. The Ruby port's 15 ms is CPU spent on the reactor thread, and while it runs no other fiber in that worker moves. The Nodo call's 0.3 ms is a socket write, a wait, and a socket read. The wait is the interesting part: does it block the reactor or yield to it?

The repository's own notes called the lookup "in-process, blocking". The check is cheap, so it was run rather than trusted. A JavaScript function that busy-loops for 500 ms, called from one fiber while a sibling fiber ticks every 50 ms:

```ruby
require "nodo"
require "async"

class Probe < Nodo::Core
  function :spin, "(ms) => { const end = Date.now() + ms; while (Date.now() < end) {} ; return ms }"
end

probe = Probe.new
ticks = []
t0 = Process.clock_gettime(Process::CLOCK_MONOTONIC)

Sync do |task|
  ticker = task.async do
    5.times { sleep 0.05; ticks << ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - t0) * 1000).round }
  end
  task.async { probe.spin(500) }.wait
  ticks << "nodo returned at #{((Process.clock_gettime(Process::CLOCK_MONOTONIC) - t0) * 1000).round} ms"
  ticker.wait
end

puts ticks.inspect
# => [51, 102, 153, 204, 255, "nodo returned at 500 ms"]
```

The ticks land on schedule while Node is busy. Nodo's client is a `Net::HTTP` subclass talking to a `UNIXSocket`, and under Async that socket read goes through Ruby's fiber scheduler, so the calling fiber parks and the reactor serves everyone else. Nothing had to be wrapped in a thread. That is the mechanical reason the JavaScript engine survived and the Ruby one did not: the JavaScript call costs one fiber's wall time, and the Ruby call costs every fiber's CPU.

The limit is on the other side of the socket. Node is single-threaded too, so a slow JavaScript function serializes every other JavaScript call in that worker even though Ruby keeps moving. The bridge moves the CPU out of the reactor. It does not make it free.

### What it costs, and the flags that did not help

The bill is memory. A warm Node process holding the astronomy package is about 56 MB resident, grows to about 71 MB over the first few hundred calls, and does not shrink when Ruby garbage-collects, because it is V8's heap and not Ruby's. Loading the timezone and country packages into the same process adds about 15.6 MB. That is why the removal plan existed: on a dyno with a fixed memory quota, 70 MB of warm JavaScript is a real fraction of the budget.

The cheaper path was to tune V8 rather than remove it. A benchmark runs each flag set in an isolated Rails process and records the child's resident memory and per-call latency. Node's flags reach the child through one environment variable:

```ruby
# config/initializers/nodo.rb
require "shellwords"

Nodo.args = Shellwords.split(ENV.fetch("NODO_NODE_ARGS", ""))
Nodo.timeout = 2

Rails.application.config.after_initialize do
  next unless Rails.env.production? || ENV["NODO_WARMUP"]

  AstronomyEngine.warmup
end
```

The local matrix, on macOS with Node 25 and 40 iterations, ordered the options:

| Flags | Node RSS | Mean | p95 |
|-------|---------:|-----:|----:|
| none | 70.6 MB | 0.55 ms | 0.93 ms |
| `--max-semi-space-size=1` | 64.2 MB | 0.61 ms | 1.10 ms |
| `--max-old-space-size=128 --max-semi-space-size=1` | 64.2 MB | 0.58 ms | 1.02 ms |
| both, plus `--jitless` | 54.6 MB | 3.05 ms | 4.33 ms |

`--jitless` buys the most memory and was rejected on sight: a fivefold latency increase on a call that runs on nearly every request. The old-space cap went to production as a canary and was rolled back after router p95 and p99 during a traffic spike jumped to 1,530 and 1,617 ms with 48 server errors. The capacity skill now records it as "do not retry in production without a controlled retest". The only production-accepted flag is the semi-space cap, which trims about 6 MB and costs nothing measurable.

The limit of the benchmark is stated in the plan that recorded it: laptop numbers on a newer Node are good for ordering the options and useless for accepting them. The old-space cap looked free locally and failed under real traffic.

### The rules for a child process on the request path

A child process on the hot path needs rules that an in-process library does not. Each of these was added after it bit.

**Warm it at boot.** Nodo spawns the child lazily on first use, and the first request paid the spawn plus package load, which could push it past the whole-request timeout the server enforces at 10 seconds. The `warmup` class method above instantiates the singleton during boot, and with the platform's preboot feature the new dyno is warm before traffic reaches it. Landed February 2026.

**Set the timeout for a request path.** Nodo's default is 60 seconds, which is a reasonable default for a build script and a hang for a web request. The initializer sets 2 seconds.

**Let the platform's shutdown signal reach the child.** Nodo registers an `at_exit` hook that sends SIGTERM to the Node process and waits for it. That hook only runs if Ruby exits normally. The server's preload script is one line before the app loads:

```ruby
# preload.rb
Signal.trap("TERM") { Process.kill("INT", $$) }

require_relative "config/environment"
```

The platform stops dynos with SIGTERM. Translating it to SIGINT lets Falcon unwind the way it expects to, so `at_exit` runs and the child is reaped instead of orphaned.

**Do not rescue what has never failed.** A pull request proposed rescuing bridge failures into a silent metric-only fallback. It was closed. The path is local, in-process, and had never been observed failing in production, so a rescue would only hide the first real failure. The error-reporting middleware already captures a raise.

**Symlink `node_modules` with `-sfn` in a worktree.** Nodo resolves `./node_modules` relative to the working directory, so a fresh git worktree needs a link to the main checkout's packages. A plain `ln -s` into a directory that already has one silently creates a self-referencing `node_modules/node_modules` and corrupts the next npm run. The symptom is `Cannot find module 'geo-tz/now'` and the fix is in the repo's worktree checklist.

### One child per process, or per dyno

Nodo's process state is class-level: `@@node_pid`, `@@tmpdir`, and a mutex live on `Nodo::Core`, so every subclass in a Ruby process shares one Node child. That is why the second package cost 15.6 MB of data and not a second runtime. It also raises the forking question: with the server running several Ruby workers, is there one Node per worker or one per dyno?

Reading the code, the answer depends on when the child is spawned. Falcon's service preloads `preload.rb` in the controlling process before it forks workers. The preload loads the Rails environment, the initializer's warmup runs in production, and the Node child is spawned there. Forked workers inherit the pid, the socket path, and the already-defined class, so what the code implies is one Node child per dyno, shared by every worker through the socket. That has not been verified on a dyno, and it is moot today because the worker count dropped from two to one in May 2026. The planning model that motivated the removal effort assumed one runtime per worker, and if the shared-child reading is right that model overstated the memory by half at two workers. Verify before reasoning from either number.

Sharing on purpose is a different matter. A spike in May 2026 tried pre-fork sharing explicitly and found it "partially works": a forked worker can call the parent's child, and the failure is shutdown ownership. Forked children inherit Nodo's `at_exit` cleanup, so a worker that exits kills the shared process and removes the socket, leaving the parent broken. That is the open upstream request, mtgrosser/nodo issue 17, and the maintainer's reply names the same problem: the socket can be shared, the shutdown cannot. The spike was parked as documentation only, with no runtime change.

## Results

- The JavaScript engine stayed and the Ruby port went back to development and test after two days. Per-call cost on the serving path is about 0.3 ms out of process versus 15.4 ms in process.
- The bridge's own overhead measured at 0.088 ms per call, and the socket wait yields to the fiber reactor, so a call parks one fiber instead of stalling the worker.
- The cost that stays is 56 to 71 MB of resident Node memory per child, trimmed by about 6 MB with the one accepted V8 flag. Removing the runtime is a parked plan, reopened only for a CPU-safe replacement.
- Two flag experiments were rejected, one locally on latency and one in production on spike-window p95, and both are recorded so they are not retried by accident.

## Lessons Learned

- On a fiber server, an out-of-process call that yields is cheaper than in-process CPU. Measure the wait, not the hop.
- Design a bridge boundary as one coarse call returning scalars. Per-call and per-byte costs are the whole price.
- A child process on the request path needs a warmup, a request-sized timeout, and a shutdown path that reaches it.
- Read the concurrency claim in your own notes with suspicion. "Blocking" was wrong and a 20-line script settled it.
- Laptop benchmarks order options. Only a production canary with guardrails accepts one.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "add to the list of stuff to blog about our use of nodo, which also is a performance/cpu cost win, but pretty cool tech I think as well"

**Prompt 3:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent researched Nodo across the web repository and a second wrote the post, re-running the fiber-yield and round-trip measurements before making either claim (Ruby 3.4.9, Node 26.5.0, nodo 1.8.2, async 2.44.0, on a laptop). Sources: `app/models/api/sources/secondary/astronomy_engine.rb` and `geo_tz.rb`, `config/initializers/nodo.rb`, `preload.rb`, `package.json`, `scripts/benchmark/nodo_memory_probe.rb` and its local result files, the astronomy backend, astronomy caching, and Nodo removal plans, the capacity and error-reporting skills, the nodo gem source (`core.rb`, `client.rb`), the async-service and falcon gem sources for preload ordering, and the commits for the April 2025 adoption, the February 2026 warmup, the May 2026 swap, rollback, flag configuration, and parked fork spike.

Judgment calls: the astronomy adapter excerpt is trimmed to the pattern with a comment marking the omitted fields, and the outer class name is shortened. The three npm packages are named because they are public and the point of the post; weather vendors, the timezone fallback chain, dyno formation history, and prices are not. The bridge-overhead numbers are this session's measurements rather than the researcher's earlier ones (0.098 ms and 1.35 ms), which were consistent. The "one child per dyno" reading is stated as what the code implies with an explicit unverified caveat, since checking it would mean touching production.
