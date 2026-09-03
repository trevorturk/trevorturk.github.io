---
layout: post
title: "ActiveRecord Under a Fiber Reactor"
date: 2026-09-01 11:00:00 -0600
summary: "We turned on fiber isolation in 2022, turned it off in 2023 after connection timeouts and an outage, turned it back on in 2024 with a workaround and a connection pooler, and deleted the workaround when Rails 7.2 let us forbid permanent connection checkout. In 2026 that same setting turned a gem regression into a 500 we couldn't miss, and an upstream fix."
tags: [ruby, rails, falcon, async, activerecord, postgres]
model: "Claude Fable 5.1"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

On February 4, 2023, we flipped one line in the Rails config of [Hello Weather](https://helloweather.com) from `:fiber` back to `:thread`. The comment next to it said "Beware ActiveRecord::ConnectionTimeoutError if changing to :fiber". Two days later we deleted the line, pinned Rails to 7.0.x, and listed the errors from that weekend's database outage in the commit message. The setting didn't come back without a workaround until August 2024.

The setting is `config.active_support.isolation_level`. Rails keeps some state per request, like which database connection the request is using, and this setting tells Rails what counts as "the current request": the current thread or the current fiber. Under Puma each request runs on its own thread, so the thread is the obvious answer. Under [Falcon](/falcon-async-performance/) every request in a worker process is a fiber (a lightweight task Ruby switches between), and all of them run on one thread. That's where the two answers split, and ActiveRecord's connection pool is the first place it hurts.

The pool tracks who holds which connection with a lease, one per "current request". Here's how it looks the lease up, quoted from the Rails 8.1 source:

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

Under `:thread`, every request fiber on a Falcon worker gets the same `Thread.current`, so they all share one lease and one connection. Their queries wait on each other, and per-request state like the query cache leaks between requests that should never see each other. Under `:fiber`, each request gets its own lease. That's the right behavior, and it's also why our first attempt failed. In Rails 7.0 and 7.1, once a request took a lease it kept the connection until the request finished. A worker with thirty requests in flight wanted thirty connections from a pool of five. The rest waited five seconds and then raised `ActiveRecord::ConnectionTimeoutError`.

## The Solution

There wasn't one fix. The setting went through four states over four years, and each change waited on something outside the app changing first.

- 2022 to 2023: fiber isolation on, then off after an outage
- 2024: back on, with a workaround in the app and a connection pooler in front of Postgres
- Rails 7.2: the workaround deleted, and permanent checkout disallowed
- 2025 to 2026: an unlimited client pool, and the disallow setting catching a gem regression

### On, then off

We added the setting on August 27, 2022, on Rails 7.0.3 and Falcon 0.42, as one line with no comment. The app had been on Falcon for a year by then, and we'd picked Falcon because it runs each request in its own fiber.

The February 4, 2023 rollback is the commit with the warning comment. The follow-up on February 6 removed the line, pinned Rails, and listed four errors from the outage: `ActiveModel::MissingAttributeError` on an attribute with an empty name, `ActiveRecord::UnknownPrimaryKey` on a table that plainly had one, `Async::Stop` inside a `find_by`, and the connection timeout. The first two mean the schema cache was corrupted, which happens when two fibers write to shared state at the same time. The commit said "I'm not sure if some changes to Rails may have caused the issue or not". We decided to run `:thread` and watch the Rails issues about fiber-safe pools.

By early 2024 Falcon set `:fiber` by default. Our January 30, 2024 upgrade to Rails 7.1 still kept the app on `:thread`, with a comment linking six discussions, including the Rails pull request that later became the 7.2 connection-handling change. The cost of staying on `:thread` is the one above: concurrent requests on a worker share a connection and share state. It caused no visible trouble at low traffic, but it was still wrong.

### Back on, behind a scoped lease

On February 2, 2024 we switched back to `:fiber`. The commit cited [socketry/falcon#219](https://github.com/socketry/falcon/pull/219) and gave three reasons the outage wouldn't repeat: a connection pooler now sat between the app and Postgres, the database was bigger, and the app no longer held a connection for the whole request. That last one was the workaround. Every request reads two database records up front, the account and its settings. We moved those two reads into a `with_connection` block in the request object's constructor:

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

The block works because of the `||=` on `account` and `settings`. Both records are loaded and cached while the connection is held. The rest of the request spends its time waiting on upstream weather sources and never touches the pool again. A request held a connection for two queries instead of its whole lifetime, so a five-connection pool could serve far more than five requests.

The limit is that nothing enforced it. Any query that ran later in the request would hold a connection the old way until the request ended, and nothing would tell you.

### Rails 7.2 deletes the workaround

Rails 7.2 changed how model queries get their connection. Each query now runs inside `with_connection`, which checks out a connection for that query and returns it afterward, unless something has already leased one for the request. In 8.1 the pool's `with_connection` reads like this:

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

The `sticky` flag is the difference. `lease_connection`, and the old `ActiveRecord::Base.connection` that calls it, mark the lease sticky, and the connection stays out until the request finishes. A plain query leaves it unset, so the connection goes back in the `ensure`. Every query now did on its own what our constructor block did, so the August 13, 2024 upgrade to Rails 7.2 deleted the block. The same commit added one line:

```ruby
# config/application.rb
config.active_record.permanent_connection_checkout = :disallowed
config.active_support.isolation_level = :fiber
```

`permanent_connection_checkout` controls what happens when code calls `ActiveRecord::Base.connection`. The default, `true`, allows it. `:deprecated` warns, and `:disallowed` raises. Here's the check in 8.1:

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

For a fiber-per-request app this is the setting that matters. A permanent lease means a request holds a connection until it finishes, and you can't see that in production until the pool runs dry during a spike. With `:disallowed`, the first code path that would take one raises in development, with the caller in the backtrace. The limit is that it only guards `Base.connection`. `lease_connection` is still allowed, and so is an open transaction, so a long transaction on the request path still holds a connection for as long as it runs.

### An unlimited pool behind a pooler

Once leases were scoped to queries, the pool size no longer capped how many requests could run at once. It only capped how many queries could run at the same instant. In December 2025 a job wrote roughly 9,300 rows in under a second while the request path was writing hit counters, and the default pool of five ran out. The timeouts came back. The fix was to stop enforcing a limit on the client side:

```yaml
# config/database.yml
default: &default
  adapter: postgresql
  encoding: unicode
  max_connections: -1
```

The real limit lives in the connection pooler in front of Postgres. Rails opens connections only when needed, so an unlimited pool never opens more than the process is actually using. Rails 8.1 renamed `pool` to `max_connections` and documents `-1` as unlimited. Our first attempt wrote `max_connections: nil`. YAML reads that as the string `"nil"`, `to_i` turns it into zero, and a pool of zero times out on every checkout in every environment. We fixed it the same day, which is why the value is `-1` and not `null`.

The limit is that this only works with a pooler that caps connections on the server side. Without one, an unlimited client pool under a fiber server can use up every Postgres connection from a single dyno.

### The setting catches a gem regression

On July 26, 2026 a routine dependency update took Solid Queue to 1.5.0, and the job dashboard started returning 500. The new release had added a module that called `ActiveRecord::Base.connection`, the form that `:disallowed` refuses. Under the default setting that call would have held a connection for the length of each dashboard request and nobody would have noticed. Here it raised on every page load and the backtrace named the gem file. We pinned the gem below 1.5 with the upstream link on the same line, and submitted [rails/solid_queue#769](https://github.com/rails/solid_queue/pull/769). The fix shipped in 1.5.1, and on August 3 we dropped the pin and the row in the plans file that had been waiting for that release.

The regression test loads the dashboard, and its comment says what it's for:

```ruby
class JobsTest < ActionDispatch::IntegrationTest
  # Catches gems breaking under permanent_connection_checkout = :disallowed
  test "jobs dashboard loads" do
    get "/jobs", headers: basic_auth_headers
    assert_response :success
  end
end
```

The limit is that it covers one page of one gem. A different gem calling `Base.connection` from a job would show up in the worker's error reporter, not in CI.

### A thread inside the reactor

One write path leaves the reactor on purpose. A buffered counter mode, turned on by an environment variable and off by default, collects per-request increments in a class-level hash and writes them from a separate thread. Under a fiber server, a plain `Thread.new` gets none of the setup a request gets: no executor, no query cache, and a lease keyed by a fiber Rails has never seen. So the flush wraps itself:

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

`executor.wrap` runs the same hooks a request gets, including releasing whatever the block leased when it ends. `with_connection` limits the checkout to the flush. It's two lines, and it's the right two lines for any thread a fiber app starts on purpose. The excerpt is trimmed from a class that also handles a deferred mode and a flush threshold.

## Results

- The app has run `isolation_level = :fiber` with `permanent_connection_checkout = :disallowed` since August 2024, with no connection-handling workaround in the app since then. The client-side pool has been unlimited since December 2025.
- We didn't measure the 2023 outage beyond the error list in the commit. We did measure the 2025 one: about 9,300 inserts against a pool of five, during the half-hourly spike window that the [capacity guardrails](/heroku-capacity/) sample.
- The 2026 gem regression cost one afternoon and got fixed upstream in the next patch release. It was a 500 instead of a slow leak because of the `:disallowed` setting.
- What it costs: we can't use any gem that calls `ActiveRecord::Base.connection` on a request or job path until the gem is fixed. We accept that, and the integration test is there to catch it.

## Lessons Learned

- If your server runs a request per fiber, set the isolation level to fiber. Otherwise every concurrent request on a worker shares one connection and one set of state.
- Concurrency is limited by requests that hold a connection to the end, not by the pool size. Check a connection out per query, and let the pool be as big as the pooler in front of it allows.
- Disallow permanent checkout as soon as your Rails version supports it. A request that holds a connection to the end then raises with a backtrace instead of becoming a scaling bug you can't see.
- A thread you start inside a reactor app gets none of Rails' request setup. Wrap it in the executor and check out its connection with a block.
- When a strict setting breaks a gem, fix the gem upstream and leave a note saying when to drop the pin.
