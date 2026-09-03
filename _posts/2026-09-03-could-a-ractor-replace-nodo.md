---
layout: post
title: "Could a Ractor Replace Nodo?"
date: 2026-09-03 15:00:00 -0600
summary: "A reader asked whether our sun-and-moon math could run in a Ractor instead of a Node child process. We benchmarked it. The Ractor does keep the fiber reactor responsive, once you wait on it through a pipe instead of Ractor#take. But the call still costs 8 ms of CPU on some core, one Ractor serves a thirtieth of Nodo's throughput, and a pool of two crashed the Ruby VM. Nodo stays, and the pipe bridge is the part worth keeping."
tags: [ruby, ractor, nodo, falcon, async, performance, negative-result]
model: "Claude Fable 5.1"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Question

A reader of [Cheaper Than Staying in Ruby](/cheaper-than-staying-in-ruby/) asked whether the astronomy calculation could run in a Ractor instead of a Node process. Their reasoning was that a complex function with a small input and a small output is exactly what Ractors are for. If it worked, we could drop Node and the npm packages, keep the code in Ruby, and still not block the server. That's the outcome we'd been hoping for since the Ruby port failed.

Some background for readers who didn't see the earlier post. [Hello Weather](https://helloweather.com) computes sunrise, sunset, civil twilight, solar noon, moonrise, moonset, moon phase, and illumination for every forecast day, on nearly every request. In May 2026 we moved that math from a JavaScript library to a pure-Ruby one, Astronoby. Each call cost 15 ms of CPU, and we rolled it back two days later. What runs in production is the JavaScript library inside a Node child process, reached over a Unix socket through the [Nodo](https://github.com/mtgrosser/nodo) gem. A call takes about half a millisecond, and the socket wait yields to other fibers.

The reason 15 ms was fatal is the server model. Under Falcon, every request in a worker is a fiber on one reactor thread, and fibers can't share CPU. While one fiber does 15 ms of math, no other request in that worker moves. [Falcon and Ruby Async](/falcon-async-performance/) explains the model, and [Compute the Sky Yourself](/compute-the-sky-yourself/) explains why the math can't be cached away.

A Ractor is Ruby's way of running two pieces of Ruby code at the same time on different cores. Ruby normally has one lock that lets only one thread run Ruby code at a time. Each Ractor gets its own lock, so two Ractors really do run in parallel. The price is isolation. A Ractor can't see the objects in the main program unless they're frozen and marked shareable, and you talk to it by sending messages. So the reader's idea was to hand the 8 ms of math to a Ractor on another core and let the reactor thread keep serving requests.

We benchmarked it on 2026-09-03. The short answer is that the Ractor does keep the reactor responsive, and that part of the idea holds. But the math costs the same CPU on a different core. One Ractor handles about a thirtieth of Nodo's throughput, and a pool of two Ractors crashed the Ruby VM. Nodo stays. The rest of this post is what we tried, what the numbers said, and the one piece we'd reuse.

## The Experiment

The work came in three parts:

- Finding a way to wait for a Ractor without stalling the reactor
- Getting three gems to load inside a Ractor at all
- Running the same benchmark against Nodo, in-process Ruby, and the Ractor

### Waiting without stalling the reactor

The obvious call is `r.send(args)` followed by `r.take`. On Ruby 3.4, `take` blocks the calling thread, and the fiber scheduler doesn't know about it. The reactor thread stops, and every other fiber waits, which is the exact problem we were trying to avoid. We didn't want to trust that reading, so we checked it the same way the Nodo post did. A Ractor busy-loops for 500 ms while a sibling fiber ticks every 50 ms, and we look at when the ticks land.

We tried four ways to wait:

| Wait shape | Ticks during the 500 ms call | Overhead per call under Async |
|---|---|--:|
| `Ractor#take` on the reactor | none; ticks land after the call returns | 24 µs |
| `Thread.new { r.take }.join` | on schedule | ~950 µs |
| A pump thread and `Queue#pop` | on schedule, with about 5 ms of jitter | ~220 µs |
| Ractor writes to an `IO.pipe`, fiber reads it | on schedule | 12 µs |

The thread wrapper works but costs almost a millisecond per call, which is twice a whole Nodo call. The pump thread is cheaper but adds jitter. The pipe is the shape that works. The Ractor gets the write end of a pipe as a raw file descriptor, does its work, and writes the answer as a length-prefixed `Marshal` frame. The fiber reads the pipe. A pipe read goes through the fiber scheduler, the same way Nodo's socket read does, so the calling fiber parks and the reactor serves everyone else.

```ruby
require "async"

rd, wr = IO.pipe
r = Ractor.new(wr.fileno) do |fd|
  out = IO.for_fd(fd, "wb")
  out.sync = true
  loop do
    n = Ractor.receive
    t = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    nil while Process.clock_gettime(Process::CLOCK_MONOTONIC) - t < n / 1000.0
    payload = Marshal.dump({ n: n, ok: true })
    out.write([payload.bytesize].pack("N"), payload)
  end
end

call = ->(n) do
  r.send(n)
  len = rd.read(4).unpack1("N")
  Marshal.load(rd.read(len))
end

ticks = []
t0 = Process.clock_gettime(Process::CLOCK_MONOTONIC)
ms = -> { ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - t0) * 1000).round }

Sync do |task|
  ticker = task.async { 5.times { sleep 0.05; ticks << ms.call } }
  task.async { call.call(500) }.wait
  ticks << "pipe read returned at #{ms.call} ms"
  ticker.wait
end

puts ticks.inspect
# => [51, 101, 152, 203, 253, "pipe read returned at 500 ms"]
```

The line to notice is `IO.for_fd(fd, "wb")`. The Ractor can't receive the `IO` object itself, because it isn't shareable, but a file descriptor is just an integer. The `b` matters under Rails, which sets the default encoding to UTF-8, and a text-mode pipe will mangle a `Marshal` frame. The other thing to notice is what the Ractor doesn't use. It only calls `Ractor.new`, `Ractor.receive`, and `Ractor#send`. Ruby 4.0 removed `Ractor#take` and `Ractor.yield` in favor of `Ractor::Port`, and this shape survives that change.

The limit is that upgrading Ruby won't make the pipe unnecessary. A research pass found an open Ruby bug, [#22211](https://bugs.ruby-lang.org/issues/22211), reporting that `Ractor::Port#receive` stalls the fiber scheduler the same way `take` does. It's filed against the 4.1 development branch and queued behind the per-Ractor garbage collection proposal, [#22227](https://bugs.ruby-lang.org/issues/22227), which hasn't merged. So on every Ruby you can run today, a fiber that wants a Ractor's answer has to wait on something the scheduler understands. A pipe is the cheapest thing we found.

### Getting the gems to load in a Ractor

The first attempt to load the ephemeris inside a Ractor raised `Ractor::IsolationError`. So did the second and the third, each on a different thing. A Ractor can only read main-program objects that are frozen all the way down, and three gems keep state that isn't. The `ephem` gem has a frozen hash of unfrozen hashes as a constant. The `iers` gem memoizes a leap-second table behind a `Mutex`. Astronoby itself sets configuration and a cache lazily into class-level instance variables, so they don't exist until the first call and aren't frozen after it.

The shim we wrote for the benchmark does three things before any Ractor starts. It runs one full calculation on the main side, so every lazy value exists. It walks the three gems' modules and calls `Ractor.make_shareable` on every constant and module-level instance variable, skipping the mutexes. And it replaces the two mutex-guarded methods in `iers` with methods that return frozen constants:

```ruby
module RactorShim
  def self.prepare!
    Astronoby.configuration
    Astronoby.cache
    IERS.configuration
    IERS::Data.leap_second_table
    IERS::Data.finals_entries

    table = IERS::Data.instance_variable_get(:@leap_second_table)
    finals = IERS::Data.instance_variable_get(:@finals)
    IERS::Data.const_set(:RACTOR_LEAP_SECOND_TABLE, Ractor.make_shareable(table))
    IERS::Data.const_set(:RACTOR_FINALS, Ractor.make_shareable(finals))
    IERS::Data.module_eval <<~RUBY
      def leap_second_table = RACTOR_LEAP_SECOND_TABLE
      def finals_entries = RACTOR_FINALS
      module_function :leap_second_table, :finals_entries
    RUBY

    [IERS, Ephem, Astronoby].each { |root| share_tree(root) }
  end

  def self.share_tree(mod, seen = {})
    return if seen[mod]
    seen[mod] = true
    mod.instance_variables.each do |iv|
      val = mod.instance_variable_get(iv)
      next if val.is_a?(Mutex) || val.is_a?(Monitor)
      Ractor.make_shareable(val)
    end
    mod.constants(false).each do |name|
      val = mod.const_get(name)
      val.is_a?(Module) ? share_tree(val, seen) : Ractor.make_shareable(val)
    end
  end
end
```

This is a monkey patch on three gems, and it's fine for a benchmark. It isn't something we'd ship. The right fix is a pull request to each gem that freezes its constants and stops memoizing into module state. That's real work in code we don't own. The research pass also checked whether the rules loosen in newer Rubies. They don't. The rules on class-level instance variables and unshareable constants are the same through the 4.1 development branch, and Ractor is still labeled experimental.

### The benchmark

Each backend ran in its own Rails process and computed the same nine fields the production adapter does. The run covered 12 cities on one date, 40 iterations after 5 warmup rounds. Each backend ran two phases. The first is sequential calls from one fiber. The second is 8 fibers calling at once. During both, a ticker fiber sleeps for 5 ms in a loop and records how late the reactor wakes it. That lateness is the number that tells you whether the call stalls every other request. We also recorded two CPU numbers per call. CPU on the reactor thread is the number that drives our dyno count. CPU across the whole process includes work on other cores.

The Ractor backend is the shape that would have shipped. One Ractor loads the ephemeris once, then loops on `Ractor.receive`, does the full calculation, and writes the result to the pipe. The main side sends four plain values and reads one frame back:

```ruby
class RactorWorker
  def initialize(ephem_path)
    @reader, writer = IO.pipe
    @reader.binmode
    @ractor = Ractor.new(ephem_path, writer.fileno) do |path, fd|
      out = IO.for_fd(fd, "wb")
      out.sync = true
      ephem = Astronoby::Ephem.load(path)
      out.write("READY")
      loop do
        lat, lon, epoch, offset = Ractor.receive
        time = Time.at(epoch, in: offset)
        observer = Astronoby::Observer.new(
          latitude: Astronoby::Angle.from_degrees(lat),
          longitude: Astronoby::Angle.from_degrees(lon)
        )
        sun = Astronoby::RiseTransitSetCalculator.new(body: Astronoby::Sun, observer: observer, ephem: ephem)
          .event_on(time, utc_offset: time.utc_offset)
        # ... twilight, moonrise, moonset, phase, and illumination follow the same shape
        payload = Marshal.dump([sun.rising_time&.to_f, sun.setting_time&.to_f, sun.transit_time&.to_f])
        out.write([payload.bytesize].pack("N"), payload)
      end
    end
    raise "ractor failed to start" unless @reader.read(5) == "READY"
  end

  def call(lat, lon, time)
    @ractor.send([lat, lon, time.to_i, time.formatted_offset])
    len = @reader.read(4).unpack1("N")
    Marshal.load(@reader.read(len)).map { |epoch| epoch && Time.at(epoch).utc }
  end
end
```

The arguments cross the boundary as an integer epoch and an offset string, not a `Time` with a zone attached. A zoned `Time` drags an unshareable timezone object with it. The results come back as floats for the same reason. For a pool, the benchmark wraps several workers in an `Async::Queue` and each fiber checks one out, calls it, and puts it back.

## Results

The three backends, on a laptop with Ruby 3.4.10, Node 26.5.0, astronoby 0.10.0, nodo 1.8.2, and async 2.45.1:

| Backend | Call mean | Call p95 | Reactor CPU / call | Process CPU / call | Ticker lateness p95 (8 fibers) | 8-fiber throughput | Ruby RSS delta | Node child RSS |
|---|--:|--:|--:|--:|--:|--:|--:|--:|
| Nodo (production) | 0.48 ms | 0.88 ms | 0.195 ms | 0.195 ms | 0.70 ms | 3,508 calls/s | +14 MB | 114 MB |
| Astronoby in-process | 8.37 ms | 9.22 ms | 8.364 ms | 8.364 ms | 4,010 ms | 121 calls/s | +29 MB | none |
| Astronoby in one Ractor | 7.86 ms | 8.62 ms | 0.064 ms | 7.874 ms | 0.65 ms | 127 calls/s | +5 MB | none |

The in-process row is the May rollback, reproduced. The ticker fiber never ran during the whole 4-second phase, because the reactor thread was doing math the entire time. The Ractor row is the reader's point, and it holds. The ticker wakes up within a millisecond. The reactor thread spends less CPU per call than it does on a Nodo call, because all it does is write a few bytes and read a few back.

The two columns that decide it are wall time and process CPU. A call still takes 8 ms from send to answer, because the math didn't get faster. It moved to another core. At about ten forecast days per request, the Ractor path adds around 80 ms of latency to every request, against about 5 ms for Nodo. One Ractor handles 127 calls per second, which caps a dyno near 12 requests per second, against Nodo's 3,500 calls per second. Process CPU per call is the same 8 ms as in-process, and on a shared-CPU dyno, the other core isn't guaranteed to be free. So the dyno-count math from [Find the Hotspot, Then Cut the Dynos](/find-the-hotspot-then-cut-the-dynos/) comes out the same as it did for the in-process port.

Memory is the one column the Ractor wins. Its own copy of the ephemeris and heap growth came to about 5 MB, against a 114 MB Node child. That Node number is higher than the 71 MB in the earlier post. This run made about 4,300 calls at 8-way concurrency on a newer Node, and we didn't chase the difference.

Then the crashes. A single Ractor ran both phases, 8,640 calls, cleanly, twice. A pool of two and a pool of four each died on the first run, partway through the concurrent phase. The error was `malloc: possible integer overflow`, raised from an ordinary keyword-argument `super` call inside Astronoby that reads a frozen module value. The pool of two reproduced it on a rerun with a different garbage size in the message. Once a Ractor dies, the main side hangs on its pipe until the benchmark's watchdog aborts the process. A separate run of the pool of two finished both phases and then hit a `[BUG] Bus Error` inside `GC.compact` with three Ractors alive. We can't say which of Ruby, Astronoby, or our shim is at fault. We can say that a pool doesn't work on this Ruby, and a single Ractor caps throughput at 127 calls per second.

Two side findings had nothing to do with Ractors. First, Astronoby and the JavaScript library agree closely. Over 12 cities, the biggest gaps were 2.6 seconds on sunset, 24.5 seconds on morning civil twilight, and 141 seconds on moonset. The two also disagreed about whether Reykjavik has a moonset that day. Second, our development-only Astronoby adapter calls `event_on` without passing the local UTC offset, so it searches the UTC day and drops the evening sunset for US cities. The benchmark passes the offset. The adapter isn't in production, so the stakes are low, but it's now on the list.

The verdict, recorded in the web repository's tracking issue for removing Node: Ractor-hosted Astronoby is not a Nodo replacement today. We'd reopen it if Ruby ships fiber-scheduler support for Ractor receive and per-Ractor garbage collection, and a pool of four runs the concurrent phase without a crash. Or if Astronoby's per-call cost drops under a millisecond, which would change the whole question.

## Lessons Learned

- A Ractor moves CPU to another core. It doesn't remove it. Measure process CPU as well as reactor CPU, because the dyno bill is the sum.
- Whatever a fiber waits on, check that the wait goes through the scheduler. A 20-line probe with a ticker fiber answers it in a second.
- A pipe is the cheapest bridge between a Ractor and a fiber, and it's the only shape that survives Ruby 4.0's API change.
- A gem that memoizes into module state isn't Ractor-ready, and a shim in your app is a benchmark tool, not a fix.
- Test the pool, not the single worker. One Ractor ran clean for thousands of calls, and two crashed the VM.
- Write down the negative result with the conditions for reopening it, so nobody re-runs the experiment without something having changed.
