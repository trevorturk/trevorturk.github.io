---
layout: post
title: "Four Shapes for Async Work"
date: 2026-09-01 14:00:00 -0600
summary: "Beyond the fan-out barrier: a semaphore as a fiber-safe memoizer, a barrier of one task as a cancellation scope, a detached fiber for side effects and GraphQL fields, and a plain thread when the work should not live in any request's task tree."
tags: [ruby, falcon, async, performance, patterns]
model: "Claude Fable 5.1"
last_edited: 2026-09-01
last_edited_by: "Claude Fable 5.1"
---

Five fibers ask the same source object for its location record, and four of them start a second HTTP request because the first one has not returned yet. That is what `@location ||= fetch(...)` does under a fiber reactor. The check runs, the fetch suspends on the socket, the reactor resumes a sibling fiber, and the sibling runs the same check against the same empty instance variable. Nothing is wrong with `||=`. It simply assumes that nothing else runs between the test and the assignment, and under a reactor that assumption fails exactly when the assignment is slow.

[Hello Weather](https://helloweather.com) serves its API from Falcon, so every in-flight request in a worker is a fiber on one reactor, and every upstream fetch is a suspension point. [Falcon and Ruby Async](/falcon-async-performance/) covers the main shape that follows from that: a barrier that fans a request out across a vendor's endpoints and waits for all of them. This post is about the four smaller shapes the codebase settled on for the questions the fan-out does not answer. How do you memoize when the callers are concurrent fibers? How do you cancel work that is not a fan-out? Where does a fire-and-forget side effect run? And when should a piece of work leave the reactor entirely? Each section gives the alternative that was tried or considered, the mechanism, the code as it runs today, and the limit the code cannot enforce.

## A Semaphore as a Memoizer

The alternative is the one the opening describes, and the app ran it until September 2021. Each source adapter memoized its endpoint responses with `||=`, and the fan-out barrier made the race routine. The vendor whose five endpoints all depend on a location lookup is the worst case: the current-conditions, hourly, daily, and alerts fibers each call `location_key`, which calls `location_data`, which finds the variable unset because the first fetch is still waiting on the socket. A contributor's commit from that month states the problem in one sentence: once `wait` is called, the reactor yields to the next fiber, and that fiber may end up calling into the same code.

The fix wraps the memoized section in an `Async::Semaphore` with the default limit of one, keyed by the name of the calling method. The base class for every adapter carries it:

```ruby
class WeatherSource
  protected

  def fetch_data(name = nil)
    name ||= caller_locations[0].label

    @_fetched_data ||= Hash.new

    with_semaphore(name) do
      unless @_fetched_data.key?(name)
        @_fetched_data[name] = yield
      end
    end

    @_fetched_data[name]
  end

  private

  def semaphore(name)
    @_semaphores ||= Hash.new { |h, k| h[k] = Async::Semaphore.new }
    @_semaphores[name]
  end

  def with_semaphore(name, &block)
    Sync do
      semaphore(name).async(&block).wait
    end
  end
end
```

An adapter's endpoint methods then read as plain memoized fetches, and the concurrency is invisible at the call site:

```ruby
def location_data
  fetch_data do
    get(:weekly, "#{host}/locations/search?q=#{lat},#{lon}")
  end
end

def hourly_data
  fetch_data do
    get(:hourly, "#{host}/forecasts/hourly/#{location_key}")
  end
end
```

The line that does the work is `semaphore(name).async(&block).wait`. In the async gem, `Semaphore#async` first waits until the count is below the limit, then spawns a child task that increments the count and releases it in an `ensure`. A second fiber arriving while the first holds the semaphore is stacked on the semaphore's waiting list and transferred back to the scheduler until the release resumes it. By the time it enters the block, the hash has the key, and the `unless` skips the fetch. The `Sync` wrapper, added a few days after the semaphore, yields the current task when there is one and starts a reactor when there is not, which is what lets the same code run under Falcon and in a test with no reactor.

Two things bound this. The semaphores live on the source instance, so the memo deduplicates fetches within one request's fan-out and nothing more; deduplication across requests belongs to the CDN, as [CloudFront as an Infinite Cache](/cloudfront-cdn-architecture/) describes. And the key is the caller's method label, so two endpoint methods that share a name in one class would share a lock and a memo slot. The base class has never had two, and nothing checks.

## A Barrier of One Task

The fan-out barrier holds several tasks and waits for all of them. The controller uses a barrier that holds exactly one, and it is there for a different reason: structured cancellation. When a request fails partway through its fan-out, whatever fibers the failure left behind must stop before the fallback starts, or the fallback shares the worker with a set of orphaned fetches that will finish and be thrown away.

The alternative was the plain rescue the controller used before March 2025, which caught the exception and moved on with the children still running. The current shape wraps the action in a barrier, waits, and stops the barrier in an `ensure` on every exit:

```ruby
class ForecastController < ApplicationController
  around_action :fallback

  private

  def fallback
    Sync do
      barrier = Async::Barrier.new
      barrier.async do
        yield
      end

      begin
        barrier.wait
      rescue Async::TimeoutError => exception
        report(exception)
        render_error_response unless performed?
      rescue StandardError => exception
        handle_exception_and_try_fallback(exception) do
          yield
        end
      ensure
        barrier.stop
      end
    end
  end

  def handle_exception_and_try_fallback(exception)
    process_exception(exception)
    @context.fallback = @context.source

    barrier = Async::Barrier.new
    barrier.async do
      yield
    end

    begin
      barrier.wait
    rescue StandardError => fallback_exception
      process_fallback_exception(fallback_exception)
      render_error_response
    ensure
      barrier.stop
    end
  end
end
```

The excerpt drops the test-environment guards and the exception classification; the control flow is unchanged. `Barrier#stop` cancels every task the barrier still holds and closes its finished queue, and the cancellation propagates down the task tree, so the fan-out fibers spawned inside the action are stopped along with the action itself. The fallback then gets a fresh barrier of its own.

The `ensure` took two commits to get right. The first, on March 10, 2025, added `ensure barrier.stop` around both waits, with the note that it was an attempt to improve server performance when upstream sources were slow and timing out. The second, four days later, restructured the method so the ensure was reached on every path. The separate rescue for `Async::TimeoutError` arrived on April 1. A per-leg timeout on one upstream call surfaces as the app's own timeout error, is classified, and gets a fallback. The whole-request deadline is different. It comes from a 15-line Rack middleware that wraps the app in `Async::Task.current.with_timeout`, and when it fires the request has already spent its budget, so trying a fallback would only spend it again. That case renders the error response and stops.

What the barrier cannot do is interrupt work that never yields. Cancellation is delivered when the reactor resumes the fiber, so a fetch blocked in C code, or a large parse, runs to completion before it notices. That limit is why the codebase caps request-path work rather than trusting the deadline, and it is why a test double exists for the timeout paths: a mock source whose only endpoint is `task.sleep` inside `with_timeout`, so the fallback and deadline branches can be exercised locally without a slow vendor.

## A Detached Fiber

Some work belongs after the response and does not need to be awaited. The per-request counter is the case here. In December 2025 the counter's Redis backing was replaced with direct Postgres writes, and the monitoring showed web transaction times climbing from a baseline of about 5ms to 25 to 45ms during peaks. The write was synchronous, and every response waited for it.

The alternative would have been a job queue, which is a second process, a table, and a delay for a single upsert. The shape chosen is an `Async` block with no `wait`:

```ruby
class RequestCounter
  def save
    Async do
      CounterRow.upsert_from(**attrs_hash)
    rescue ActiveRecord::ActiveRecordError => exception
      report(exception)
    end
  end
end
```

Inside a request there is always a current task, and the kernel `Async` method spawns a child of it and returns immediately. The controller renders, the response goes out, and the child fiber runs its write when the reactor gets to it. The rescue is inside the block because nothing outside will ever see an exception from a task nobody waits on. The accepted cost, recorded in the commit, is that a process exiting between the response and the write loses one count.

The same shape carries the GraphQL API, and there it is doing the fan-out's job. graphql-ruby resolves the fields of an object one at a time, so a `Weather` object with a `currently` field and an `hourly` field would fetch the two endpoints in sequence. Since May 2022 every object type uses a field class whose extension wraps resolution in `Async` and returns the task:

```ruby
class Types::AsyncExtension < GraphQL::Schema::FieldExtension
  def resolve(object:, arguments:, **rest)
    Async do
      yield(object, arguments)
    end
  end
end

class Types::AsyncField < GraphQL::Schema::Field
  def initialize(*args, **kwargs, &block)
    super
    extension(Types::AsyncExtension)
  end
end

class Types::AsyncObject < GraphQL::Schema::Object
  field_class Types::AsyncField
end

class Schema < GraphQL::Schema
  lazy_resolve Async::Task, :wait
end
```

The last line is the one to notice. `lazy_resolve` tells graphql-ruby that a field returning an `Async::Task` is a lazy value, to be resolved by calling `wait` only after the sibling fields at that level have all been visited. Every field starts its task, then graphql-ruby waits on each, so the endpoints run concurrently with no barrier in the schema and sixteen lines in total.

The limit of a detached fiber is that it still runs on the reactor. A database write is I/O and yields; anything that computes instead stalls every other request on the worker until it finishes. Under the fiber isolation described in [ActiveRecord Under a Fiber Reactor](/activerecord-under-a-fiber-reactor/), the write takes a scoped connection lease and returns it, so the detached fiber cannot hold a connection past its own query.

## A Thread, on Purpose

The last shape is the counterexample. In May 2026 the team wanted to know whether the counter's direct writes were what pushed the database during the half-hourly refresh spike, and built a buffered mode to find out: aggregate counts in process memory and flush them in batches. A flusher has to wake on a timer or on a threshold, and it has to outlive every request. A detached fiber is the wrong home for that, because it is spawned inside a request's task tree and shares that request's fate under cancellation. The flusher is a thread:

```ruby
class BufferedCounter
  @buffer = {}
  @buffered_requests = 0
  @condition = ConditionVariable.new
  @mutex = Mutex.new
  @worker_thread = nil

  class << self
    def increment(key, requests:, **counters)
      start!

      @mutex.synchronize do
        @buffer[key] ||= Hash.new(0)
        counters.each { |name, n| @buffer[key][name] += n }
        @buffered_requests += requests

        @condition.signal if @buffered_requests >= flush_threshold
      end
    end

    private

    def start!
      @mutex.synchronize do
        return if @worker_thread&.alive?

        @worker_thread = Thread.new do
          loop do
            flush_worker_once
          rescue StandardError => exception
            report(exception)
          end
        end
      end
    end

    def flush_worker_once
      entries = @mutex.synchronize do
        @condition.wait(@mutex, flush_interval) if @buffered_requests < flush_threshold
        drain_buffer
      end

      flush_entries(entries)
    end

    def flush_entries(entries)
      return if entries.empty?

      Rails.application.executor.wrap do
        ActiveRecord::Base.connection_pool.with_connection do
          entries.each { |key, counters| CounterRow.upsert_from(key, **counters) }
        end
      end
    end
  end
end
```

The excerpt trims the deferred variant and the per-row rescue. The request fibers and the worker thread both touch the buffer, so the `Mutex` is not optional, and the `ConditionVariable` with a timed wait gives the thread its two wake conditions in one call. The two wrappers in `flush_entries` are the price of leaving the reactor. A thread the framework did not start has no executor state and no connection lease of its own; `executor.wrap` gives it the former and `with_connection` the latter, and the lease is returned when the block ends.

The mode is selected by an environment variable and is not the default. A production trial of the deferred variant across one refresh spike on May 18, 2026 did not reduce database load, and the database skill now records the result so nobody rebuilds it: the direct path benchmarks faster per call, the toggle stays as diagnostic scaffolding, and the thread only runs when someone turns it on.

## What Shipped and What It Cost

- The semaphore memoizer has been the base of every source adapter since September 2021. Its cost is one semaphore per endpoint per source instance, created lazily and discarded with the request.
- The barrier of one has wrapped every forecast action since March 2025, with two follow-up commits before the ensure was right. Nothing was measured; the motivation was worker time spent on orphaned fetches during upstream slowdowns.
- The detached write went in on December 12, 2025 against a measured climb from about 5ms to 25 to 45ms at peak. The accepted downside is a lost count when a process exits mid-write.
- The GraphQL extension has run since May 2022 and is sixteen lines.
- The buffered thread shipped on May 17, 2026 as a non-default mode, produced a negative result the next day, and stays in the tree as a documented experiment.

## Lessons Learned

- Memoize under a lock keyed by what you are memoizing, and keep the lock in the reactor's own vocabulary. A one-slot semaphore serializes the fill and lets every later caller read the value.
- Wrap work in a barrier when you need to stop it, not only when you need to wait for it. A barrier of one task is a cancellation scope, and the stop belongs in an ensure.
- Separate a per-leg timeout from the whole-request deadline. The first earns a retry elsewhere; the second has already spent the budget.
- A detached fiber is for I/O that can be lost. Rescue inside it, because nothing outside will.
- Work that must outlive requests does not belong in a request's task tree. Give it a thread, and give the thread an executor and a connection lease.
