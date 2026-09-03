---
layout: post
title: "ActiveRecord Under a Fiber Reactor"
date: 2026-09-01 11:00:00 -0600
summary: "We turned on fiber isolation in 2022, turned it off in 2023 after connection timeouts and an outage, turned it back on in 2024 with a workaround and a connection pooler, and deleted the workaround when Rails 7.2 let us forbid permanent connection checkout. In 2026 that same setting turned a gem regression into a 500 we couldn't miss, and an upstream fix."
tags: [ruby, rails, falcon, async, activerecord, postgres]
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

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent researched the web repository's async and Falcon history and proposed this post as a candidate; a second agent verified the history and wrote it. Sources: the `config/application.rb` commits of August 27, 2022, February 4 and 6, 2023, January 30, 2024 (Rails 7.1), February 2, 2024 (#730), and August 13, 2024 (Rails 7.2); the December 17, 2025 pool commits (#1058, #1059); the July 27 and August 3, 2026 Solid Queue commits (#1687, #1794) and the jobs dashboard integration test; the buffered hit counter; and the ActiveSupport and ActiveRecord 8.1 source for `IsolatedExecutionState`, `connection_lease`, `with_connection`, and `Base.connection`. Judgment calls: the mechanism excerpts are quoted from the ActiveRecord 8.1 source installed in the repository rather than from the 7.2 release the app adopted, and the post says so; the request object and counter excerpts use illustrative class names and are trimmed to the connection-handling lines; the Postgres host, the pooler product, the error reporter, and the job that caused the 2025 pool exhaustion are not named; the outage error list is quoted from the commit but the cause is stated only as far as the commit states it. The Rails pull request number linked from the January 2024 comment is not reproduced because its title was not verified.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 2, run after batch 1 (#68) merged. Prose rewritten paragraph by paragraph in first person with the code, headings, dates, numbers, quotes, and links held fixed: "fiber" and "lease" are now defined at first use, the cleft sentences ("the `||=` memoization is what makes the block work", "the `:disallowed` setting is what made it a 500") were turned around to put the subject first, and the "earns its keep" subheading became "catches a gem regression". Judgment calls: "enqueued roughly 9,300 rows" is now "wrote roughly 9,300 rows", matching the Results bullet's "inserts"; the examples of a stray query ("an analytics write or a lazy association") were cut in favor of the category. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"
