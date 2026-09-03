---
layout: post
title: "Cheaper Than Staying in Ruby"
date: 2026-09-01 12:00:00 -0600
summary: "Nodo runs one long-lived Node process next to Ruby and makes an npm function look like a Ruby method. On our fiber server the socket hop costs a tenth of a millisecond and parks one fiber, while the pure-Ruby port cost 15 ms and stalled every fiber. The price is memory, plus a few rules that keep a child process safe on the request path."
tags: [ruby, nodo, node, falcon, async, performance]
model: "Claude Fable 5.1"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

In May 2026 we swapped the JavaScript astronomy library for a pure-Ruby one in production. Each sun-and-moon calculation went from 0.28 ms to 15.4 ms and 203,328 Ruby allocations. Two days later we rolled it back. The JavaScript library had never been running inside Ruby. It ran in a separate Node process that we reached over a Unix socket, and that round trip was cheaper than doing the math in Ruby.

[Hello Weather](https://helloweather.com) computes sunrise, sunset, civil twilight, solar noon, moon phase, illumination, moonrise, and moonset for every forecast day, because most vendors supply only the first two. [Compute the Sky Yourself](/compute-the-sky-yourself/) covers why the engine has to be a real ephemeris (a model of where the sun and moon are at a given moment) and why a cache over the Ruby port didn't help. This post is about the bridge underneath. It covers how our Rails app on Falcon calls three npm packages as if they were Ruby, why the process boundary turned out to be the cheap part, what the bridge costs in memory, and the rules that keep a child process safe on the request path.

The server model decides everything here. Under Falcon, every in-flight request in a worker is a fiber on one reactor thread. Fibers can share waiting time, but they can't share Ruby CPU. [Falcon and Ruby Async](/falcon-async-performance/) explains the model, and [Find the Hotspot, Then Cut the Dynos](/find-the-hotspot-then-cut-the-dynos/) shows how CPU per request turns into dyno count. Fifteen milliseconds of Ruby math per forecast day, on nearly every request, adds up to more dynos, not just slower responses.

## The Solution

We kept the math in JavaScript, where the best implementations live, and accepted a process boundary instead of a port. The [Nodo](https://github.com/mtgrosser/nodo) gem does the plumbing. It spawns one Node process, loads the packages once, and turns each JavaScript function we declare into a Ruby method. The parts that matter:

- A JavaScript function as a Ruby method
- One coarse call, not many fine ones
- Why the hop beats in-process on a fiber server
- What it costs, and the flags that did not help
- The rules for a child process on the request path
- One child per process, or per dyno

### A JavaScript function as a Ruby method

We tried or costed the alternatives. The pure-Ruby port is the measurement above. A remote lookup service for the timezone and country half was written up as the way to remove Node, then parked. An embedded JavaScript engine got a quick spike and was folded into the same parked plan. What shipped is a subclass of `Nodo::Core`, in April 2025 for timezone and country lookups and in July 2025 for astronomy:

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

`require Astronomy: "astronomy-engine"` becomes `const Astronomy = require("astronomy-engine")` inside the Node process. `function :data` defines a Ruby method with the same name. When we call it, Nodo turns the arguments into JSON, posts them over a Unix socket to a tiny HTTP server it runs inside Node, and parses the JSON that comes back. The second adapter has the same shape with two packages, `geo-tz` and `@rapideditor/country-coder`, and a `script` block that turns off geo-tz's internal cache. That's three packages total, and `package.json` lists nothing else.

The line to notice is `@_node ||= Node.new`. Nodo creates a JavaScript-side context for every Ruby instance and registers a finalizer to clean it up when Ruby garbage-collects the instance. If we created a `Node` per request, we'd churn contexts on both sides and depend on Ruby finalizers to clean up. So one class-level object holds the bridge, and per-request state lives in the outer adapter.

### One coarse call, not many fine ones

Every call is a fresh HTTP request over the socket, with JSON in both directions. So the cost is per call plus per byte, and the design follows from that: make few calls and keep the payloads small. The astronomy function computes nine fields in one round trip and returns a flat hash of numbers and strings, instead of nine functions the Ruby side would call one at a time.

We measured this on a laptop while writing the post, with a function that does nothing and one that echoes back an array of 5,000 small objects:

| Call | Mean | p95 |
|------|-----:|----:|
| No-op round trip | 0.088 ms | 0.150 ms |
| Echo 5,000 small objects | 1.877 ms | 2.094 ms |

So the bridge's floor is under a tenth of a millisecond. An astronomy call takes about 0.3 to 0.4 ms locally, and most of that is V8 doing real work. A careless payload costs twenty times the floor. The limit is that you have to design the coarse boundary up front. There's no batching layer, and a JavaScript function that returns a big structure has to serialize all of it on every call.

### Why the hop beats in-process on a fiber server

Most people assume a process boundary must be slower than a library call. That's right for a threaded server and wrong for this one. The Ruby port's 15 ms is CPU on the reactor thread, and while it runs, no other fiber in that worker moves. The Nodo call's 0.3 ms is a socket write, a wait, and a socket read. The question is the wait: does it block the reactor, or yield to it?

Our own notes in the repository called the lookup "in-process, blocking". Checking is cheap, so we checked instead of trusting the note. Here's a JavaScript function that busy-loops for 500 ms, called from one fiber while a sibling fiber ticks every 50 ms:

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

The ticks land on schedule while Node is busy. Nodo's client is a `Net::HTTP` subclass talking to a `UNIXSocket`. Under Async, that socket read goes through Ruby's fiber scheduler, so the calling fiber parks and the reactor serves everyone else. We didn't have to wrap anything in a thread. This is why the JavaScript engine survived and the Ruby one didn't: the JavaScript call costs one fiber's waiting time, and the Ruby call costs every fiber's CPU.

The limit is on the other side of the socket. Node is single-threaded too, so a slow JavaScript function makes every other JavaScript call in that worker wait, even though Ruby keeps moving. The bridge moves the CPU work out of the reactor, but the work still has to happen somewhere.

### What it costs, and the flags that did not help

The cost is memory. A warm Node process holding the astronomy package uses about 56 MB, grows to about 71 MB over the first few hundred calls, and doesn't shrink when Ruby garbage-collects, because it's V8's heap and not Ruby's. Loading the timezone and country packages into the same process adds about 15.6 MB. That's why we had a plan to remove it: on a dyno with a fixed memory quota, 70 MB of warm JavaScript is a real share of the budget.

The cheaper option was to tune V8 instead of removing it. We wrote a benchmark that runs each set of flags in its own Rails process and records the child's memory and per-call latency. Node's flags reach the child through one environment variable:

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

The local run, on macOS with Node 25 and 40 iterations, ranked the options:

| Flags | Node RSS | Mean | p95 |
|-------|---------:|-----:|----:|
| none | 70.6 MB | 0.55 ms | 0.93 ms |
| `--max-semi-space-size=1` | 64.2 MB | 0.61 ms | 1.10 ms |
| `--max-old-space-size=128 --max-semi-space-size=1` | 64.2 MB | 0.58 ms | 1.02 ms |
| both, plus `--jitless` | 54.6 MB | 3.05 ms | 4.33 ms |

`--jitless` saves the most memory, and we rejected it right away, because it makes a call that runs on nearly every request five times slower. The old-space cap went to production as a canary. During a traffic spike, router p95 and p99 jumped to 1,530 and 1,617 ms with 48 server errors, and we rolled it back. The capacity skill now records it as "do not retry in production without a controlled retest". The only flag that stayed in production is the semi-space cap, which trims about 6 MB and costs nothing we can measure.

The plan that recorded the benchmark also states its limit: laptop numbers on a newer Node can rank the options but can't accept one. The old-space cap looked harmless locally and failed under real traffic.

### The rules for a child process on the request path

A child process on the request path needs rules that an in-process library doesn't. We added each of these after it bit us.

**Warm it at boot.** Nodo spawns the child on first use, so the first request paid for the spawn and the package load. That could push it past the 10-second timeout the server puts on a whole request. The `warmup` class method above creates the shared `Node` object during boot, and with the platform's preboot feature the new dyno is warm before traffic reaches it. This landed in February 2026.

**Set a timeout sized for a request.** Nodo's default is 60 seconds. That's fine for a build script and a hang for a web request. The initializer sets 2 seconds.

**Let the platform's shutdown signal reach the child.** Nodo registers an `at_exit` hook that sends SIGTERM to the Node process and waits for it. That hook only runs if Ruby exits normally. Our server's preload script has one line before the app loads:

```ruby
# preload.rb
Signal.trap("TERM") { Process.kill("INT", $$) }

require_relative "config/environment"
```

The platform stops dynos with SIGTERM. Turning it into SIGINT lets Falcon shut down the way it expects to, so `at_exit` runs and the child is killed instead of left running.

**Don't rescue what has never failed.** A pull request proposed catching bridge errors and falling back to a metric with no report. We closed it. The path is local, on the same dyno, and we've never seen it fail in production, so a rescue would only hide the first real failure. The error-reporting middleware already catches a raise.

**Symlink `node_modules` with `-sfn` in a worktree.** Nodo looks for `./node_modules` in the working directory, so a fresh git worktree needs a link to the main checkout's packages. A plain `ln -s` into a directory that already has the link quietly creates a nested `node_modules/node_modules` and breaks the next npm run. The symptom is `Cannot find module 'geo-tz/now'`, and the fix is in the repo's worktree checklist.

### One child per process, or per dyno

Nodo keeps its process state at the class level. `@@node_pid`, `@@tmpdir`, and a mutex live on `Nodo::Core`, so every subclass in a Ruby process shares one Node child. That's why the second package cost 15.6 MB of data and not a second runtime. It also raises a question about forking: when the server runs several Ruby workers, is there one Node per worker or one per dyno?

Reading the code, it depends on when the child is spawned. Falcon's service loads `preload.rb` in the controlling process before it forks workers. The preload loads the Rails environment, the warmup in the initializer runs in production, and that's where the Node child is spawned. Forked workers inherit the pid, the socket path, and the class that's already defined. So the code implies one Node child per dyno, shared by every worker through the socket. We haven't verified that on a dyno, and it doesn't matter today because the worker count dropped from two to one in May 2026. The planning model behind the removal effort assumed one runtime per worker. If the shared-child reading is right, that model counted twice the memory at two workers. Check before you reason from either number.

Sharing on purpose is a different matter. A spike in May 2026 tried sharing the child across forks deliberately and found it "partially works". A forked worker can call the parent's child. The problem is who owns shutdown. Forked workers inherit Nodo's `at_exit` cleanup, so a worker that exits kills the shared process and removes the socket, and the parent is left broken. That's the open upstream request, mtgrosser/nodo issue 17, and the maintainer's reply names the same problem: the socket can be shared, but the shutdown can't. We parked the spike as documentation only, with no runtime change.

## Results

- The JavaScript engine stayed, and after two days the Ruby port went back to development and test only. A call on the serving path costs about 0.3 ms out of process versus 15.4 ms in process.
- The bridge's own overhead is 0.088 ms per call, and the socket wait yields to the fiber reactor, so a call parks one fiber instead of stalling the worker.
- The cost that stays is 56 to 71 MB of Node memory per child, trimmed by about 6 MB with the one V8 flag we kept. Removing the runtime is a parked plan, and we'd only reopen it for a replacement that doesn't cost Ruby CPU.
- We rejected two flag experiments, one locally for latency and one in production for p95 during a spike, and recorded both so nobody retries them by accident.

## Lessons Learned

- On a fiber server, an out-of-process call that yields is cheaper than CPU in the process. Measure whether the wait yields, not how long the hop takes.
- Design a bridge boundary as one coarse call that returns plain values. The price is per call and per byte, so keep both down.
- A child process on the request path needs a warmup, a timeout sized for a request, and a shutdown signal that reaches it.
- Don't trust the concurrency claim in your own notes. Ours said "blocking", it was wrong, and a 20-line script settled it.
- A laptop benchmark can rank options. Only a production canary with guardrails can accept one.
