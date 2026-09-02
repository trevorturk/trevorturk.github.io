---
layout: post
title: "ActiveRecord Under a Fiber Reactor"
date: 2026-09-01 11:00:00 -0600
summary: "Fiber isolation was switched on in 2022, rolled back in 2023 after connection timeouts and an outage, restored in 2024 behind a scoped lease and a connection pooler, and made safe by Rails 7.2's disallowed permanent checkout. In 2026 that same setting turned a gem regression into a loud 500 and an upstream fix."
tags: [ruby, rails, falcon, async, activerecord, postgres]
---

## The Problem

On February 4, 2023, one line in the Rails config of [Hello Weather](https://helloweather.com) was flipped from `:fiber` back to `:thread` with the comment "Beware ActiveRecord::ConnectionTimeoutError if changing to :fiber". Two days later the line was deleted entirely, Rails was pinned to 7.0.x, and the commit message listed the errors from that weekend's database outage. It took until August 2024 for the setting to come back without a workaround attached.

The setting is `config.active_support.isolation_level`. It decides what Rails treats as the unit of execution when it stores per-request state: the current thread or the current fiber. Under Puma the answer is obviously the thread. Under [Falcon](/falcon-async-performance/), every in-flight request in a worker process is a fiber on one reactor thread, so the two answers diverge, and ActiveRecord's connection pool is where the divergence bites first.

The pool keys its leases by that execution context. This is the whole mechanism, quoted from the 8.1 source:

```ruby
# activesupport/lib/active_support/isolated_execution_state.rb (trimmed)
def isolation_level=(level)
  @scope = case level
           when :thread then Thread
           when :fiber  then Fiber
           end
end

def context
  @scope.current
end

# activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb (trimmed)
def connection_lease
  @leases[ActiveSupport::IsolatedExecutionState.context]
end

def lease_connection
  lease = connection_lease
  lease.connection ||= checkout
  lease.sticky = true
  lease.connection
end
```

Under `:thread`, every request fiber on a Falcon worker resolves to the same `Thread.current`, so they all share one lease and one connection. Queries from concurrent requests serialize on it, and per-request state such as the current query cache is shared between requests that should never see each other. Under `:fiber`, each request fiber gets its own lease. That is correct, and it is also why the first attempt failed: in Rails 7.0 and 7.1 a lease taken during a request was sticky until the request finished, so a worker with thirty requests in flight wanted thirty connections from a pool sized at five. The rest waited, and after five seconds they raised `ActiveRecord::ConnectionTimeoutError`.

## The Solution

There was no single fix. The setting went through four states over four years, and each transition depended on something outside the app changing first.

- 2022 to 2023: fiber isolation on, then off after an outage
- 2024: fiber isolation back on behind a scoped lease and a connection pooler
- Rails 7.2: the scoped lease deleted, permanent checkout disallowed
- 2025 to 2026: an unlimited client pool, and the disallow setting catching a gem regression

### On, then off

The setting went in on August 27, 2022, on Rails 7.0.3 and Falcon 0.42, as a single line with no comment. The app had been on Falcon for a year by then, and the fiber-per-request model was the reason it was there.

The rollback commit of February 4, 2023 is the one with the warning comment. The follow-up on February 6 removed the line, pinned Rails, and recorded four error classes from the outage: `ActiveModel::MissingAttributeError` on an attribute with an empty name, `ActiveRecord::UnknownPrimaryKey` on a table that plainly had one, `Async::Stop` inside a `find_by`, and the connection timeout. The first two are schema-cache corruption, which is what shared per-request state looks like when two fibers write to it at once. The commit's own assessment was "I'm not sure if some changes to Rails may have caused the issue or not", and the decision was to run `:thread` and keep an eye on the Rails issues about fiber-safe pools.

By early 2024 Falcon had made `:fiber` the isolation level it sets by default, and the January 30, 2024 upgrade to Rails 7.1 still kept the app on `:thread`, with a comment linking six discussions, including the then-open Rails pull request that would become the 7.2 connection-handling change. The limit of this state is the one above: under `:thread`, concurrent requests on a worker share a connection and share state. It was tolerable at low concurrency and wrong at any concurrency.

### Back on, behind a scoped lease

The February 2, 2024 switch back to `:fiber` cited [socketry/falcon#219](https://github.com/socketry/falcon/pull/219) and gave three reasons the outage would not repeat: a connection pooler now sat between the app and Postgres, the database was larger, and the app no longer held a connection for the whole request. That last part was the workaround. Every request reads two database records up front, the account and its settings. Those reads were moved into a `with_connection` block in the request object's constructor:

```ruby
class ForecastRequest
  def initialize(args)
    @api_key = args[:api_key]
    initialize_database_records_with_connection
  end

  # Re: isolation_level fiber https://github.com/socketry/falcon/pull/219
  # A workaround until Rails improves connection handling: the connection
  # is released at the end of this block, not at the end of the request.
  def initialize_database_records_with_connection
    ActiveRecord::Base.connection_pool.with_connection do
      account
      settings
    end
  end

  def account
    @_account ||= Account.find_by(api_key: @api_key)
  end

  def settings
    @_settings ||= Settings.find_by(account: account)
  end
end
```

The `||=` memoization is what makes the block work. Both records are loaded and cached while the connection is held, so the rest of the request, which spends its time waiting on upstream weather sources, never touches the pool again. A request held a connection for two queries instead of for its whole lifetime, and a five-connection pool could serve far more than five fibers.

The limit is that the guarantee depended on discipline. Any later query outside that block, an analytics write or a lazy association, would take a sticky lease at the old cost, and nothing would tell you.

### Rails 7.2 deletes the workaround

Rails 7.2 changed how model queries get their connection. A query on a model or relation now runs inside `with_connection`, which checks a connection out for the query and returns it, unless something has already leased one for the request. In 8.1 the pool's `with_connection` reads like this:

```ruby
def with_connection(prevent_permanent_checkout: false)
  lease = connection_lease
  if lease.connection
    yield lease.connection
  else
    begin
      yield lease.connection = checkout
    ensure
      release_connection(lease) unless lease.sticky
    end
  end
end
```

The `sticky` flag is the difference between the two worlds. `lease_connection`, and the old `ActiveRecord::Base.connection` that calls it, set the lease sticky and the connection stays leased until the request completes. A bare query leaves it unset and the connection goes back in the `ensure`. So the scoped block in the constructor was now what every query did on its own, and the August 13, 2024 upgrade to Rails 7.2 deleted it. The same commit added one line:

```ruby
# config/application.rb
config.active_record.permanent_connection_checkout = :disallowed
config.active_support.isolation_level = :fiber
```

`permanent_connection_checkout` governs `ActiveRecord::Base.connection`. The default of `true` allows it, `:deprecated` warns, and `:disallowed` raises. Under 8.1 the check is exact:

```ruby
# activerecord/lib/active_record/connection_handling.rb (trimmed)
def connection
  pool = connection_pool
  if pool.permanent_lease?
    case ActiveRecord.permanent_connection_checkout
    when :disallowed
      raise ActiveRecordError, "Called deprecated `ActiveRecord::Base.connection` method. " \
                               "Either use `with_connection` or `lease_connection`."
    end
    pool.lease_connection
  else
    pool.active_connection
  end
end
```

For a fiber-per-request app this is the setting that matters. A permanent lease is the thing that turns a request fiber into a held connection, and it is invisible in production until the pool runs dry during a spike. With `:disallowed`, the first code path that would hold one raises in development, with the caller in the backtrace. The limit is that it only guards `Base.connection`. `lease_connection` is still allowed, as is an open transaction, so a long transaction on the request path still holds a connection for its duration.

### An unlimited pool behind a pooler

With leases scoped to queries, the pool size stopped being a concurrency ceiling and became a cap on burst. In December 2025, a job that enqueued roughly 9,300 rows in under a second ran into the default pool of five while the request path was also writing hit counters, and the timeouts came back. The fix was to stop enforcing a client-side limit:

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  max_connections: -1
```

The reasoning is that the real limit lives in the connection pooler in front of Postgres, and Rails opens connections lazily, so an unlimited pool never opens more than the process actually needs at once. Rails 8.1 renamed `pool` to `max_connections` and documents `-1` as unlimited. The first attempt wrote `max_connections: nil`, which YAML parses as the string `"nil"`, which `to_i` turns into a pool of zero, which times out every checkout in every environment. The correction the same day is why the value is `-1` and not `null`.

The limit is that this only works with a pooler that enforces a server-side cap. Without one, an unlimited client pool under a fiber server is a way to exhaust Postgres connections from a single dyno.

### The setting earns its keep

On July 26, 2026 a routine dependency update took Solid Queue to 1.5.0, and the job dashboard started returning 500. The new release had added a module that called `ActiveRecord::Base.connection`, the deprecated form that `:disallowed` refuses. Under the default setting that call would have leased a connection for the length of a dashboard request and nobody would have noticed. Here it raised on every page load, the backtrace named the gem file, and the fix took two steps: pin the gem below 1.5 with the upstream link on the same line, and submit [rails/solid_queue#769](https://github.com/rails/solid_queue/pull/769). The fix shipped in 1.5.1 and the pin was dropped on August 3 along with a row in the plans file that had been waiting for exactly that release.

The regression test is an integration test that loads the dashboard, with a comment saying what it catches:

```ruby
class JobsTest < ActionDispatch::IntegrationTest
  # Catches gems breaking under permanent_connection_checkout = :disallowed
  test "jobs dashboard loads" do
    get "/jobs", headers: basic_auth_headers
    assert_response :success
  end
end
```

The limit is that it covers one gem's one page. A different gem calling `Base.connection` from a job would surface in the worker's error reporter, not in CI.

### A thread inside the reactor

One write path deliberately leaves the reactor. A buffered counter mode, selectable by environment variable and not the default, accumulates per-request increments in a class-level hash and flushes them from a dedicated thread. Under a fiber server, a plain `Thread.new` is outside every Rails wrapper: no executor, no query cache, and a lease keyed by a fiber Rails has never seen. The flush wraps itself:

```ruby
def flush_entries(entries)
  return if entries.empty?

  Rails.application.executor.wrap do
    ActiveRecord::Base.connection_pool.with_connection do
      entries.each { |key, counters| Counter.add(key, **counters) }
    end
  end
rescue StandardError => exception
  ErrorReporter.capture(exception)
end
```

`executor.wrap` runs the same hooks a request gets, including releasing whatever the block leased when it ends, and `with_connection` scopes the checkout to the flush. The pattern is two lines and it is the correct one for any thread a fiber app starts on purpose. Trimmed here from a class that also handles a deferred mode and a flush threshold.

## Results

- The app has run `isolation_level = :fiber` with `permanent_connection_checkout = :disallowed` since August 2024, with no connection-handling workaround in application code since then. The pool is unlimited on the client side as of December 2025.
- The 2023 outage was not measured beyond the error list in the commit. The 2025 pool exhaustion was: a fan-out of about 9,300 inserts against a pool of five, during the half-hourly spike window that the [capacity guardrails](/heroku-capacity/) sample.
- The 2026 gem regression cost one afternoon and produced an upstream fix in the next patch release. The `:disallowed` setting is what made it a 500 instead of a slow leak.
- What it cost: an app on this configuration cannot use any gem that calls `ActiveRecord::Base.connection` on a request or job path until the gem is fixed. That is the trade accepted, and the integration test is the tripwire for it.

## Lessons Learned

- If your server runs a request per fiber, set the isolation level to fiber. Thread isolation makes every concurrent request on a worker share one lease and one bag of state.
- A sticky lease is the concurrency ceiling, not the pool size. Scope connection checkout to the query, and let the pool be as large as the pooler in front of it allows.
- Disallow permanent checkout as soon as your Rails version supports it. It turns "held a connection for the whole request" from an invisible scaling bug into an exception with a backtrace.
- A thread you start inside a reactor app gets none of Rails' request wrappers. Wrap it in the executor and take its connection with a block.
- When a strict setting breaks a dependency, fix the dependency upstream and leave a dated trigger for dropping the pin.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent researched the web repository's async and Falcon history and proposed this post as a candidate; a second agent verified the history and wrote it. Sources: the `config/application.rb` commits of August 27, 2022, February 4 and 6, 2023, January 30, 2024 (Rails 7.1), February 2, 2024 (#730), and August 13, 2024 (Rails 7.2); the December 17, 2025 pool commits (#1058, #1059); the July 27 and August 3, 2026 Solid Queue commits (#1687, #1794) and the jobs dashboard integration test; the buffered hit counter; and the ActiveSupport and ActiveRecord 8.1 source for `IsolatedExecutionState`, `connection_lease`, `with_connection`, and `Base.connection`. Judgment calls: the mechanism excerpts are quoted from the ActiveRecord 8.1 source installed in the repository rather than from the 7.2 release the app adopted, and the post says so; the request object and counter excerpts use illustrative class names and are trimmed to the connection-handling lines; the Postgres host, the pooler product, the error reporter, and the job that caused the 2025 pool exhaustion are not named; the outage error list is quoted from the commit but the cause is stated only as far as the commit states it. The Rails pull request number linked from the January 2024 comment is not reproduced because its title was not verified.
