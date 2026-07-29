---
layout: post
title: "Never Trust the Vendor's Language List: Probing 10 APIs in 27 Languages"
date: 2026-07-29 08:40:00 -0600
summary: "We live-probed every upstream data source in all 27 supported languages and found four distinct failure modes hiding behind vendor documentation."
tags: [api-design, localization, i18n, ruby, testing]
---

## The Problem

[Hello Weather](https://helloweather.com) aggregates ten upstream weather sources and serves 27 languages. Which source can serve which language is a matrix with 270 cells, and for years that matrix was built the obvious way: read each vendor's documentation, write down the list of language codes they claim to support, subtract the ones they don't.

```ruby
class Api::Sources::VendorC < Api::Sources::Base
  def self.supported_languages
    Api::Units::LANGUAGE - %w(zh_TW hi th vi)
  end
end
```

That subtraction list came from a docs page. Nobody had ever checked it against the API.

The reason nobody checked is that localization bugs don't announce themselves. If a source silently returns English for Korean, the response is a perfectly valid 200 with a perfectly readable summary. Tests pass. Monitoring is green. The only person who notices is a Korean user who sees "Mostly cloudy" sitting in the middle of an otherwise-Korean interface, and that user does not file a bug report. They just think the app is bad.

So we went and checked all 270 cells.

## The Solution

The procedure is unglamorous: hit the vendor's real endpoint, once per language, and compare what comes back to the English baseline.

```ruby
BASELINE = fetch(language: "en").summary   # e.g. "Cloudy. Warm."

Api::Units::LANGUAGE.each do |lang|
  response = fetch(language: lang)
  puts [lang, response.summary, response.summary == BASELINE ? "UNSUPPORTED" : "ok"].join("\t")
end
```

The load-bearing line is the comparison. **If the response equals the English baseline byte-for-byte, the language is not supported — no matter what the docs say.** There is no other reliable signal. Vendors do not set a header, do not return a `language` field, do not warn you. Byte equality with English is the tell.

Two refinements matter in practice:

1. **Probe variant codes too, not just your canonical ones.** For each language we also tried the plausible neighbors: `zh_tw` / `zh-tw` / `zh_hk`, `no` versus `nb`, `pt` versus `pt-br` versus `pt-pt`, plus whatever nonstandard code the vendor's docs hinted at. Several "unsupported" languages turned out to be supported under a code we'd never sent.
2. **Inspect the characters, not the code you sent.** Byte-inequality with English proves *something* was translated. It does not prove the *right* thing was translated. Traditional versus Simplified Chinese has to be verified by looking at the actual glyphs.

Everything below came out of a single day of probing, on 2026-07-20.

## The Four Failure Modes

When you send a language code a vendor can't honor, there are exactly four things that happen. They are not equally bad, and — importantly — they are not equally *visible*.

| # | Failure mode | What the client sees | Example |
|---|---|---|---|
| 1 | **Hard error** | Request fails outright | Vendor A rejects the request with a "language not supported" error body and the entire forecast fails. Vendor B returns HTTP 400 on an unrecognized code. |
| 2 | **Silent English fallback** | A valid response, in English | Vendor C documents Korean support; Korean requests return English. So does the vendor's own alternate Korean code. |
| 3 | **Silent wrong script / register** | A valid response, translated — into the wrong variant | Vendor D's documented Traditional Chinese code returns *Simplified*. Their Hong Kong code is the one that actually returns Traditional. Separately, a bare `pt` gets you European Portuguese when your content is Brazilian. |
| 4 | **Placeholder token leak** | A valid response containing raw internal tokens | Vendor E returns strings like `type_42` for one claimed language — untranslated template keys the client would render verbatim to the user. |

Mode 1 is the friendly one. It's loud, it's caught in staging, and it's the only mode a naive integration survives — because it's the only one that can't be ignored. It's also the most dangerous to *guess* about: on a hard-error source, an unsupported code doesn't degrade the summary, it kills the whole forecast. That gate has to be exact.

Modes 2, 3, and 4 all return HTTP 200. They are indistinguishable from success at the transport layer, which is why every one of them survived years in production behind a docs-derived matrix.

Mode 3 is the subtlest. Byte-comparison against the English baseline catches modes 2 and (usually) 4, but a Traditional Chinese request answered in Simplified passes that check with flying colors — it's not English, it's genuinely translated, it's just wrong for the reader. Nothing but human inspection of the characters catches it.

Mode 4 is the one that produces the worst user-visible artifact. Not English, not a wrong dialect: literal `type_42` rendered in the forecast.

The corrections we shipped went in both directions, which is the part worth internalizing. The docs were not pessimistically wrong or optimistically wrong — they were *randomly* wrong:

```ruby
# Before: exclusions copied from vendor docs
def self.supported_languages
  Api::Units::LANGUAGE - %w(zh_TW hi el id ko ro th uk vi)
end

# After: probed 2026-07-20 — every one of those works at the plain code,
# and Traditional Chinese works via the vendor's Hong Kong code
def self.supported_languages
  Api::Units::LANGUAGE
end

def api_language
  case source_language
  when "pt" then "pt_br"
  when "zh_TW" then "zh_hk"
  else language_code
  end
end
```

One source had nine languages wrongly excluded. Another had a claimed-and-documented language that had to be newly excluded. A third needed three of its language codes remapped because the vendor's wire codes silently diverge from ISO 639-1 in a handful of places — send the standard code, get English back, no complaint.

Two sources that had been marked English-only for their entire lifetime turned out to be localizable: one had an undocumented-in-our-notes `language` parameter that translated all 27 languages, and one returns numeric condition codes rather than prose, which means the localization is ours to do:

```ruby
SUMMARY_MAP = {
  1001 => "cloudy",
  2100 => "light_fog",
  6200 => "light_freezing_rain",
  # ...
}.freeze

def to_summary(val)
  key = SUMMARY_MAP.fetch(val) { raise(Api::Weather::NotImplementedError, val) }
  return nil if key.nil?

  I18n.t("api.condition_code.#{key}", locale: source_language)
end
```

A vendor that returns codes instead of sentences isn't a localization problem, it's a localization *opportunity*: you get all 27 languages for the price of a translation file, and you own the wording.

## Reporting What Actually Happened

Fixing the matrix fixes today. It doesn't fix the next time a vendor quietly changes behavior, and it doesn't help the client, which still has no idea whether the prose it just received is in the language it asked for.

So we added a single additive top-level field to every output — JSON and GraphQL — reporting the language the source **actually served**:

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

Request `lang=de` from a source that can't serve German and you get `"language": "en"`. The client can compare the field to what it asked for and decide what to do — fall back to a different source, suppress a mixed-language string, or just render it.

The design decision worth calling out is **emit it always, not only on mismatch**. Emitting on mismatch only is tempting — it's smaller, and the field is "interesting" exactly when it differs. It's also a trap:

- A sometimes-present key forces every client to write `response.language ?? requestedLanguage`, which is a re-derivation of server logic, in every client, forever.
- Absence is ambiguous. Missing key means "matched" — but it also means "old server version", "field renamed", "serializer bug". You cannot tell those apart.
- The neighboring fields (`source`, `promo`, `retiredSources`) are always present. Conventions inside a payload are worth more than per-field cleverness.
- An always-present field is self-documenting. Anyone reading a response sees that effective language is a thing the API has opinions about.

The general principle: **report what actually happened, not what was requested**. An API that echoes your parameters back at you tells you nothing. An API that tells you what it did with them lets clients be correct.

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

The `assert de.key?(:language)` is not redundant with the `assert_equal` below it. It's the test that fails if someone later "optimizes" the field into being conditional.

## Results

- **Ten sources probed across 27 languages**, plus variant codes, in one session.
- **Every source's capability declaration was wrong** in at least one direction. Exclusion lists were rewritten from probe evidence, not docs.
- **Two sources that had been English-only for years gained full 27-language support** — one via an unused vendor parameter, one via server-side translation of numeric condition codes.
- **One newly-excluded language**, caught only because the probe compared bytes: claimed-supported, silently English.
- **One placeholder-token leak** caught before it reached users.
- **One wrong-script mapping** corrected: the vendor's documented Traditional Chinese code returns Simplified.
- **A new always-present `language` field** on every response, so clients never have to guess.
- The probe procedure, the failure-mode table, and the probe date now live in a skill file that any future source integration is required to follow, so the matrix can't silently rot back into docs-derived guesses.

## Lessons Learned

- **Silent degradation is the default failure mode of third-party capability claims.** Vendors are not lying; their docs describe intent, their infrastructure describes reality, and nobody reconciles the two. Assume any "supported" list is aspirational.

- **Probe empirically. Never trust docs.** One day of hitting real endpoints found more defects than years of reading documentation. If a capability claim is load-bearing for your product, it deserves a test against production, not a citation.

- **Byte-equality with the baseline is the strongest available signal.** Vendors won't tell you they fell back. The response body will, if you have something to compare it to. Capture a known-language baseline first; everything after that is a diff.

- **A passing byte-comparison isn't proof of correctness.** Wrong-script and wrong-register failures produce genuinely-translated text that is genuinely wrong. Some checks require a human looking at glyphs.

- **Report what actually happened as an API design principle.** Echoing back the request is noise. Reporting the effective behavior — which source, which language, which fallback — is what lets a client be correct without reimplementing your logic.

- **Additive always-present fields beat sometimes-present ones.** A field that appears only when "interesting" pushes a branch into every consumer and makes absence ambiguous. Always emit; let the client compare.

- **Encode the procedure, not just the fix.** The corrected matrix is a snapshot with an expiration date. The written probe procedure — including "identical to English means unsupported" and "verify scripts by inspecting characters" — is the thing that keeps the next integration honest.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.
