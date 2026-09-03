---
layout: post
title: "Four Shapes for Async Work"
date: 2026-09-01 14:00:00 -0600
summary: "Four smaller patterns from running Rails on a fiber reactor: a one-slot semaphore for memoizing, a one-task barrier for cancelling, a detached fiber for side effects and GraphQL fields, and a plain thread for work that has to outlive the request."
tags: [ruby, falcon, async, performance, patterns]
model: "Claude Fable 5.1"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

Five fibers ask the same source object for its location record, and four of them start a second HTTP request because the first one hasn't returned yet. That's what `@location ||= fetch(...)` does under a fiber reactor. The check runs, the fetch suspends on the socket, the reactor resumes a sibling fiber, and the sibling runs the same check against the same empty instance variable. Nothing is wrong with `||=`. It assumes nothing else runs between the check and the assignment. Under a reactor, something else runs whenever the assignment is slow.

[Hello Weather](https://helloweather.com) serves its API from Falcon. Every request in flight on a worker is a fiber on one reactor, which means the requests take turns on one thread and swap whenever one of them waits on the network. [Falcon and Ruby Async](/falcon-async-performance/) covers the main shape that follows from that: a barrier that fans a request out across a vendor's endpoints and waits for all of them. This post covers the four smaller shapes we settled on for the questions the fan-out doesn't answer. How do we memoize when the callers are concurrent fibers? How do we cancel work that isn't a fan-out? Where does a fire-and-forget side effect run? And when should a piece of work leave the reactor entirely? Each section gives the alternative we tried or considered, the mechanism, the code as it runs today, and the limit the code can't enforce.

## A Semaphore as a Memoizer

The alternative is the one the opening describes, and the app ran it until September 2021. Each source adapter memoized its endpoint responses with `||=`, meaning it saved each response in an instance variable the first time and reused it after that. Since the fan-out barrier starts the fetches at the same time, the race in the opening was routine. The worst case is the vendor whose five endpoints all depend on a location lookup. The current-conditions, hourly, daily, and alerts fibers each call `location_key`, which calls `location_data`, which finds the variable unset because the first fetch is still waiting on the socket. A contributor's commit from that month states the problem in one sentence: once `wait` is called, the reactor yields to the next fiber, and that fiber may end up calling into the same code.

The fix wraps the memoized section in an `Async::Semaphore` with its default limit of one, keyed by the name of the calling method. A semaphore with a limit of one is a lock that lets one fiber through at a time. The base class every adapter inherits from carries it:

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

An adapter's endpoint methods then read as plain memoized fetches, and the concurrency doesn't show at the call site:

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

The line that does the work is `semaphore(name).async(&block).wait`. In the async gem, `Semaphore#async` first waits until the count is below the limit, then spawns a child task that increments the count and releases it in an `ensure`. A second fiber arriving while the first holds the semaphore goes onto the semaphore's waiting list and hands control back to the scheduler until the release wakes it. By the time it enters the block, the hash has the key, and the `unless` skips the fetch. The `Sync` wrapper was added a few days after the semaphore. It yields the current task when there is one and starts a reactor when there isn't, so the same code runs under Falcon and in a test with no reactor.

Two things limit this. The semaphores live on the source instance, so the memo deduplicates fetches within one request's fan-out and nothing more. Deduplicating across requests is the CDN's job, as [CloudFront as an Infinite Cache](/cloudfront-cdn-architecture/) describes. And the key is the caller's method name, so two endpoint methods with the same name in one class would share a lock and a memo slot. The base class has never had two, and nothing checks.

## A Barrier of One Task

The fan-out barrier holds several tasks and waits for all of them. The controller uses a barrier that holds one task, and it's there for a different reason: cancellation. When a request fails partway through its fan-out, the fibers the failure left behind have to stop before the fallback starts. Otherwise the fallback shares the worker with a set of orphaned fetches that will finish and be thrown away.

The alternative was the plain rescue the controller used before March 2025. It caught the exception and moved on with the children still running. The current shape wraps the action in a barrier, waits, and stops the barrier in an `ensure` on every exit:

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

The excerpt drops the test-environment guards and the exception classification; the control flow is unchanged. `Barrier#stop` cancels every task the barrier still holds and closes its finished queue. The cancellation runs down the task tree, so the fan-out fibers spawned inside the action stop along with the action itself. The fallback then gets a fresh barrier of its own.

The `ensure` took two commits to get right. The first, on March 10, 2025, added `ensure barrier.stop` around both waits, with a note that it was an attempt to improve server performance when upstream sources were slow and timing out. The second, four days later, restructured the method so the ensure ran on every path. The separate rescue for `Async::TimeoutError` arrived on April 1. A timeout on one upstream call shows up as the app's own timeout error, gets classified, and gets a fallback. The whole-request deadline is different. It comes from a 15-line Rack middleware that wraps the app in `Async::Task.current.with_timeout`. When it fires, the request has already spent its budget, so trying a fallback would only spend it again. That case renders the error response and stops.

The barrier can't interrupt work that never yields. Cancellation arrives when the reactor resumes the fiber, so a fetch blocked in C code, or a large parse, runs to the end before it sees the cancellation. That's why we cap work on the request path rather than trusting the deadline, and why a test double exists for the timeout paths: a mock source whose only endpoint is `task.sleep` inside `with_timeout`, so we can exercise the fallback and deadline branches locally without a slow vendor.

## A Detached Fiber

Some work belongs after the response and doesn't need to be waited on. The per-request counter is the case here. In December 2025 we replaced the counter's Redis backing with direct Postgres writes, and monitoring showed web transaction times climbing from a baseline of about 5ms to 25 to 45ms during peaks. The write was synchronous, so every response waited for it.

The alternative would have been a job queue, which means a second process, a table, and a delay, all for a single upsert. The shape we chose is an `Async` block with no `wait`:

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

Inside a request there's always a current task, and the kernel `Async` method spawns a child of it and returns right away. The controller renders, the response goes out, and the child fiber runs its write when the reactor gets to it. The rescue is inside the block because nothing outside will ever see an exception from a task nobody waits on. The accepted cost, recorded in the commit, is that a process exiting between the response and the write loses one count.

The same shape carries the GraphQL API, and there it does the fan-out's job. graphql-ruby resolves the fields of an object one at a time, so a `Weather` object with a `currently` field and an `hourly` field would fetch the two endpoints one after the other. Since May 2022 every object type uses a field class whose extension wraps resolution in `Async` and returns the task:

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

The last line is the one to notice. `lazy_resolve` tells graphql-ruby that a field returning an `Async::Task` is a lazy value, and to resolve it by calling `wait` only after it has visited all the sibling fields at that level. So every field starts its task first, and then graphql-ruby waits on each one. The endpoints run concurrently with no barrier in the schema, in sixteen lines.

The limit of a detached fiber is that it still runs on the reactor. A database write is I/O, so it yields. Anything that computes instead stalls every other request on the worker until it finishes. Under the fiber isolation described in [ActiveRecord Under a Fiber Reactor](/activerecord-under-a-fiber-reactor/), the write borrows a database connection for its one query and returns it, so the detached fiber can't hold a connection past that.

## A Thread, on Purpose

The last shape is the exception. In May 2026 we wanted to know whether the counter's direct writes were loading the database during the half-hourly refresh spike, and built a buffered mode to find out: aggregate counts in process memory and flush them in batches. A flusher has to wake on a timer or on a threshold, and it has to outlive every request. A detached fiber is the wrong home for that, because it's spawned inside a request's task tree and gets cancelled along with that request. The flusher is a thread:

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

The excerpt trims the deferred variant and the per-row rescue. The request fibers and the worker thread both touch the buffer, so the `Mutex` is required, and the `ConditionVariable` with a timed wait gives the thread its two wake conditions in one call. The two wrappers in `flush_entries` are there because the thread is outside the reactor. A thread the framework didn't start has no executor state and no database connection of its own. `executor.wrap` gives it the first and `with_connection` the second, and the connection goes back to the pool when the block ends.

The mode is selected by an environment variable and isn't the default. A production trial of the deferred variant across one refresh spike on May 18, 2026 didn't reduce database load, and the database skill now records that result so nobody rebuilds it. The direct path benchmarks faster per call, the toggle stays as diagnostic scaffolding, and the thread only runs when someone turns it on.

## What Shipped and What It Cost

- The semaphore memoizer has been the base of every source adapter since September 2021. It costs one semaphore per endpoint per source instance, created on first use and discarded with the request.
- The barrier of one has wrapped every forecast action since March 2025, with two follow-up commits before the ensure was right. We didn't measure anything; the motivation was worker time spent on orphaned fetches during upstream slowdowns.
- The detached write went in on December 12, 2025 against a measured climb from about 5ms to 25 to 45ms at peak. The accepted downside is a lost count when a process exits mid-write.
- The GraphQL extension has run since May 2022 and is sixteen lines.
- The buffered thread shipped on May 17, 2026 as a non-default mode, produced a negative result the next day, and stays in the tree as a documented experiment.

## Lessons Learned

- Memoize under a lock keyed by the thing you're memoizing, and use the reactor's own lock. A one-slot semaphore lets one caller do the fill and every later caller read the value.
- Wrap work in a barrier when you need to stop it, not only when you need to wait for it. A barrier of one task gives you a way to cancel the whole action, and the stop belongs in an ensure.
- Keep a timeout on one upstream call separate from the whole-request deadline. The first can fall back to another source; the second has already spent the budget.
- A detached fiber is for I/O you can afford to lose. Rescue inside it, because nothing outside will.
- Work that has to outlive requests doesn't belong in a request's task tree. Give it a thread, and give the thread an executor and a database connection.
