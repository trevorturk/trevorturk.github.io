---
layout: post
title: "Never Trust the Vendor's Language List"
date: 2026-07-29 08:40:00 -0600
summary: "We live-probed ten upstream data sources in all 27 supported languages and found four distinct failure modes hiding behind vendor documentation."
tags: [api-design, localization, i18n, ruby, testing]
---

## The Problem

Each upstream source declared its language support in one line of Ruby: the full language list, minus the codes the vendor's documentation said it could not serve.

```ruby
class Api::Sources::VendorC < Api::Sources::Base
  def self.supported_source_languages
    Api::Units::LANGUAGE - %w(zh_TW hi th vi)
  end
end
```

That subtraction list came from a docs page. Nobody had ever checked it against the API.

[Hello Weather](https://helloweather.com) aggregates eleven sources and serves 27 languages. Ten of them carried a docs-derived declaration, a 270-cell matrix, and since it was written at the end of 2025 every cell had been filled in by reading.

Nobody checked because localization bugs do not announce themselves. A source that silently returns English for Korean still returns a valid 200 with a readable summary. Tests pass. Monitoring is green. The only person who notices is a Korean user who sees "Mostly cloudy" in the middle of an otherwise-Korean interface, and that user does not file a bug report. They just think the app is bad.

So we checked all 270 cells.

## The Solution

The procedure is unglamorous. Hit the vendor's real endpoint once per language and compare what comes back to the English baseline.

```ruby
BASELINE = fetch(language: "en").summary   # e.g. "Cloudy. Warm."

Api::Units::LANGUAGE.each do |lang|
  response = fetch(language: lang)
  puts [lang, response.summary, response.summary == BASELINE ? "UNSUPPORTED" : "ok"].join("\t")
end
```

The comparison does all the work. **If the response equals the English baseline byte-for-byte, the language is not supported — no matter what the docs say.** There is no other reliable signal. Vendors do not set a header, do not return a `language` field, and do not warn you. Byte equality with English is the tell.

Two refinements matter in practice:

1. **Probe variant codes, not just your canonical ones.** For each language we also tried the plausible neighbors: `zh_tw` / `zh-tw` / `zh_hk`, `no` versus `nb`, `pt` versus `pt-br` versus `pt-pt`, plus whatever nonstandard code the vendor's docs hinted at. Several "unsupported" languages turned out to be supported under a code we had never sent.
2. **Inspect the characters, not the code you sent.** Byte-inequality with English proves *something* was translated. It does not prove the *right* thing was translated. Traditional versus Simplified Chinese has to be verified by looking at the glyphs.

Everything below came out of a single day of probing, on 2026-07-20.

## The Four Failure Modes

When you send a language code a vendor cannot honor, one of exactly four things happens. They are not equally bad, and they are not equally *visible*.

| # | Failure mode | What the client sees | Example |
|---|---|---|---|
| 1 | **Hard error** | Request fails outright | Vendor A rejects the request with a "language not supported" error body and the entire forecast fails. Vendor B returns HTTP 400 on an unrecognized code. |
| 2 | **Silent English fallback** | A valid response, in English | Vendor C documents Korean support; Korean requests return English. So does the vendor's own alternate Korean code. |
| 3 | **Silent wrong script / register** | A valid response, translated — into the wrong variant | Vendor D's documented Traditional Chinese code returns *Simplified*. Their Hong Kong code is the one that actually returns Traditional. Separately, a bare `pt` gets you European Portuguese when your content is Brazilian. |
| 4 | **Placeholder token leak** | A valid response containing raw internal tokens | Vendor E returns strings like `type_42` for one claimed language — untranslated template keys the client would render verbatim to the user. |

Mode 1 is the friendly one. It is loud, it is caught in staging, and it is the only mode a naive integration survives, because it is the only one that cannot be ignored. It is also the most dangerous to *guess* about. On a hard-error source, an unsupported code does not degrade the summary. It kills the whole forecast, so that gate has to be exact.

Modes 2, 3, and 4 all return HTTP 200. They are indistinguishable from success at the transport layer, which is why every one of them survived seven months in production behind a docs-derived matrix.

Mode 3 is the subtlest. Byte-comparison against the English baseline catches mode 2 and usually mode 4, but a Traditional Chinese request answered in Simplified passes that check. It is not English. It is genuinely translated. It is just wrong for the reader, and only a human looking at the characters catches it.

Mode 4 produces the worst user-visible artifact: not English, not a wrong dialect, but a literal `type_42` rendered in the forecast.

The corrections we shipped went in both directions. The docs were not pessimistically wrong or optimistically wrong. They were *randomly* wrong:

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

One source had nine languages wrongly excluded. Another had a documented language that had to be newly excluded. A third needed three language codes remapped, because the vendor's wire codes silently diverge from ISO 639-1 in a handful of places. Send the standard code, get English back, no complaint.

Two sources that had been marked English-only for their entire lifetime turned out to be localizable. One had a `language` parameter, missing from our notes, that translated all 27 languages. The other returns numeric condition codes rather than prose, which makes the localization ours to do:

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

A vendor that returns codes instead of sentences is not a localization problem. It is a localization *opportunity*: you get all 27 languages for the price of a translation file, and you own the wording.

## Reporting What Actually Happened

Fixing the matrix fixes today. It does not fix the next time a vendor quietly changes behavior, and it does not help the client, which still has no idea whether the prose it just received is in the language it asked for.

So every output, JSON and GraphQL, gained one additive top-level field reporting the language the source actually served:

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

Request `lang=de` from a source that cannot serve German and you get `"language": "en"`. The client compares the field to what it asked for and decides what to do: fall back to a different source, suppress a mixed-language string, or just render it.

The field is emitted always, not only on mismatch. Emitting on mismatch is tempting. It is smaller, and the field is "interesting" exactly when it differs. It is also a trap:

- A sometimes-present key forces every client to write `response.language ?? requestedLanguage`, which re-derives server logic in every client, forever.
- Absence is ambiguous. A missing key means "matched", but it also means "old server version", "field renamed", or "serializer bug". You cannot tell those apart.
- The neighboring fields (`source`, `promo`, `retiredSources`) are always present. Conventions inside a payload are worth more than per-field cleverness.
- An always-present field is self-documenting. Anyone reading a response sees that effective language is a thing the API has opinions about.

The principle underneath is to report what actually happened, not what was requested. An API that echoes your parameters back tells you nothing. An API that tells you what it did with them lets clients be correct.

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

The `assert de.key?(:language)` is not redundant with the `assert_equal` below it. It is the test that fails if someone later "optimizes" the field into being conditional.

## Results

- Ten sources probed across 27 languages, plus variant codes, in one session on 2026-07-20. Nine of the ten needed a correction, to the exclusion list or to the wire code, and the exclusion lists were rewritten from probe evidence.
- Two sources that had been English-only for years gained full 27-language support: one via an unused vendor parameter, one via server-side translation of numeric condition codes.
- One claimed-supported language newly excluded, one placeholder-token leak caught before it reached users, and one wrong-script mapping corrected.
- Every response now carries an always-present `language` field. The corrected matrix will drift as vendors change, so the probe procedure, the failure-mode table, and the probe date live in a skill file that any future source integration is required to follow.

## Lessons Learned

- **Assume any "supported" list is aspirational.** Docs describe intent, not reality, and nobody reconciles the two. A claim your product depends on deserves a test against production, not a citation.
- **A baseline diff catches fallback, not the wrong variant.** Byte-inequality proves something was translated, not the right thing. Some checks still need a human looking at the output.
- **Always-present fields beat sometimes-present ones.** A field that appears only when "interesting" pushes a branch into every consumer and makes absence ambiguous. Always emit; let the client compare.
- **Encode the procedure, not just the fix.** A corrected table is a snapshot with an expiration date. The written procedure is what keeps the next integration honest.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The title lost its subtitle, the post now opens on the one-line exclusion list nobody had checked, the failure-mode commentary was split into shorter sentences, Results dropped the bullets that restated the Solution, and Lessons Learned went from seven rules to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The probe date, the four failure modes, the nine-language exclusion list, the three remapped codes, and the `language` field and its test all matched the 2026-07-20 source-language PR and the api-localization skill. Corrections: the source count is eleven, with ten carrying the docs-derived declarations that make the 270-cell matrix; that matrix dated from the end of 2025, so "for years" became "seven months"; "every source's declaration was wrong" became "nine of the ten needed a correction," since one English-only source was left unchanged; the excerpts now use `supported_source_languages` and `else super`, the names the code took the day after the probe when `supported_languages` was renamed and the `language_code` helper was deleted; and the condition-code map excerpt gained the `nil` entry and error message from the real code.
