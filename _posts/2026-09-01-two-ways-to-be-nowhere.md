---
layout: post
title: "Two Ways to Be Nowhere"
date: 2026-09-01 17:00:00 -0600
summary: "Our weather API was answering 502 for points in the open ocean, so clients kept retrying. We replaced it with two kinds of 422, each with a machine-readable reason: one for a source that doesn't cover the point, one for a point no source can resolve."
tags: [api-design, ruby, rails, errors]
---

## The Problem

In July 2026, about 500 requests a day to the [Hello Weather](https://helloweather.com) API were failing with 502 Bad Gateway. Nearly all of them came from a handful of coordinates in the open ocean. One saved location in the Norwegian Sea, between Iceland and the Faroe Islands, accounted for 57 captured events on its own. A widget was re-fetching that point about every 15 minutes, getting a 502, and trying again next cycle.

Nothing upstream was broken. Every timezone provider in the chain answered normally, and they all said the same thing: there's no timezone here. The API had no way to say that, so it said "source data error" instead, and a 502 is the status a well-behaved client retries.

The second problem came from the other direction. We were about to add regional sources, national weather services that only cover one country. Every plan for one of those had left the same question open. What happens when a user picks a regional source and then looks at a city outside its coverage? Serve an empty forecast, or reject? Each plan had pushed the question to the next one.

Both problems have the same shape. The request is well-formed and authenticated, and the answer is "nowhere", but for two different reasons. In one case a different source would work. In the other, nothing would. The API needed to say which.

## The Solution

Three changes landed over two days, July 24 to 26, 2026, and they build on each other:

- Garbage coordinates are rejected with 400 during validation, before any lookup runs.
- A source can declare the regions it covers, and a request outside them is rejected with 422 before any upstream call.
- When every timezone provider agrees a point has no timezone, the request is rejected with a different 422, at the point where the chain runs out.

The two 422s share a status code and differ in one field, `reason`. That one field does most of the work.

### Validation, not a fetch precondition

The first version of the region check lived in `preload`, next to the upstream fetches, because that's where the source class was already in hand. We moved it out on July 24, before shipping, for one reason: whether a source covers a point is knowable from the request alone. The source key and the coordinates are all it needs. So it's request validation, the same kind of check as "is `lat` between -90 and 90", and it belongs in the same place.

Running the check during the fetch has a cost that's easy to miss. A rejected request that has already opened an upstream connection shows up in that source's error rate, and the source did nothing wrong. Rejecting during validation means we never make the upstream call, so the source-health metrics only see requests the source had a chance to answer.

```ruby
class Api::Sources::Base
  # nil means global. A regional source returns ISO alpha-2 and/or UN M49 codes.
  def self.supported_source_regions
    nil
  end
end

class Api::Sources::NationalService < Api::Sources::Base
  def self.supported_source_regions
    %w(US)
  end
end

class Api::Weather
  include ActiveModel::Validations

  validates_presence_of :lat, :lon, :source
  validates_inclusion_of :lat, in: -90..90, allow_nil: true
  validates_inclusion_of :lon, in: -180..180, allow_nil: true

  validate do
    errors.add(:region) if region_unsupported?
  end

  private

  def region_unsupported?
    return if errors.has_key?(:lat) || errors.has_key?(:lon)

    regions = @source_class.supported_source_regions
    return if regions.nil?

    !geo_lookup.in_regions?(regions)
  end

  def geo_lookup
    @_geo_lookup ||= Api::Sources::Secondary::GeoLookup.new(lat: lat, lon: lon)
  end
end
```

The excerpt is trimmed to the region check. The real validate block also checks units, settings, and backfill flags. The first line of `region_unsupported?` matters most. The range checks and the region check run in one `valid?` pass, and the region check bails out if the coordinates already failed. So the geographic lookup, which is real work, never runs on a latitude of 91. Garbage stays a 400 and never reaches the lookup. The lookup only runs for sources that declare regions, so the many global sources skip it entirely.

The declaration joins two the codebase already had: supported units and supported languages. Those two are soft. An unsupported language falls back to English. This one is hard, because falling back for a region would mean serving another source's data under the name the user chose, and we don't do that. See [Multi-Source API Adapter Pattern in Ruby](/multi-source-api-adapter-pattern/) for how the sources are built.

### A reason beside the status

The controller already looked at the error key when validation failed: an `:account` error became a 401. The region error joins that branch and adds a `reason` string to the body.

```ruby
class Api::ApiController < ApplicationController
  rescue_from ActiveModel::ValidationError do
    if @weather.errors.has_key?(:account)
      render json: { code: 401, error: "Unauthorized" }, status: :unauthorized
    elsif @weather.errors.has_key?(:region)
      render json: {
        code: 422,
        error: "Unprocessable Entity: #{@weather.source_key} does not cover this location",
        reason: "unsupported_location",
      }, status: :unprocessable_entity
    else
      render json: { code: 400, error: "Bad Request: invalid #{@weather.validation_errors}" }, status: :bad_request
    end
  end

  rescue_from Api::Weather::UnresolvableLocationError do
    render json: {
      code: 422,
      error: "Unprocessable Entity: this location cannot be resolved",
      reason: "unresolvable_location",
    }, status: :unprocessable_entity
  end
end
```

The order is set once, in this one rescue: 401, then 422, then the generic 400. A request that is both unauthenticated and out of region gets the 401, because there's no point telling an anonymous caller about coverage.

We decided on the `reason` field before the first 422 shipped, and the argument was about clients that hadn't been written yet. A status code alone forces every client to treat all 422s the same way. A reason field lets a client match the reasons it knows and fall through to a generic error for the rest. The iOS client does that:

```swift
if statusCode == 422, errorReason(from: data) == "unsupported_location" {
    throw ServiceError.unsupportedLocation
}

guard statusCode == 200 else {
    throw ServiceError.responseFailure(statusCode: statusCode)
}
```

The `unsupportedLocation` case shows a notice that the chosen source doesn't provide weather for this location. Any other 422 shows the generic error. The client never changes the user's stored source on a 422. Their choice still works everywhere else, and the rejection is per-location. There's a separate flow for a source that has been retired everywhere, and the two must not be confused.

Two days later the fall-through got its first real use.

### Classify at the raise site

The mid-ocean 502s were the second kind. The timezone chain asks three providers in turn and raises when all three fail. The obvious change was to map that final raise to a 422 with the existing `unsupported_location` reason. That would have been wrong in two ways.

The first is the advice. The client's notice for `unsupported_location` tells the user this source doesn't cover this place, which implies another source would. For a point in the open ocean, nothing would. Every source shares the timezone chain, so switching changes nothing. So the second kind got its own reason, `unresolvable_location`. Shipped clients that had never heard of it fell through to the generic error, which was the right behavior, and the client can add specific copy whenever it wants.

A second review of the first version caught the other problem. That version aggregated on the chain's `TimeZoneError` class: if all three providers raised it, answer 422. But the adapters raised that class for three different reasons, and one of them was a zone name the Rails timezone table couldn't map. That happens after an IANA rename, and it had already happened once, when Europe/Kiev became Europe/Kyiv. Under the first version, an inhabited city would have gotten "this location cannot be resolved" the day after the next rename.

The fix was to stop guessing the cause three layers up and classify it where the data is in hand:

```ruby
class Api::Sources::Secondary::Base
  private

  # A blank zone means the point has no timezone. An unmappable zone is bad data.
  def find_timezone!(name)
    raise Api::Weather::UnresolvableLocationError, "#{key} timezone" if name.blank?

    find_timezone(name) || raise(Api::Weather::DataError, "#{key} timezone #{name}")
  end
end

class Api::Sources::Base
  RESCUABLE_TIMEZONE_LOOKUP_ERRORS = [
    Api::Weather::DataError,
    Api::Weather::RateLimitError,
    Api::Weather::TimeoutError,
    Api::Weather::TimeZoneError,
    Api::Weather::UnresolvableLocationError,
  ].freeze

  def timezone_data
    fetch_data do
      timezone_data_for("ProviderA") || timezone_data_for("ProviderB") || timezone_data_for("ProviderC") || raise_timezone_error!
    end
  end

  def timezone_data_for(source)
    adapter = "Api::Sources::Secondary::#{source}".constantize.new(lat: lat, lon: lon)
    adapter.preload
  rescue *RESCUABLE_TIMEZONE_LOOKUP_ERRORS => exception
    timezone_failures[adapter.key] = exception
    nil
  end

  def raise_timezone_error!
    message = timezone_failures.map { |key, val| "#{key}: #{val.class.name.demodulize}" }.join(", ")

    if timezone_failures.values.all?(Api::Weather::UnresolvableLocationError)
      raise Api::Weather::UnresolvableLocationError, message
    else
      raise Api::Weather::TimeZoneError, message
    end
  end
end
```

Provider names are placeholders and the metrics call is removed. The `all?` in `raise_timezone_error!` is the guard that matters. The 422 is raised only when every provider failure is an unresolvable-location error. If one provider timed out and two said "no timezone", the request stays a 502, because a provider being down tells us nothing about the location. Six near-identical adapter methods became one-liners calling `find_timezone!`. The failure metric also gained the ocean-versus-provider-trouble split, because the exception class now carries it.

The same day, we moved one more case onto the same error. One vendor's location search returns HTTP 200 with a literal JSON `null` for coordinates it has no entry for. Its adapter had been raising a data error, which is a 502, for what was an honest no-results answer. That's the second, quieter way to be nowhere: not "no timezone exists" but "this provider has no location within reach". It takes the same 422 because the client's correct behavior is the same.

## Results

- The repository records an expectation, not a measurement: roughly 500 requests a day would move from 502 to 422 once the timezone change deployed on July 26, 2026. Nobody wrote down an after-count, and the retry loops depend on client behavior we don't control.
- This is a public API change. Third-party customers with retry-on-5xx logic now see 400 for out-of-range coordinates and 422 for unresolvable points where they used to see 502. The PRs flagged it as deliberate, because the retry was always futile.
- The error reporter no longer captures either 422. They're a handled client-input outcome, not a failure, and they land on the existing request-status metric under a value the first 422 had already introduced.
- As of September 1, 2026, nothing on the main branch uses the region declaration yet. Each regional source adopts it when it merges, and the first one, a US national service, is at the head of the merge queue awaiting review. The check shipped ahead of its first user so the queue could depend on it.
- Accepted: a point that passes the region check can still fall outside the provider's real grid at a coastline. That fails fast as a 502, with no rescue-and-return, rather than serving empty data for a request we know is invalid.

The related failure, a coordinate rounded across a timezone line, is the one this design can't catch. A valid-but-wrong zone looks like success at every layer, as [The Rounding That Moved a City](/the-rounding-that-moved-a-city/) describes.

## Lessons Learned

- If a rejection is knowable from the request alone, it's validation. Reject before the first upstream call so it never lands in a provider's health metrics.
- Don't serve an empty-but-valid payload for "not covered". It looks the same as a working source with nothing to say, and it hides the problem from the client that could act on it.
- Put a machine-readable reason beside the status code from the first kind onward. Clients that predate a new reason fall through to a generic error instead of showing the wrong advice.
- Classify a failure where the data is in hand, not by aggregating exception classes three layers up. The aggregate would have called an inhabited city nowhere after the next timezone rename.
- Escalate to "this can never work" only when every provider agrees. One provider being down tells you nothing about the location.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

**Prompt 3:** "kick off the remainder in sub-agents, we have capacity"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent researched the web repository and proposed this post among a batch of candidates; a second agent verified the claims and wrote it. Sources: `plans/source-region-validation.md` (decided 2026-07-24), `app/models/api/weather.rb`, `app/models/api/sources/base.rb`, `app/models/api/sources/secondary/base.rb`, `app/controllers/api/api_controller.rb`, `test/models/api/region_validation_test.rb`, the iOS `ForecastService` and forecast view model, and the web repository's pull requests of July 24 to 26, 2026 (coordinate range validation, the pre-flight region gate, the unresolvable-location error, the null-location vendor case, and the source-error review that surfaced the ~500/day figure). Judgment calls: timezone providers and weather sources are unnamed, with placeholder names in the excerpts; the regional source is described as a US national service without naming it; the metrics and error-reporter calls are removed from the excerpts and the tools are unnamed; the Norwegian Sea coordinates are described by place rather than by number; the "no consumer on main" caveat is stated as of the writing date because the adopting pull requests had not merged; no after-count for the 502 volume is claimed because none is recorded.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 2, run after batch 1 (#68) merged. This post was written after the 2026-09-01 archive rewrite and had no earlier pass. Prose moved to "we" and contractions; "flavor" and "first cut" became "kind" and "first version" throughout; "adversarial review" became "a second review"; "for free" and "paid for itself" were cut; the "one to notice" line was reworded so "notice" names only the client's message; and the summary was rewritten in the same register. Code, headings, numbers, quoted text, and links are unchanged. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"
