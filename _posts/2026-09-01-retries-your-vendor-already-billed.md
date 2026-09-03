---
layout: post
title: "Retries Your Vendor Already Billed"
date: 2026-09-01 10:00:00 -0600
summary: "We audited how our Rails app uses async-http and found three defaults nobody had chosen. The client tries a request up to three times, including on errors that can happen after a paid request already reached the vendor. Our error-path cleanup read the rest of the body with no timeout. And a pool limit, which sounded careful, would have added waiting instead of safety."
tags: [ruby, async, async-http, falcon, http, api-design]
model: "Claude Fable 5.1"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

By default, `Async::HTTP::Client#call` tries each request up to three times. Some of the errors it retries on happen after the request has already reached the server. When the request is a GET to a weather vendor that bills per call, the second and third attempts are calls we pay for and never see. Our hit counter records one hit per response that comes back to us, so a retried request counts as one call in our books and two or three in the vendor's.

[Hello Weather](https://helloweather.com) fetches every weather source through one wrapper around async-http, running under Falcon (see [Falcon and Ruby Async](/falcon-async-performance/)). Since the day it was written, that wrapper had used the shared `Async::HTTP::Internet.instance` with the library defaults. Searching the wrapper's history for the word `retries` finds nothing. Nobody had set the option either way, because nobody had read the code that used it.

In August 2026 the async-http maintainer rewrote the client guides ([socketry/async-http#241](https://github.com/socketry/async-http/pull/241), [#242](https://github.com/socketry/async-http/pull/242), [#243](https://github.com/socketry/async-http/pull/243)). We read the installed gem source alongside the new guides, then compared both with how our app uses the library. Three defaults came out of that reading. We changed two and left the third alone.

## The Solution

We wrote the audit up as a plan on August 21, 2026, checked against async-http 0.100.0 and its dependencies. Two pull requests carry the changes. One sets the retry count on a client the wrapper owns. The other switches the fetch to the block form, so the response closes whether the block returns or raises. The third decision, on the connection pool limit, is a written won't-fix with a recipe for measuring the pool before anyone changes it.

### One attempt per vendor call

The retry logic lives in `Async::HTTP::Client#call`. At 0.102.0, the version in our lockfile as of September 1, it has two rescue clauses. The first catches `Protocol::HTTP::RefusedError`, which means the server did not process the request: an HTTP/1 write failed, or HTTP/2 answered with GOAWAY or REFUSED_STREAM. Those retry for any method, because the request never arrived. The second clause catches `RemoteError`, `SocketError`, `IOError`, `EOFError`, `ECONNRESET`, and `EPIPE`. Those retry only for methods the request considers safe. POST, PATCH, and CONNECT are not safe. GET is.

That second clause is the problem. A connection can drop while the client waits for headers or reads the body, after the server has already received the whole request. Since 0.99.0 the client also retries when an HTTP/2 server resets the stream with `INTERNAL_ERROR` before responding. In all of those cases the vendor may already have processed and billed the call. A CDN in front of the vendor absorbs a retry when the cache is warm, but on a miss each attempt goes through to the vendor.

We considered adding our own retry on `RefusedError` alone, the one error that can't cause a double charge. We put that off. The plan is to stop the library from retrying first, watch what errors show up, and add the narrow retry only if they matter.

`retries:` is set when a client is built, not per request, and `Internet.instance` takes no options. So vendor calls have to leave the shared instance, and the wrapper has to hold an `Internet` of its own:

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

The instance is one per thread, and the review thread explains why. `Internet` keeps a plain hash of clients keyed by host, with no lock around it. Background job threads share this wrapper with the web server, so one instance for the whole class would race. One per fiber would go wrong the other way. Falcon runs each request as a fiber, so a per-fiber `Internet` would open a fresh connection pool for every request and lose the connection reuse we wanted from the shared instance. One per thread is safe, and the connections stay warm.

`retries: 0` would behave the same, because the check is `attempt < @retries`, but `1` says what it means: one attempt. The test checks the setting where it ends up, on the client the `Internet` builds for an endpoint. `Internet` wraps each client in an `AcceptEncoding` middleware, so the test reaches through `delegate`:

```ruby
test "vendor clients make one attempt" do
  endpoint = Async::HTTP::Endpoint.parse("https://example.test")
  client = VendorHttp.internet.client_for(endpoint)

  assert_equal 1, client.delegate.retries
end
```

Other callers in the app stay on `Internet.instance`: push notifications, a newsletter service, App Store tooling. They send POSTs, which the second clause never retries, and none of them bill per call.

The code can't hide the cost. The library retries had been quietly covering real connection blips: an HTTP/2 GOAWAY when the CDN recycles a long-lived connection, a stale HTTP/1 keep-alive, a reset. With one attempt, those become errors. Where no fallback source covers the call, they become a failed user request. The plan calls this a change in visibility, not a new failure, because the same exceptions already escaped once three attempts were used up. One attempt changes how often they escape, not which ones can.

The first draft of the PR also mapped those transport errors into the app's data-error class, so they'd show up as 502s with a metric instead of 500s with an error report. The owner pushed back. That mapping wasn't in the brief, and the repository's rule is to rescue only errors the error reporter has actually shown. Hiding the blips would hide the signal the change was meant to expose. A smaller diff, 16 insertions and 6 deletions instead of the original 40 and 7, is ready to replace it. As of September 1 the PR still carries the larger version, and the thread records the smaller one as the next step.

One thing has changed since the plan was written. async-http 0.102.0 refuses a request before writing it if the HTTP/2 connection it was assigned has already closed. It also stops handing out connections that received a graceful GOAWAY, while letting their in-flight streams finish. Both changes turn cases that used to need a retry into cases that don't fail at all, so a single attempt will expose fewer blips.

### Close, do not drain, on the error path

The wrapper cleaned up with `ensure response&.finish` on the outer `Async` block. `finish` reads the rest of the body so the connection can be reused. `close` throws the rest away. After a full `read` both do nothing, so on the success path the choice never mattered.

On every error path it did. When the wrapper raises on a non-2xx status, a parse error, or a timeout, the body is still unread, and `finish` reads it with no timeout running. The `ensure` belongs to the `Async` block, not to `task.with_timeout`, so the deadline that gave up on the request has already fired before the read starts. For HTTP/1 that read is bounded by the length of whatever error page the vendor sent. For HTTP/2 the body is a queue that the connection writes into. `read` is a `Queue#pop`, the timeout sent no RST_STREAM, and the pop blocks until the vendor ends the stream. The request fiber sits there for as long as the vendor takes.

The one-line fix would swap `finish` for `close`. The owner chose the block form of `Internet#get` instead, because it's the maintainer's current idiom and it closes the response whether the block returns or raises. The `Internet#call` source is short: `yield response` inside a `begin`, `response.close` in the `ensure`. Here is the wrapper with both changes applied, trimmed to the fetch path, with our logging and CDN header checks removed:

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

The line to notice is `status = response.status` at the top of the block. By the time the outer `rescue` runs, the response object is gone, so the status is copied into a local before anything can raise. The old version kept the whole response in an outer variable for the same reason, and that is why a trailing `finish` was possible at all.

The cost is small. On an error path, an HTTP/1 keep-alive socket is dropped instead of drained, which is the right thing to do with a connection whose response we abandoned. The rewrite also had to move the development request log, so it starts before the request and finishes when headers arrive, because a test helper works out whether requests ran in parallel or in series from the timing of those log lines. We only found that coupling by running the suite.

### Leave the pool limit unset

The third default is the `limit` on each host's connection pool, which we've never set. The guides discuss it, and setting one sounds like the careful choice. The audit said the opposite.

With `limit: nil`, `Async::Pool::Controller#acquire` never waits. If no connection is free, it opens another. A limit adds a wait: a fiber-aware condition wait with no timeout of its own. That wait would show up as our own two-second request timeout, during exactly the traffic bursts where a limit would kick in. In production every vendor sits behind its own CDN host, which negotiates HTTP/2 at 128 streams per connection. So each pool already holds about one connection, and a source's burst is one to five requests at once. Without a pool policy there is nothing that closes idle connections. HTTP/1 connections stay open until the other side drops them, and the viability probe added in 0.98.1 retires them lazily.

The plan says when to revisit this: if we run out of sockets, or a vendor complains about connection counts. Even then, the first step is to measure. The recipe logs each host's pool from the request path during a spike:

```ruby
VendorHttp.internet.clients.each do |key, wrapper|
  Rails.logger.info "#{key} #{wrapper.delegate.pool}"
end
```

The pool's `inspect` shows usage over capacity for each connection. A capacity of 128 means HTTP/2. A capacity of 1 means HTTP/1. Only an HTTP/1 pool growing to tens of connections would justify a limit, and the limit would go above the biggest healthy burst we'd seen.

## Results

- Neither PR has merged. Both opened on August 21, 2026 and were still open as of September 1. The retries PR was closed as won't-fix on August 31, swept up in a stand-down of unrelated vendor work, and reopened the same day once the billing reason was separated out.
- The risk is real, but so far we haven't seen it. No vendor invoice shows meaningful duplicate billing. The one possible sign is a vendor whose invoice runs about 1% above our own counter. That gap is stable, and we check it monthly.
- The cost of shipping is a few more connection errors in the error reporter, on top of the roughly 20 to 50 bad-gateway errors a day we already see. The plan says to watch for two weeks after deploy.
- The audit produced four rules for the source-fetching skill, to be moved there when the plan closes: one attempt per vendor fetch, fetch through the block form, leave the pool limit unset, and run the Falcon line-length test on any async-http or protocol-http1 bump.

## Lessons Learned

- If no commit in your history sets a client option, you've been running on someone else's choice. Read the code path that uses it.
- Retry only on errors that prove the server never got the request. Any error that can happen after the request was sent might mean a duplicate charge.
- Before designing the fix, read the gem source to see where the option is set. If it's set when the client is built and the client is shared, you'll need your own client.
- On an error path, close the response instead of reading the rest of it. Reading with no deadline means waiting as long as the other server takes.
- A limit on a pool that never waits doesn't add safety. It adds a wait, and that wait needs its own timeout.
