---
layout: post
title: "Retries Your Vendor Already Billed"
date: 2026-09-01 10:00:00 -0600
summary: "An audit of how a Rails app drives async-http found three defaults nobody had set: the client makes up to three attempts on errors that can fire after a metered request was sent, an error-path cleanup call drains bodies with no timeout, and a pool limit that would add queueing instead of safety."
tags: [ruby, async, async-http, falcon, http, api-design]
---

## The Problem

`Async::HTTP::Client#call` makes up to three attempts per request by default, and some of the errors it retries on can fire after the request has already reached the server. For a GET against a weather vendor that bills per call, the second and third attempts are invoices the application never sees. The app's own hit counter records one hit per response that comes back to it, so a retried request looks like one call in the app's books and two or three in the vendor's.

[Hello Weather](https://helloweather.com) fetches every weather source through a single wrapper around async-http, running under Falcon (see [Falcon and Ruby Async](/falcon-async-performance/)). That wrapper had used the shared `Async::HTTP::Internet.instance` with library defaults since it was written. A search of the wrapper's history for the word `retries` returns nothing. Nobody had set the option, in either direction, because nobody had read the code path that used it.

What changed in August 2026 was the documentation. The async-http maintainer rewrote the client guides ([socketry/async-http#241](https://github.com/socketry/async-http/pull/241), [#242](https://github.com/socketry/async-http/pull/242), [#243](https://github.com/socketry/async-http/pull/243)). Rather than skim them, the team read the installed gem source against the guides, then against the app's usage. Three defaults came out of that reading, each with a verdict: change it, change it, leave it alone.

## The Solution

The audit was recorded as a plan on August 21, 2026, verified against async-http 0.100.0 and its dependencies. Two pull requests carry the changes. One sets the retry count on a client the wrapper owns. The other switches the fetch to the block form so the response closes on every exit. The third verdict, on the connection pool limit, is a written won't-fix with a measurement recipe attached.

### One attempt per vendor call

The retry logic lives in `Async::HTTP::Client#call`, and at 0.102.0, the version in the app's lockfile as of September 1, it reads like this. A `Protocol::HTTP::RefusedError` means the server provably did not process the request: an HTTP/1 write failure, or an HTTP/2 GOAWAY or REFUSED_STREAM. Those retry for any method, because the request never landed. A second rescue clause covers `RemoteError`, `SocketError`, `IOError`, `EOFError`, `ECONNRESET`, and `EPIPE`. Those retry only for methods the request considers safe, which excludes POST, PATCH, and CONNECT, and includes GET.

That second group is the problem. A connection can drop while the client is waiting for headers or reading the response, after the server has fully received the request. Since 0.99.0 the client also retries when a remote HTTP/2 endpoint resets the stream with `INTERNAL_ERROR` before responding. In every one of those cases the origin may already have processed and billed the call. A CDN in front of the vendor absorbs retries on a warm cache, but a miss forwards each attempt to the origin.

The alternative considered was an app-level retry on `RefusedError` alone, which is the only provably billing-safe form. It was deferred rather than built: first stop the library from retrying, watch what surfaces, and add the narrow retry only if the blips are material.

`retries:` is a client-construction option, not a per-request one, and `Internet.instance` takes no options. So the fix has to abandon the shared instance for vendor calls and hold an `Internet` the wrapper owns:

```ruby
require "async"
require "async/http/internet"

class VendorHttp
  # retries: 1 means one attempt. The library default is 3.
  def self.internet
    Thread.current.thread_variable_get(:vendor_http_internet) ||
      Thread.current.thread_variable_set(:vendor_http_internet, Async::HTTP::Internet.new(retries: 1))
  end
end
```

The thread-local scope is deliberate and the review thread spells out why. `Internet` keeps an unsynchronized hash of clients keyed by origin, and background job threads share this wrapper with the web server, so a class-level instance would race. Fiber-local would be worse in the other direction: Falcon runs each request as a fiber, so a per-fiber `Internet` would build a fresh connection pool per request and throw away the connection reuse the shared instance exists to provide. One instance per thread is the scope that is both safe and warm.

A value of `retries: 0` would behave identically, since the check is `attempt < @retries`, but `1` reads as what it means: one attempt. The test asserts the setting where it lands, on the client the `Internet` builds for an endpoint. `Internet` wraps each client in an `AcceptEncoding` middleware, so the assertion reaches through `delegate`:

```ruby
test "vendor clients make one attempt" do
  endpoint = Async::HTTP::Endpoint.parse("https://example.test")
  client = VendorHttp.internet.client_for(endpoint)

  assert_equal 1, client.delegate.retries
end
```

Other callers in the app, for push notifications and a newsletter service and App Store tooling, stay on `Internet.instance`. They POST, which the socket-error group never retries, and they are not billed per call.

What the code cannot enforce is the consequence. The library retries were silently absorbing real connection blips: an HTTP/2 GOAWAY when the CDN recycles a long-lived connection, a stale HTTP/1 keep-alive, a reset. With one attempt those surface as errors, and where no fallback source covers the call, as a failed user request. The plan calls this a visibility shift rather than a new failure, since the same exception classes already escaped once three attempts were spent. Going to one attempt changes their frequency, not what can escape.

The first draft of the PR pre-mapped those transport errors into the app's data-error class so they would land as 502s with a metric rather than 500s with an error report. The owner pushed back: that mapping was not in the brief, and the repository's rule is to rescue only what the error reporter has actually shown. Hiding the blips would hide exactly the signal the change was meant to surface. A smaller diff, 16 insertions and 6 deletions against the original 40 and 7, is ready as the replacement. As of September 1 the PR still carries the larger version, with the smaller one recorded in the thread as the next step.

One more thing has moved since the plan was written. async-http 0.102.0 refuses requests assigned to an HTTP/2 connection that has already closed before writing them, and removes connections that received a graceful GOAWAY from availability while letting their accepted streams finish. Both turn cases that used to need a retry into cases that never fail, which shrinks the population of blips a single attempt will expose.

### Close, do not drain, on the error path

The wrapper's cleanup was an `ensure response&.finish` on the outer `Async` block. `finish` reads the rest of the body so the connection can be reused. `close` discards it. After a full `read` both are no-ops, so on the success path the choice never mattered.

On every error path it did. When the wrapper raises on a non-2xx status, a parse error, or a timeout, the body is unread, and `finish` drains it with no timeout in force. The `ensure` belongs to the `Async` block, not to `task.with_timeout`, so the deadline that abandoned the request has already fired by the time the drain starts. For HTTP/1 the drain is a length-bounded read of whatever error page the vendor sent. For HTTP/2 the body is a writable queue: `read` is a `Queue#pop`, the timeout sent no RST_STREAM, and the pop blocks until the vendor concludes the stream. That parks the request fiber for as long as the vendor takes.

The one-line fix would swap `finish` for `close`. The owner chose the block form of `Internet#get` instead, because it is the maintainer's current idiom and it closes on exit whether the block returns or raises. The `Internet#call` source is short: `yield response` inside a `begin`, `response.close` in the `ensure`. The wrapper with both changes applied, trimmed to the fetch path and with the app's logging and CDN header checks removed:

```ruby
require "async"
require "async/http/internet"
require "json"

class VendorHttp
  class Error < StandardError; end
  class TimeoutError < Error; end
  class RateLimitError < Error; end
  class AuthenticationError < Error; end
  class DataError < Error
    attr_reader :status

    def initialize(host, status: nil)
      @status = status
      super("#{host} returned #{status || 'no response'}")
    end
  end

  DEFAULT_TIMEOUT = 2

  def self.internet
    Thread.current.thread_variable_get(:vendor_http_internet) ||
      Thread.current.thread_variable_set(:vendor_http_internet, Async::HTTP::Internet.new(retries: 1))
  end

  def self.get(url, headers = {}, timeout: DEFAULT_TIMEOUT)
    Async do |task|
      host = URI(url).host
      status = nil

      task.with_timeout(timeout) do
        internet.get(url, headers) do |response|
          status = response.status

          raise AuthenticationError, host if [401, 403].include?(status)
          raise RateLimitError, host if status == 429
          raise DataError.new(host, status: status) unless response.success?

          body = response.read
          body = "{}" if body.nil? && status == 204
          raise DataError.new(host, status: status) if body.nil?

          JSON.parse(body, symbolize_names: true)
        end
      end
    rescue Async::TimeoutError
      raise TimeoutError, host
    rescue JSON::ParserError, Protocol::HTTP2::StreamError
      raise DataError.new(host, status: status)
    end
  end
end
```

The line to notice is `status = response.status` at the top of the block. The response object is gone by the time the outer `rescue` runs, so the status is hoisted into a local before anything can raise. The old version kept the whole response in an outer variable for the same reason, which is what made the trailing `finish` possible in the first place.

The cost is small and correct: on an error path an HTTP/1 keep-alive socket is dropped instead of drained, which is what should happen to a connection whose response the app abandoned. The rewrite also had to move the development request log so it starts before the request and completes on headers, because a test helper infers parallel-versus-serial execution from the timing of those log lines. That coupling is the kind of thing a refactor of a client wrapper finds only by running the suite.

### Leave the pool limit unset

The third default is the per-origin connection pool's `limit`, which the app has never set. The guides discuss it, and setting one sounds like the careful choice. The audit's verdict was the opposite.

With `limit: nil`, `Async::Pool::Controller#acquire` never waits. If no connection is available it opens another. Setting a limit is what introduces a wait: a fiber-aware condition wait with no timeout of its own, which would surface as the app's own two-second request timeout during exactly the traffic bursts where a limit would bind. In production every origin is fronted by its own CDN host that negotiates HTTP/2, at 128 streams per connection, so pools already sit at about one connection each, and a source's burst is one to five concurrent requests. There is no idle reaper without a pool policy. HTTP/1 connections persist until the peer drops them and are retired lazily by the viability probe added in 0.98.1.

The plan records the trigger for revisiting this, socket exhaustion or a vendor complaining about connection counts, and insists the first action is measurement. The recipe logs each origin's pool from the request path during a spike:

```ruby
VendorHttp.internet.clients.each do |key, wrapper|
  Rails.logger.info "#{key} #{wrapper.delegate.pool}"
end
```

The pool's `inspect` shows usage over capacity per connection. A capacity of 128 means HTTP/2, a capacity of 1 means HTTP/1. Only an HTTP/1 pool growing to tens of connections would justify a limit, set above the observed healthy burst.

## Results

- Neither PR has merged. Both were opened on August 21, 2026 and were still open as of September 1. The retries PR was closed as won't-fix on August 31, bundled into a stand-down of unrelated vendor work, and reopened the same day once the billing rationale was separated from that work.
- The exposure is real but, so far, theoretical. No vendor invoice shows material duplicate billing. The only candidate signal is one vendor whose invoice runs about 1% above the app's own counter, a gap that is stable and watched monthly.
- The measured cost of shipping is visibility: a few more connection-level errors in the error reporter, judged against a residual of roughly 20 to 50 bad-gateway errors a day. The plan's watch period is two weeks after deploy.
- The audit produced four operating rules for the source-fetching skill, to be moved there when the plan closes: one attempt per vendor fetch, fetch through the block form, leave the pool limit unset, and run the Falcon line-length test on any async-http or protocol-http1 bump.

## Lessons Learned

- A client default is a decision someone made for you. If your history has no commit touching it, you have not made it yet.
- Retry only on errors that prove the server never processed the request. Any error that can fire after the request was sent is a possible duplicate charge.
- Read the gem source for the option's scope before designing the fix. A construction-time option on a shared instance forces you to own an instance.
- On an error path, close the response instead of draining it. A drain with no deadline is a wait of unbounded length on someone else's server.
- A limit on a pool that never waits does not add safety. It adds a wait, and the wait needs its own timeout.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent ran a research pass over the web repository looking for material relevant to Falcon and async-http users and proposed this post as a candidate; a second agent verified the claims and wrote it. Sources: the async-http client hygiene plan (decisions dated 2026-08-21), the two open pull requests and their review thread (opened 2026-08-21, one closed and reopened 2026-08-31), the client wrapper and source base class, and the installed gem source for async-http 0.102.0, async-pool 0.12.0, and protocol-http 0.71.0, read directly to confirm the retry clauses, the `Internet#call` block form, `Pool::Controller#acquire`, and the writable body's `read`.

Judgment calls: the code block combines both pull requests into one wrapper and is trimmed and renamed, which the post says; the app's transport-error mapping is described as the reviewed-and-rejected first draft rather than shown, since the smaller replacement is not yet pushed. The 1% invoice gap is attributed to "one vendor" without a name. The vendors on the shared instance are described by what they do, not by product. The gem version note (plan verified at 0.100.0, lockfile at 0.102.0) was added after reading the 0.102.0 release notes and confirming the retry logic is unchanged.
