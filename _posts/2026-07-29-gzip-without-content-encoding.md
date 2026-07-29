---
layout: post
title: "The Vendor Sent Gzip Without a Content-Encoding Header — and the CDN Cached It"
date: 2026-07-29 08:20:00 -0600
summary: "A vendor returned raw gzip bytes in a 200 with no Content-Encoding header, CloudFront cached it for 24 hours, and 12.7k errors a day were invisible in CDN logs. The fix sniffs magic bytes at one transport chokepoint — with a hard cap on inflation, because Falcon shares one reactor."
tags: [ruby, falcon, cloudfront, debugging, http]
---

## The Problem

On 2026-07-15, [Hello Weather](https://helloweather.com) started throwing about 12,700 `bad_gateway` errors a day from a single upstream data provider. The errors were real, users saw them, and they were completely invisible in our CDN logs.

The cause: the provider intermittently returned an HTTP 200 whose body was raw gzip bytes — with no `Content-Encoding: gzip` header. As far as every layer of the stack was concerned, this was a perfectly good response. Status 200. Content-Length present. Body non-empty. It just happened to start with `\x1f\x8b` instead of `{`.

Our HTTP client did what it always does: read the body, `JSON.parse` it, and raise `DataError` (a 502) when the parse failed. That part worked fine. The problem was the layer above it.

We use [CloudFront as an infinite cache](/cloudfront-cdn-architecture/) in front of every upstream provider, with cache keys built from a `Cache-Expires` boundary time rather than a duration. For the affected endpoint, that boundary rotated only at UTC midnight. So:

1. One origin fetch returns a mislabeled gzip body.
2. CloudFront sees a 200 and caches it.
3. Every subsequent request for that location, for up to 24 hours, gets served the poisoned entry from the edge.
4. Each one fails `JSON.parse` and 502s in ~30ms.

A single bad response from the origin poisoned one location for a full day. And because CloudFront only ever saw and logged a `200`, none of our CDN dashboards showed anything wrong at all. The only signal was our own application error rate.

We mitigated live with CloudFront invalidations, which is a fine emergency lever and a terrible plan. The real fix had to make this class of failure degrade gracefully.

## The Investigation

The tell was the 30ms latency. Real upstream failures are slow — timeouts, retries, connection resets. These were fast. Fast failures mean the data arrived and something local rejected it, which pointed at parsing rather than fetching.

Dumping a failing body confirmed it immediately: `1f 8b 08 00`. Gzip. With response headers containing no `Content-Encoding` at all.

Two things made this worse than a normal vendor hiccup:

**The CDN turned a transient error into a persistent one.** Without a cache, this would have been an intermittent blip — some requests fail, the next one succeeds. With a 24-hour cache key, one bad fetch became 24 hours of guaranteed failure for that location. A CDN is an amplifier, and it amplifies whatever the origin says, including lies.

**The observability was structurally wrong.** We had a `bad_gateway` counter, and it was firing correctly. But `bad_gateway` is a symptom bucket — it catches timeouts, malformed payloads, 5xx passthroughs, and now this. Nothing in our metrics said "a vendor is sending us corrupt encodings," so nothing could alert on it specifically.

## The Fix

All upstream HTTP in our app goes through one class, `Api::AsyncHttp` — the async client described in [the Falcon post](/falcon-async-performance/). Every source adapter, every CloudFront distribution, every endpoint. That made the fix a single-site change instead of a per-adapter one.

The core of it is four lines in the body-handling branch:

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

And one line in the rescue, so a genuinely corrupt or truncated gzip stream still produces the same 502 it did before rather than an unhandled exception:

```ruby
rescue JSON::ParserError, Protocol::HTTP2::StreamError, Zlib::Error
  raise Api::Weather::DataError.new(host, status: response&.status)
```

Three details are worth pulling out.

**`\x1f\x8b` is a safe sniff.** Those two bytes cannot begin any legal JSON, XML, or text response we'd ever accept. There is no false-positive path — if a body starts with the gzip magic number, it is gzip, regardless of what the headers claim.

**The `.b` is load-bearing.** `response.read` returns a `BINARY`-encoded string. Comparing it against a UTF-8 literal containing `\x8b` raises `Encoding::CompatibilityError`. Forcing the literal to binary is the difference between a fix and a new crash.

**Vendor misbehavior gets its own metric.** The `encoding_error` counter, tagged by host, means the repair is visible rather than silent. Before the fix, the signal was a `bad_gateway` spike. After it, the signal is `encoding_error` with *no* corresponding `bad_gateway` spike — that's the fallback working. If both stay elevated, the bodies are corrupt beyond repair, which is a different problem needing a different response.

That last point matters more than it looks. A repair with no telemetry converts a loud problem into a quiet one, and quiet problems don't get fixed upstream. You want the fallback to hold the line *and* file a report.

## Why Every Cap-Free Design Fails

The interesting engineering question wasn't "should we inflate it." It was "how much are we willing to inflate."

Decompression is an amplifier. We measured it on a deliberately hostile payload: **474KB compressed inflated to 500MB — roughly 1026x — and took about 142ms of blocking C.**

That 142ms is the whole problem. We run Falcon, so all in-flight requests in a process share one fiber reactor. Blocking CPU on the request path stalls *every* concurrent request on that process, and `task.with_timeout` cannot interrupt blocking C code — timeouts only fire when the reactor gets to run. Worse, a poisoned CDN entry would replay that stall on every request for up to 24 hours. That path ends in memory-limit restarts.

So we needed a bound. Every design that avoids one fails for a specific reason:

**Trust the gzip ISIZE footer.** Gzip's trailer includes an uncompressed-size field, so you could read it and refuse anything too large. Except ISIZE is attacker-controlled — it's just four bytes at the end of a stream the vendor wrote — and it's stored mod 2³², so a 5GB stream reports 705MB. A field you can't trust isn't a limit.

**Cap the compressed size instead.** Refuse bodies over, say, 1MB. But the compressed size tells you nothing about the output size; that's the definition of a compression bomb. A 474KB input producing 500MB of output passes any input-side cap you'd be willing to set on real traffic.

**Offload the inflation to a thread.** This is the tempting one, and it fixes the smallest part of the problem. Moving the work off the reactor stops it from stalling other fibers, but it doesn't bound memory — 500MB is still 500MB — and it adds thread-pool machinery to a codebase whose entire performance story is "one reactor, no thread coordination." It trades a real bound for a partial one plus complexity.

The bound has to be on the *output*, applied *during* inflation. Which is what `GzipReader#read(MAX_INFLATED_BYTES + 1)` does: it stops inflating at the cap. CPU and memory are both bounded in one pass, no streaming loop required. An over-cap body 502s after a bounded probe of roughly 8ms.

```ruby
MAX_INFLATED_BYTES = 10.megabytes
```

There's a known trade-off. `read(len)` skips the CRC/footer verification that `Zlib.gunzip` performs. In practice, corrupt or truncated bodies still 502 — either `GzipReader` raises or `JSON.parse` chokes on the garbage — so the only reachable difference is on `parse: :raw`, which has no production callers.

Steady-state cost is negligible: inflating a realistic 721KB body takes 0.13ms, about 10x cheaper than the `JSON.parse` already running on the same reactor for that same response.

It's worth being honest that bomb-sized bodies have never been observed here, and the uncompressed path has never had a size cap either. The cap leans protective. It stays because it costs two self-explanatory lines and nothing on the happy path, and deleting it later loses nothing else.

## Testing It

Four tests, one per path, all constructing real gzip streams rather than mocking the inflation:

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

The other two cover `parse: :raw` and a corrupt stream (`"\x1f\x8b garbage"`), which must still raise `DataError` carrying the original response status. The over-cap and corrupt tests both assert the `encoding_error` metric fires — the repair failing is still vendor misbehavior worth counting.

## Results

- The failure mode is gone. Mislabeled gzip bodies now parse normally instead of 502ing, across every source and every distribution, because the fix sits below all of them.
- ~12,700 daily errors eliminated, with no CloudFront invalidations required.
- Vendor encoding problems now have a dedicated, host-tagged metric instead of hiding inside a generic `bad_gateway` bucket.
- The reactor is protected by a hard output bound rather than by hoping vendors send reasonable payloads.
- Total diff: 55 lines added, one line changed, two files.

We also promoted the underlying rule into the repo's always-loaded agent instructions, so future work inherits it:

> Web traffic is served by Falcon, so all in-flight requests in a process share one fiber reactor. Blocking CPU on the request path (Zlib inflation, large `JSON.parse`, long-running C calls) stalls every concurrent request on that process, and `task.with_timeout` cannot interrupt blocking C code — timeouts only fire when the reactor runs. Keep request-path work small and bounded: cap input/output sizes and fail fast instead of doing unbounded work, and benchmark new costs against what the path already pays.

## Lessons Learned

**A cacheable 200 is not a valid 200.** CDNs cache on status codes, and status codes are the origin's opinion of its own output. When the origin is wrong, the CDN faithfully amplifies the error — turning a transient blip into a persistent outage scoped to your cache key lifetime. The longer your TTL, the bigger the amplifier. If you cache aggressively, you need validation that doesn't come from the same source as the data.

**Sniff and repair at one chokepoint, not per adapter.** We have many source adapters and many distributions. Fixing this in each one would have meant N implementations, N tests, and a guarantee that adapter N+1 forgets. Because every upstream request funnels through one HTTP class, the fix was four lines that cover code we haven't written yet. The general form: when a defect can occur at any of N call sites, find the one place they all pass through — and if there isn't one, that's the real finding.

**Bound decompression cost when everything shares one thread.** Decompression is an amplifier with an attacker-controlled ratio, and every self-describing size field in the format is written by the same party sending the payload. On a shared-reactor server, unbounded blocking work isn't just slow for one request — it's a stall for every concurrent request in the process, and your timeout mechanism can't save you because it can't preempt C. Bound the output during inflation, fail fast, and measure the new cost against what the request path already pays.

**Make the repair visible.** The instinct after a fix like this is relief that the errors stopped. But a silent fallback means the vendor never hears about it and you never know how often it happens. Give repairs their own metric, and make sure you know what "the fallback is working" looks like on a dashboard versus "the fallback is failing."

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and code, then reviewed before publishing.
