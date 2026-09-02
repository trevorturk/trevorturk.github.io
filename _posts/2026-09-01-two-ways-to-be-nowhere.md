---
layout: post
title: "Two Ways to Be Nowhere"
date: 2026-09-01 17:00:00 -0600
summary: "A weather API was answering 502 for open-ocean coordinates, and clients kept retrying. The fix was two flavors of 422 with a machine-readable reason: one for a source that does not cover the point, one for a point no source can resolve."
tags: [api-design, ruby, rails, errors]
---

## The Problem

In July 2026, about 500 requests a day to the [Hello Weather](https://helloweather.com) API were failing with 502 Bad Gateway, and nearly all of them came from a handful of coordinates in the open ocean. One saved location in the Norwegian Sea, between Iceland and the Faroe Islands, accounted for 57 captured events on its own: a widget re-fetching the same point about every 15 minutes, getting a 502, and trying again next cycle.

Nothing upstream was broken. Every timezone provider in the lookup chain answered normally and said the same thing: there is no timezone here. The API had no way to say that, so it said "source data error" instead, and a 502 is exactly the status a well-behaved client retries.

A second problem was arriving from the other direction. The source list was about to grow with regional providers, national weather services that only cover one country. Every plan for one of those sources had left the same line open: what happens when a user picks a regional source and then looks at a city outside its coverage? Serve an empty forecast, or reject? Each plan had deferred the question to the next one.

Both problems are the same shape. The request is well-formed and authenticated, and the answer is "nowhere", but for two different reasons. In one case a different source would work. In the other, nothing would. The API needed to say which.

## The Solution

Three changes landed over two days, July 24 to 26, 2026, and they stack:

- Garbage coordinates are rejected with 400 during validation, before any lookup runs.
- A source can declare the regions it covers, and a request outside them is rejected with 422 before any upstream call.
- When every timezone provider agrees a point has no timezone, the request is rejected with a different 422, at the point where the chain runs out.

The two 422s share a status code and differ in one field, `reason`. That field is the part of the design that does the most work.

### Validation, not a fetch precondition

The first version of the region gate lived in `preload`, next to the upstream fetches, because that is where the source class was already in hand. It moved out on July 24, before shipping, on one argument: whether a source covers a point is knowable from the request alone. The source key and the coordinates are all it needs. That makes it request validation, the same kind of check as "is `lat` between -90 and 90", and it belongs in the same place.

The alternative, running the check during the fetch, has a cost that is easy to miss. A rejected request that has already opened an upstream connection shows up in that source's error rate, and the source did nothing wrong. Rejecting during validation means no upstream HTTP is made, and the source-health metrics only ever see requests the source had a chance to answer.

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

The excerpt is trimmed to the region check; the real validate block also checks units, settings, and backfill flags. The first line of `region_unsupported?` is the one to notice. The range validation and the region validation run in a single `valid?` pass, and the region check bails out if the coordinates already failed. That ordering keeps the geographic lookup, which is a real computation, from ever running on a latitude of 91. Garbage stays a 400 and never reaches the lookup. The lookup itself only runs for sources that declare regions, so the many global sources pay nothing.

The declaration is the third member of a family the codebase already had, alongside supported units and supported languages. Those two are soft: an unsupported language falls back to English. This one is hard. Falling back for a region would mean serving another source's data under the name the user chose, and the API does not do that. See [Multi-Source API Adapter Pattern in Ruby](/multi-source-api-adapter-pattern/) for how the source abstraction is built.

### A reason beside the status

The controller already branched a validation failure on the error key, with an `:account` error mapping to 401. The region error joins that branch and adds a `reason` string to the body.

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

The ordering is stated once, in this one rescue: 401, then 422, then the generic 400. A request that is both unauthenticated and out of region gets the 401, because there is no point telling an anonymous caller about coverage.

The `reason` field was decided before the first 422 shipped, and the reasoning was about clients that had not been written yet. A status code alone forces every client to treat all 422s the same way. A discriminator lets a client match the reasons it knows and fall through to a generic error for the rest. The iOS client does exactly that:

```swift
if statusCode == 422, errorReason(from: data) == "unsupported_location" {
    throw ServiceError.unsupportedLocation
}

guard statusCode == 200 else {
    throw ServiceError.responseFailure(statusCode: statusCode)
}
```

The `unsupportedLocation` case shows an informational notice that the chosen source does not provide weather for this location. Any other 422 shows the generic error. The client never changes the user's stored source on a 422. Their choice is still valid everywhere else, and the rejection is per-location, so it must not be confused with the separate flow for a source that has been retired globally.

Two days later that fall-through paid for itself.

### Classify at the raise site

The mid-ocean 502s were the second flavor. The timezone chain asks three providers in turn and raises when all three fail. The obvious change was to map that final raise to a 422 with the existing `unsupported_location` reason. That would have been wrong twice.

The first problem is the advice. The client's notice for `unsupported_location` tells the user the source does not cover this place, which implies another source would. For a point in the open ocean, nothing would: the timezone chain is shared by every source, so switching changes nothing. That is why the second flavor got its own reason, `unresolvable_location`. Shipped clients that had never heard of it fell through to the generic error, which was the correct behavior, and the client can add specific copy whenever it wants.

The second problem was caught in an adversarial review of the first cut. That cut aggregated on the chain's `TimeZoneError` class: if all three providers raised it, answer 422. But the adapters raised that class for three different reasons, and one of them was a zone name that the Rails timezone table could not map. That happens after an IANA rename, and it had already happened once, when Europe/Kiev became Europe/Kyiv. Under the first cut, an inhabited city would have been answered "this location cannot be resolved" the day after the next rename.

The fix was to stop inferring the cause three layers up and classify it where the data is in hand:

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

Provider names are placeholders and the metrics call is removed. The `all?` in `raise_timezone_error!` is the guard that matters. The 422 is raised only when every provider failure is an unresolvable-location error. If one provider timed out and two said "no timezone", the request stays a 502, because a provider being down is not evidence about the location. Six near-identical adapter methods collapsed to one-liners calling `find_timezone!`, and the failure-classification metric gained the ocean-versus-provider-trouble split for free, since the exception class now carries it.

The same day, a vendor whose location search returns HTTP 200 with a literal JSON `null` for coordinates it has no entry for was moved onto the same error. Its adapter had been raising a data error, which is a 502, for what was an honest no-results answer. That is the second, quieter way to be nowhere: not "no timezone exists" but "this provider has no location within reach", and it takes the same 422 because the client's correct behavior is the same.

## Results

- The repository records the expectation that roughly 500 requests a day would move out of the 502 series and into 422 once the timezone change deployed on July 26, 2026. No after-count was written down, and the retry loops depend on client behavior the server does not control.
- The change is a public contract change. Third-party API customers with retry-on-5xx logic see 400 for out-of-range coordinates and 422 for unresolvable points where they used to see 502. The PRs flagged it as deliberate: the retry was always futile.
- Both 422s stopped being captured by the error reporter. They are a handled client-input outcome, not a failure, and they land on the existing request-status metric under a value the first 422 had already introduced.
- As of September 1, 2026, the region declaration has no consumer on the main branch. Each regional source adopts it at its own merge, and the first one, a US national service, is at the head of the merge queue awaiting review. The gate shipped ahead of its first user so the queue could depend on it.
- Accepted: a point that passes the region gate can still fall outside the provider's real grid at a coastline. That fails fast as a 502, with no rescue-and-return, rather than serving empty data for a known-invalid request.

The related failure, a coordinate rounded across a timezone line, is the one this design cannot catch: a valid-but-wrong zone looks like success at every layer, as [The Rounding That Moved a City](/the-rounding-that-moved-a-city/) describes.

## Lessons Learned

- If a rejection is knowable from the request alone, it is validation. Reject before the first upstream call so it never lands in a provider's health metrics.
- Never serve an empty-but-valid payload for "not covered". It is indistinguishable from a working source with nothing to say, and it hides the problem from the client that could act on it.
- Put a machine-readable reason beside the status code from the first flavor onward. Clients that predate a new reason fall through to a generic error instead of showing the wrong advice.
- Classify a failure where the data is in hand, not by aggregating exception classes three layers up. The aggregate would have called an inhabited city nowhere after the next timezone rename.
- Escalate to "this can never work" only when every provider agrees. One provider being down is not evidence about the location.

---

## How This Post Was Made

**Prompt 1:** "kick off a post in a PR for that, then let's kick off another more comprehensive round of digging into the web and ios code looking for more good stuff to post. to start I'd like to find more stuff I can share for falcon/async/async-http users. the author of async is asking if I've done any writing about out cost savings, so this is a great start, but I'd love to find more to share."

**Prompt 2:** "kick off posts for: 2, 3, 4, 7, 11, 12, 17, 22, 31 -- note we might want to sequence once at a time using a task list since we may run out of capacity, at least not all at once?"

**Prompt 3:** "kick off the remainder in sub-agents, we have capacity"

Generated by Claude Fable 5.1 using the blog-post-generator skill. One agent researched the web repository and proposed this post among a batch of candidates; a second agent verified the claims and wrote it. Sources: `plans/source-region-validation.md` (decided 2026-07-24), `app/models/api/weather.rb`, `app/models/api/sources/base.rb`, `app/models/api/sources/secondary/base.rb`, `app/controllers/api/api_controller.rb`, `test/models/api/region_validation_test.rb`, the iOS `ForecastService` and forecast view model, and the web repository's pull requests of July 24 to 26, 2026 (coordinate range validation, the pre-flight region gate, the unresolvable-location error, the null-location vendor case, and the source-error review that surfaced the ~500/day figure). Judgment calls: timezone providers and weather sources are unnamed, with placeholder names in the excerpts; the regional source is described as a US national service without naming it; the metrics and error-reporter calls are removed from the excerpts and the tools are unnamed; the Norwegian Sea coordinates are described by place rather than by number; the "no consumer on main" caveat is stated as of the writing date because the adopting pull requests had not merged; no after-count for the 502 volume is claimed because none is recorded.
