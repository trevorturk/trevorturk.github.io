---
layout: post
title: "Gzip Without a Content-Encoding Header"
date: 2026-07-29 08:20:00 -0600
summary: "A vendor sent gzip bytes in a 200 with no Content-Encoding header, CloudFront cached it for 24 hours, and 12.7k errors a day never showed up in the CDN logs. We check for the gzip magic bytes in the one class every request goes through, and cap the inflated size because Falcon runs everything on one reactor."
tags: [ruby, falcon, cloudfront, debugging, http]
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

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and code, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. This was a light pass on a post that already fit the standard: the title dropped to five words, the bolded paragraph runs in Investigation, Fix, and the cap section became bullet lists, Results lost one bullet that restated the fix, and Lessons Learned was cut to four short rules. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The code, tests, and every number matched the original commit; the agent-instructions blockquote was restored to the rule's verbatim wording, which the post had trimmed; and Results was rewritten to say what the source shows: the errors stopped because the vendor fixed its origin, the `encoding_error` counter never fired in production, and a 2026-07-31 commit removed the gzip handling entirely, so the "eliminated with no invalidations" bullet and the present-tense "it stays" sentence were corrected.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 3, run after batch 2 (#69) merged. One Claude Fable 5.1 agent wrote an ELI5 of the post and redrafted the prose from it: "we" and contractions throughout, "poisoned" and "mislabeled" replaced with plain "bad" and "with no header", the gzip magic bytes and compression bombs defined at first use, "symptom bucket" and "cap-free" said in ordinary words, and the quotable closers on the CDN bullet, the ISIZE bullet, the telemetry paragraph, and the Lessons Learned cut in favor of the fact each one was restating. Judgment calls: "surface a recurrence" in Results became "show it if the problem came back" to retire the verb; the invented quote "a vendor is sending us corrupt encodings" and the "one reactor, no thread coordination" phrase were kept verbatim as quoted text. Code blocks, headings, dates, numbers, links, and the blockquote are unchanged, and no facts were added. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"

**Prompt 10:** "usage is fine, please continue -- one more thing -- at the end (or perhaps with future batches?) I'd like to change the "How This Post Was Made" sections in all posts to not have the prompt in the post itself, rather, the prompts should be moved into PR body if editable, or comments, then the "How This Post Was Made" can have the last edit date and a link to the Pull Requests / Prompts -- then there's less cruft at the end for readers that just want to copy paste a post into their agent -- wdyt?"
