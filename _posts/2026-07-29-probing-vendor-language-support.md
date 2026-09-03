---
layout: post
title: "Never Trust the Vendor's Language List"
date: 2026-07-29 08:40:00 -0600
summary: "We sent real requests to ten upstream data sources in all 27 supported languages and found four different ways a vendor fails that its documentation doesn't mention."
tags: [api-design, localization, i18n, ruby, testing]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Each upstream source declared its language support in one line of Ruby: the full language list, minus the codes the vendor's documentation said it couldn't serve.

```ruby
class Api::Sources::VendorC < Api::Sources::Base
  def self.supported_source_languages
    Api::Units::LANGUAGE - %w(zh_TW hi th vi)
  end
end
```

That subtraction list came from a docs page. Nobody had checked it against the API.

[Hello Weather](https://helloweather.com) pulls from eleven sources and serves 27 languages. Ten of them had a list like this, which makes a 270-cell matrix, and since the lists were written at the end of 2025 every cell had been filled in by reading the docs.

Nobody checked because localization bugs don't announce themselves. A source that returns English for a Korean request still returns a valid 200 with a readable summary. Tests pass. Monitoring is green. The only person who notices is a Korean user who sees "Mostly cloudy" in the middle of an otherwise Korean screen, and that user doesn't file a bug report. They just decide the app is bad.

So we checked all 270 cells.

## The Solution

The procedure is simple. Hit the vendor's real endpoint once per language and compare what comes back to the English answer.

```ruby
BASELINE = fetch(language: "en").summary   # e.g. "Cloudy. Warm."

Api::Units::LANGUAGE.each do |lang|
  response = fetch(language: lang)
  puts [lang, response.summary, response.summary == BASELINE ? "UNSUPPORTED" : "ok"].join("\t")
end
```

The comparison does all the work. If the response is the same text as the English one, the language isn't supported, whatever the docs say. We found no other signal. Vendors don't set a header, don't return a `language` field, and don't warn you.

Two refinements matter in practice:

1. **Probe variant codes, not just your own.** For each language we also tried the nearby codes: `zh_tw`, `zh-tw` and `zh_hk`; `no` and `nb`; `pt`, `pt-br` and `pt-pt`; and whatever nonstandard code the vendor's docs hinted at. Several "unsupported" languages worked under a code we'd never sent.
2. **Look at the characters, not the code you sent.** A response that differs from English proves something was translated. It doesn't prove the right thing was translated. Traditional versus Simplified Chinese has to be checked by eye.

Everything below came out of one day of probing, 2026-07-20.

## The Four Failure Modes

When you send a language code a vendor can't honor, one of four things happens. They aren't equally bad, and they aren't equally visible.

| # | Failure mode | What the client sees | Example |
|---|---|---|---|
| 1 | **Hard error** | Request fails outright | Vendor A rejects the request with a "language not supported" error body and the entire forecast fails. Vendor B returns HTTP 400 on an unrecognized code. |
| 2 | **Silent English fallback** | A valid response, in English | Vendor C documents Korean support; Korean requests return English. So does the vendor's own alternate Korean code. |
| 3 | **Silent wrong script / register** | A valid response, translated — into the wrong variant | Vendor D's documented Traditional Chinese code returns *Simplified*. Their Hong Kong code is the one that actually returns Traditional. Separately, a bare `pt` gets you European Portuguese when your content is Brazilian. |
| 4 | **Placeholder token leak** | A valid response containing raw internal tokens | Vendor E returns strings like `type_42` for one claimed language — untranslated template keys the client would render verbatim to the user. |

Mode 1 is the friendly one. It's loud and it gets caught in staging, because nobody can ignore it. It's also the most dangerous one to guess about. On a hard-error source, an unsupported code doesn't just degrade the summary, it kills the whole forecast. That exclusion list has to be exact.

Modes 2, 3, and 4 all return HTTP 200. At the transport layer they look like success, which is how all three survived seven months in production behind a matrix copied from the docs.

Mode 3 is the subtlest. Comparing against the English baseline catches mode 2 and usually mode 4, but a Traditional Chinese request answered in Simplified passes that check. It isn't English, and it's genuinely translated. It's just wrong for the reader, and only a person looking at the characters catches it.

Mode 4 produces the worst thing a user can see: not English, not the wrong dialect, but a literal `type_42` in the forecast.

The corrections we shipped went in both directions. The docs weren't too cautious or too optimistic. They were wrong at random:

```ruby
# Before: exclusions copied from vendor docs
def self.supported_source_languages
  Api::Units::LANGUAGE - %w(zh_TW hi el id ko ro th uk vi)
end

# After: probed 2026-07-20 — every one of those works at the plain code,
# and Traditional Chinese works via the vendor's Hong Kong code
def self.supported_source_languages
  Api::Units::LANGUAGE
end

def api_language
  case source_language
  when "pt" then "pt_br"
  when "zh_TW" then "zh_hk"
  else super
  end
end
```

One source had nine languages wrongly excluded. Another had a documented language we had to exclude. A third needed three language codes remapped, because the codes the vendor's API accepts differ from the ISO 639-1 standard in a few places. Send the standard code and you get English back with no complaint.

Two sources we'd marked English-only for their whole lifetime turned out to be localizable. One had a `language` parameter, missing from our notes, that translated all 27 languages. The other returns numeric condition codes instead of prose, so we do the translating ourselves:

```ruby
SUMMARY_MAP = {
  nil  => nil,
  1001 => "cloudy",
  2100 => "light_fog",
  6200 => "light_freezing_rain",
  # ...
}.freeze

def to_summary(val)
  key = SUMMARY_MAP.fetch(val) { raise(Api::Weather::NotImplementedError, val.nil? ? "nil" : val) }
  return nil if key.nil?

  I18n.t("api.condition_code.#{key}", locale: source_language)
end
```

A vendor that returns codes instead of sentences isn't a localization problem. We get all 27 languages for the price of a translation file, and we own the wording.

## Reporting What Actually Happened

Fixing the matrix fixes today. It doesn't fix the next time a vendor quietly changes behavior, and it doesn't help the client, which still has no idea whether the prose it just received is in the language it asked for.

So every output, JSON and GraphQL, gained one top-level field that reports the language the source actually served:

```ruby
class Api::Sources::Base
  def source_language
    if self.class.supported_source_languages.include?(return_units.language)
      return_units.language
    else
      "en"
    end
  end
end
```

```ruby
class Api::Outputs::Base
  attribute :source, &:source_name
  attribute :language, &:source_language
end
```

```json
{
  "latitude": 41.87,
  "longitude": -87.62,
  "source": "some_source",
  "language": "en"
}
```

Request `lang=de` from a source that can't serve German and you get `"language": "en"`. The client compares the field to what it asked for and decides what to do: fall back to a different source, hide a mixed-language string, or just render it.

The field is always there, not only on a mismatch. Emitting it only on a mismatch is tempting. It's smaller, and the field is interesting exactly when it differs. It's also a trap:

- A sometimes-present key forces every client to write `response.language ?? requestedLanguage`, which copies server logic into every client, forever.
- Absence is ambiguous. A missing key means "matched", but it also means "old server version", "field renamed", or "serializer bug". You can't tell those apart.
- The neighboring fields (`source`, `promo`, `retiredSources`) are always present. A convention that holds across the payload is worth more than a clever exception for one field.
- An always-present field documents itself. Anyone reading a response sees that the API has an opinion about the language it served.

The rule underneath is to report what happened, not what was requested. An API that echoes the parameters back tells the client nothing. One that says what it did with them gives the client something to act on.

Tests pin all three behaviors, including the boring one:

```ruby
test "language is emitted at the top level of the payload" do
  de = Api::Weather.new(source: "mock", output: "hello_weather", lang: "de").to_h
  en = Api::Weather.new(source: "mock", output: "hello_weather", lang: "en").to_h

  assert de.key?(:language)
  assert_equal "en", de[:language]   # mock is English-only
  assert_equal "en", en[:language]
end
```

The `assert de.key?(:language)` isn't redundant with the `assert_equal` below it. It's the line that fails if someone later "optimizes" the field into being conditional.

## Results

- Ten sources probed across 27 languages, plus variant codes, in one session on 2026-07-20. Nine of the ten needed a correction, to the exclusion list or to the code we send, and the exclusion lists were rewritten from what the probe showed.
- Two sources that had been English-only for years gained full 27-language support: one through a vendor parameter we hadn't been using, one through translating numeric condition codes on our server.
- One claimed-supported language newly excluded, one placeholder-token leak caught before it reached users, and one wrong-script mapping corrected.
- Every response now carries an always-present `language` field. The corrected matrix will drift as vendors change, so the probe procedure, the failure-mode table, and the probe date live in a skill file that every future source integration has to follow.

## Lessons Learned

- **Assume any "supported" list is aspirational.** Docs say what the vendor meant to build, and nobody checks that against what shipped. If your product depends on a claim, test it against the live API.
- **A baseline diff catches fallback, not the wrong variant.** A response that differs from English proves something was translated, not the right thing. Some checks still need a person looking at the output.
- **Always-present fields beat sometimes-present ones.** A field that appears only when it's interesting pushes a branch into every consumer and makes absence ambiguous. Always emit it and let the client compare.
- **Write down the procedure, not just the fix.** A corrected table goes stale. The written procedure makes the next integration run the same check.
