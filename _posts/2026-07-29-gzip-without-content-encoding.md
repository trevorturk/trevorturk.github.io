---
layout: post
title: "Gzip Without a Content-Encoding Header"
date: 2026-07-29 08:20:00 -0600
summary: "A vendor returned raw gzip bytes in a 200 with no Content-Encoding header, CloudFront cached it for 24 hours, and 12.7k errors a day were invisible in CDN logs. The fix sniffs magic bytes at one transport chokepoint, with a hard cap on inflation because Falcon shares one reactor."
tags: [ruby, falcon, cloudfront, debugging, http]
---

## The Problem

On 2026-07-15, [Hello Weather](https://helloweather.com) started throwing about 12,700 `bad_gateway` errors a day from a single upstream data provider. Users saw them. Our CDN logs did not.

The provider was intermittently returning an HTTP 200 whose body was raw gzip bytes, with no `Content-Encoding: gzip` header. To every layer of the stack this was a good response: status 200, Content-Length present, body non-empty. It just started with `\x1f\x8b` instead of `{`.

Our HTTP client did what it always does: read the body, `JSON.parse` it, and raise `DataError` (a 502) when the parse failed. That part worked. The problem was the layer above it.

We use [CloudFront as an infinite cache](/cloudfront-cdn-architecture/) in front of every upstream provider, with cache keys built from a `Cache-Expires` boundary time rather than a duration. For the affected endpoint, that boundary rotated only at UTC midnight. So:

1. One origin fetch returns a mislabeled gzip body.
2. CloudFront sees a 200 and caches it.
3. Every subsequent request for that location, for up to 24 hours, gets served the poisoned entry from the edge.
4. Each one fails `JSON.parse` and 502s in ~30ms.

One bad origin response poisoned one location for a full day. CloudFront only ever saw and logged a `200`, so none of our CDN dashboards showed anything wrong. The only signal was our own application error rate.

We mitigated live with CloudFront invalidations, a fine emergency lever and a terrible plan. The real fix had to make this class of failure degrade gracefully.

## The Investigation

The tell was the 30ms latency. Real upstream failures are slow: timeouts, retries, connection resets. These were fast. A fast failure means the data arrived and something local rejected it, which pointed at parsing rather than fetching.

Dumping a failing body confirmed it: `1f 8b 08 00`. Gzip, with no `Content-Encoding` anywhere in the response headers.

Two things made this worse than a normal vendor hiccup.

- **The CDN turned a transient error into a persistent one.** Without a cache, some requests would fail and the next would succeed. With a 24-hour cache key, one bad fetch became 24 hours of guaranteed failure for that location. A CDN amplifies whatever the origin says, including lies.
- **The observability was structurally wrong.** The `bad_gateway` counter fired correctly, but `bad_gateway` is a symptom bucket. It catches timeouts, malformed payloads, 5xx passthroughs, and now this. Nothing in our metrics said "a vendor is sending us corrupt encodings," so nothing could alert on it specifically.

## The Fix

All upstream HTTP in our app goes through one class, `Api::AsyncHttp`, the async client described in [the Falcon post](/falcon-async-performance/). Every source adapter, every CloudFront distribution, every endpoint. That made the fix a single-site change instead of a per-adapter one.

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

- **`\x1f\x8b` is a safe sniff.** Those two bytes cannot begin any legal JSON, XML, or text response we would accept. If a body starts with the gzip magic number, it is gzip, whatever the headers claim.
- **The `.b` matters.** `response.read` returns a `BINARY`-encoded string. Comparing it against a UTF-8 literal containing `\x8b` raises `Encoding::CompatibilityError`. Forcing the literal to binary is the difference between a fix and a new crash.
- **Vendor misbehavior gets its own metric.** The `encoding_error` counter, tagged by host, makes the repair visible. Before the fix, the signal was a `bad_gateway` spike. After it, the signal is `encoding_error` with *no* corresponding `bad_gateway` spike, which is the fallback working. If both stay elevated, the bodies are corrupt beyond repair, a different problem needing a different response.

A repair with no telemetry converts a loud problem into a quiet one, and quiet problems never get fixed upstream. The fallback should hold the line and file a report.

## Why Every Cap-Free Design Fails

The question was not whether to inflate the body but how much inflation to allow.

Decompression is an amplifier. We measured it on a deliberately hostile payload: 474KB compressed inflated to 500MB, roughly 1026x, and took about 142ms of blocking C.

That 142ms is the whole problem. We run Falcon, so all in-flight requests in a process share one fiber reactor. Blocking CPU on the request path stalls every concurrent request on that process, and `task.with_timeout` cannot interrupt blocking C code, because timeouts only fire when the reactor gets to run. A poisoned CDN entry would replay that stall on every request for up to 24 hours. That path ends in memory-limit restarts.

So we needed a bound. Every design that avoids one fails for a specific reason.

- **Trust the gzip ISIZE footer.** Gzip's trailer includes an uncompressed-size field, so you could read it and refuse anything too large. But ISIZE is attacker-controlled, just four bytes at the end of a stream the vendor wrote, and it is stored mod 2³², so a 5GB stream reports 705MB. A field you cannot trust is not a limit.
- **Cap the compressed size instead.** Refuse bodies over, say, 1MB. But the compressed size tells you nothing about the output size; that is the definition of a compression bomb. A 474KB input producing 500MB of output passes any input-side cap you would set on real traffic.
- **Offload the inflation to a thread.** This is the tempting one, and it fixes the smallest part of the problem. Moving the work off the reactor stops it stalling other fibers, but 500MB is still 500MB, and it adds thread-pool machinery to a codebase whose whole performance story is "one reactor, no thread coordination." It trades a real bound for a partial one plus complexity.

The bound has to be on the output, applied during inflation. That is what `GzipReader#read(MAX_INFLATED_BYTES + 1)` does: it stops inflating at the cap, so CPU and memory are both bounded in one pass with no streaming loop. An over-cap body 502s after a bounded probe of roughly 8ms.

```ruby
MAX_INFLATED_BYTES = 10.megabytes
```

There is a known trade-off. `read(len)` skips the CRC/footer verification that `Zlib.gunzip` performs. In practice, corrupt or truncated bodies still 502, because either `GzipReader` raises or `JSON.parse` chokes on the garbage. The only reachable difference is on `parse: :raw`, which has no production callers.

Steady-state cost is negligible. Inflating a realistic 721KB body takes 0.13ms, about 10x cheaper than the `JSON.parse` already running on the same reactor for that same response.

Bomb-sized bodies had never been observed here, and the uncompressed path has never had a size cap either. The cap leaned protective. It stayed because it cost two self-explanatory lines and nothing on the happy path, and deleting it later would lose nothing else. Two weeks later, that is what happened.

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

The other two cover `parse: :raw` and a corrupt stream (`"\x1f\x8b garbage"`), which must still raise `DataError` carrying the original response status. The over-cap and corrupt tests both assert the `encoding_error` metric fires, because a repair that fails is still vendor misbehavior worth counting.

## Results

- Mislabeled gzip bodies parsed normally instead of 502ing, across every source and every distribution, because the fix sat below all of them. It was verified by fetching the affected endpoint through the production distribution with the patch applied.
- Vendor encoding problems got a dedicated, host-tagged metric instead of hiding inside a generic `bad_gateway` bucket.
- Total diff: 55 lines added, one line changed, two files, landed 2026-07-16.

The ~12,700 daily errors stopped, but not because of this code. The live invalidations had already cleared the poisoned entries, and the vendor fixed its origin after the incident. In the 14 days the branch was deployed, the `encoding_error` counter fired zero times. On 2026-07-31 a follow-up commit removed the sniff, the cap, the counter, the `Zlib::Error` rescue, and the tests, on the grounds that `bad_gateway` monitoring would surface a recurrence and the commit history is the revive path.

What survived is the rule. It went into the repo's always-loaded agent instructions the same day as the fix, in a separate commit, and it is still there as of September 2026:

> Web traffic is served by Falcon (see `Procfile`), so all in-flight requests in a process share one fiber reactor. Blocking CPU on the request path (Zlib inflation, large `JSON.parse`, long-running C calls) stalls every concurrent request on that process, and `task.with_timeout` cannot interrupt blocking C code — timeouts only fire when the reactor runs. Keep request-path work small and bounded: cap input/output sizes and fail fast (502) instead of doing unbounded work, and benchmark new costs against what the path already pays (see #1605, `Api::AsyncHttp` gzip inflation).

## Lessons Learned

- **A cacheable 200 is not a valid 200.** A status code is the origin's opinion of its own output, and a CDN amplifies it for the life of the cache key. Validate with something the origin did not write.
- **Repair at the one chokepoint every call site passes through.** N adapters means N implementations and a guarantee that adapter N+1 forgets. If there is no chokepoint, that is the real finding.
- **Bound decompression on the output, during inflation.** Every size field in the format is written by the party sending the payload, and on a shared reactor a timeout cannot preempt C.
- **Know what a working fallback looks like on a dashboard.** A repair without its own metric is a silent one, and the vendor never hears about a silent one.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and code, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. This was a light pass on a post that already fit the standard: the title dropped to five words, the bolded paragraph runs in Investigation, Fix, and the cap section became bullet lists, Results lost one bullet that restated the fix, and Lessons Learned was cut to four short rules. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The code, tests, and every number matched the original commit; the agent-instructions blockquote was restored to the rule's verbatim wording, which the post had trimmed; and Results was rewritten to say what the source shows: the errors stopped because the vendor fixed its origin, the `encoding_error` counter never fired in production, and a 2026-07-31 commit removed the gzip handling entirely, so the "eliminated with no invalidations" bullet and the present-tense "it stays" sentence were corrected.
