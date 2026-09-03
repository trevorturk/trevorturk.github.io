---
layout: post
title: "Gzip Without a Content-Encoding Header"
date: 2026-07-29 08:20:00 -0600
summary: "A vendor sent gzip bytes in a 200 with no Content-Encoding header, CloudFront cached it for 24 hours, and 12.7k errors a day never showed up in the CDN logs. We check for the gzip magic bytes in the one class every request goes through, and cap the inflated size because Falcon runs everything on one reactor."
tags: [ruby, falcon, cloudfront, debugging, http]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

On 2026-07-15, [Hello Weather](https://helloweather.com) started throwing about 12,700 `bad_gateway` errors a day from a single upstream data provider. Users saw them. Our CDN logs did not.

Some of the time, the provider was returning an HTTP 200 whose body was gzip bytes, with no `Content-Encoding: gzip` header to say so. Every layer of our stack took that as a good response: status 200, a Content-Length, a body with something in it. The body just started with `\x1f\x8b`, the two bytes every gzip stream starts with, instead of `{`.

Our HTTP client read the body, ran `JSON.parse` on it, and raised `DataError`, which is a 502, when the parse failed. That part worked as designed. The problem was the layer above it.

We use [CloudFront as an infinite cache](/cloudfront-cdn-architecture/) in front of every upstream provider, and the cache key includes a `Cache-Expires` boundary time instead of a duration. For this endpoint, that boundary only changed at UTC midnight. So:

1. One fetch to the provider returns a gzip body with no header.
2. CloudFront sees a 200 and caches it.
3. Every later request for that location, for up to 24 hours, gets that cached body from the edge.
4. Each one fails `JSON.parse` and returns a 502 in about 30ms.

One bad response from the provider broke one location for a full day. CloudFront only ever saw and logged a `200`, so our CDN dashboards showed nothing wrong. The only signal was our own application error rate.

While it was happening, we cleared the bad entries with CloudFront invalidations. That works in an emergency, but we can't run on it. The real fix had to handle this kind of response without falling over.

## The Investigation

The clue was the 30ms latency. Real upstream failures are slow, because they involve timeouts, retries, or dropped connections. These were fast. A fast failure means the data arrived and something on our side rejected it, so the problem was in parsing, not fetching.

We dumped a failing body and it started with `1f 8b 08 00`. That's gzip, and there was no `Content-Encoding` anywhere in the response headers.

Two things made this worse than an ordinary vendor hiccup.

- **The CDN turned a passing error into a lasting one.** Without a cache, some requests would fail and the next would succeed. With a 24-hour cache key, one bad fetch meant 24 hours of failures for that location.
- **Our metrics couldn't name the problem.** The `bad_gateway` counter fired, but that counter catches every kind of upstream failure. Nothing in our metrics said "a vendor is sending us corrupt encodings," so nothing could alert on that specifically.

## The Fix

Every upstream request in our app goes through one class, `Api::AsyncHttp`, the async client from [the Falcon post](/falcon-async-performance/). Every source adapter and every CloudFront distribution uses it. So we could fix this in one place instead of once per adapter.

The fix is four lines in the branch that handles the body:

```ruby
body = response.read

if body.nil? && response.status == 204
  body = "{}"
elsif body.nil?
  raise Api::Weather::DataError.new(host, status: response.status)
elsif body.start_with?("\x1f\x8b".b)
  DatadogJob.perform_async("encoding_error", "host" => host)
  body = Zlib::GzipReader.new(StringIO.new(body)).read(MAX_INFLATED_BYTES + 1).to_s
  raise(Api::Weather::DataError.new(host, status: response.status)) if body.bytesize > MAX_INFLATED_BYTES
end

data = case parse
  when :json then JSON.parse(body, symbolize_names: true)
  when :raw  then body
else
  raise Api::Weather::NotImplementedError, parse
end
```

And one line in the rescue, so a corrupt or cut-off gzip stream still returns the same 502 it did before instead of an unhandled exception:

```ruby
rescue JSON::ParserError, Protocol::HTTP2::StreamError, Zlib::Error
  raise Api::Weather::DataError.new(host, status: response&.status)
```

Three details matter here.

- **Checking for `\x1f\x8b` is safe.** No JSON, XML, or text response we'd accept can start with those two bytes. If a body starts with them, it's gzip, whatever the headers say.
- **The `.b` matters.** `response.read` returns a string with `BINARY` encoding. Comparing it against a UTF-8 literal that contains `\x8b` raises `Encoding::CompatibilityError`. The `.b` makes the literal binary too, and without it the fix would be a new crash.
- **The vendor's mistake gets its own metric.** The `encoding_error` counter is tagged by host, so we can see the repair happening. Before the fix, the signal was a `bad_gateway` spike. After it, the signal is `encoding_error` going up while `bad_gateway` stays flat, which means the fallback is working. If both go up, the bodies are too damaged to repair, and that's a different problem.

Without the counter, the repair would have hidden the problem, and we'd never have had a reason to report it to the vendor.

## Why Every Cap-Free Design Fails

The question wasn't whether to inflate the body. It was how big to let it get.

Decompression can make a body much bigger than what arrived. We measured it on a payload built to be hostile: 474KB compressed inflated to 500MB, roughly 1026x, and took about 142ms of blocking C code.

That 142ms is the problem. We run Falcon, so every request in flight on a process shares one fiber reactor. Any blocking CPU work on the request path stalls every other request on that process. `task.with_timeout` can't interrupt C code, because a timeout only fires when the reactor gets a turn. A bad cache entry would repeat that stall on every request for up to 24 hours, and eventually the process would hit its memory limit and restart.

So we needed a limit. We looked at three ways to avoid one, and each fails for its own reason.

- **Trust the gzip ISIZE footer.** A gzip stream ends with a field that records the uncompressed size, so we could read it and refuse anything too large. But whoever wrote the stream wrote that field, so it's four bytes we can't trust. It's also stored mod 2³², so a 5GB stream reports 705MB.
- **Cap the compressed size instead.** Refuse bodies over, say, 1MB. But the compressed size tells you nothing about the output size. That's what a compression bomb is: a small input that inflates to something huge. Our 474KB test input became 500MB, and it would pass any input cap we could set on real traffic.
- **Inflate on a separate thread.** This one is tempting, and it fixes the smallest part of the problem. Moving the work off the reactor stops it stalling other requests, but the output is still 500MB. It also adds a thread pool to a codebase whose performance comes from "one reactor, no thread coordination."

The limit has to be on the output, and it has to apply while inflating. `GzipReader#read(MAX_INFLATED_BYTES + 1)` does that: it stops inflating one byte past the cap, so CPU and memory are both limited in one call, with no streaming loop. A body over the cap returns a 502 after about 8ms of work.

```ruby
MAX_INFLATED_BYTES = 10.megabytes
```

There's a trade-off. `read(len)` skips the CRC check that `Zlib.gunzip` does on the footer. Corrupt or cut-off bodies still return a 502, because either `GzipReader` raises or `JSON.parse` fails on the garbage. The only case where the skipped check would matter is `parse: :raw`, and nothing in production uses it.

The everyday cost is tiny. Inflating a realistic 721KB body takes 0.13ms, about 10x less than the `JSON.parse` that already runs on the same reactor for the same response.

We'd never seen a bomb-sized body from a vendor, and the uncompressed path has never had a size cap either. So the cap was more caution than need. We kept it because it cost two lines that explain themselves and nothing on the normal path, and we could delete it later without losing anything else. Two weeks later, that's what we did.

## Testing It

Four tests, one per path. Each builds a real gzip stream instead of mocking the inflation:

```ruby
test "gzip body without Content-Encoding is inflated and tracked" do
  body = Zlib.gzip({ forecast: [] }.to_json)
  response = build_response(status: 200, success: true, body: body)
  Async::HTTP::Internet.instance.stubs(:get).returns(response)
  DatadogJob.expects(:perform_async).with("encoding_error", { "host" => "example.com" })

  result = Api::AsyncHttp.get("https://example.com/forecast?geocode=1,2").wait

  assert_equal({ forecast: [] }, result.data)
end

test "gzip body inflating over the size limit raises DataError" do
  body = Zlib.gzip("0" * (Api::AsyncHttp::MAX_INFLATED_BYTES + 1.megabyte))
  response = build_response(status: 200, success: true, body: body)
  Async::HTTP::Internet.instance.stubs(:get).returns(response)
  DatadogJob.expects(:perform_async).with("encoding_error", { "host" => "example.com" })

  error = assert_raises(Api::Weather::DataError) do
    Api::AsyncHttp.get("https://example.com/forecast?geocode=1,2").wait
  end

  assert_equal 200, error.status
end
```

The other two cover `parse: :raw` and a corrupt stream (`"\x1f\x8b garbage"`), which still has to raise `DataError` with the original response status. The over-cap and corrupt tests both check that the `encoding_error` metric fires, because a repair that fails still started with a vendor sending us the wrong thing, and we want to count that.

## Results

- Gzip bodies with no header parsed normally instead of returning a 502, for every source and every distribution, because the fix sat below all of them. We checked by fetching the affected endpoint through the production distribution with the patch applied.
- Vendor encoding problems got their own metric, tagged by host, instead of hiding inside `bad_gateway`.
- Total diff: 55 lines added, one line changed, two files, landed 2026-07-16.

The ~12,700 daily errors stopped, but not because of this code. The invalidations had already cleared the bad entries, and the vendor fixed its side after the incident. In the 14 days the code was deployed, the `encoding_error` counter never fired. On 2026-07-31 a follow-up commit removed the check, the cap, the counter, the `Zlib::Error` rescue, and the tests. The reasoning was that `bad_gateway` monitoring would show it if the problem came back, and the code is in git history if we need it again.

The rule survived. It went into the repo's agent instructions, the file every agent loads, the same day as the fix in a separate commit, and it's still there as of September 2026:

> Web traffic is served by Falcon (see `Procfile`), so all in-flight requests in a process share one fiber reactor. Blocking CPU on the request path (Zlib inflation, large `JSON.parse`, long-running C calls) stalls every concurrent request on that process, and `task.with_timeout` cannot interrupt blocking C code — timeouts only fire when the reactor runs. Keep request-path work small and bounded: cap input/output sizes and fail fast (502) instead of doing unbounded work, and benchmark new costs against what the path already pays (see #1605, `Api::AsyncHttp` gzip inflation).

## Lessons Learned

- **A 200 the CDN can cache isn't proof the body is good.** The status code is the vendor's own opinion of its output, and the CDN repeats it for the life of the cache key. Check the body with something the vendor didn't write.
- **Fix it in the one place every request passes through.** If each adapter has its own fix, the next adapter will forget it. If there's no such place, that's the first thing to fix.
- **Limit decompression by output size, while inflating.** Every size field in the format was written by whoever sent the payload, and on a shared reactor a timeout can't interrupt C code.
- **Give the fallback its own metric.** Without one, the repair hides the problem, and the vendor never hears about it.
