---
layout: post
title: "One Portuguese: Why a Partial Locale Is Worse Than None"
date: 2026-08-24 08:50:00 -0600
summary: "Apple's String Catalog has no key-level fallback between locale variants, so a present-but-incomplete pt-BR table renders English for every missing key - and the fix is one complete locale, not two half-finished ones."
tags: [localization, i18n, ios, api-design]
---

## The Problem

[Hello Weather](https://helloweather.com) carries far more translated text than a typical weather app, built by one person with a model doing the first-pass translation into 26 languages. That combination makes a specific mistake tempting: when you add a regional variant, translate the part you have time for and let the rest inherit from the base language. It feels safe because every intuition about locale resolution says a variant layers on top of its base - fill in what is specific, inherit the rest.

That intuition is wrong on Apple's platforms, and Brazil is where it bites hardest, because Brazil is the dominant Portuguese market for the app. We shipped a `pt` locale in European register and added `pt-BR` beside it with 582 of 1,087 keys translated, expecting the missing 505 to render as European Portuguese from `pt`. On a device, this is what happens instead:

> When a `pt-BR.lproj` table exists but is missing a key, the system does not fall back to `pt`. It falls back to the source language. For us that is English.

A Brazilian user did not see European Portuguese for the untranslated half of the app. They saw English, interleaved word by word with the Portuguese we had finished: a stat card in Portuguese above a button in English above an alert in Portuguese. It looked broken because it was, and no test caught it, because every string was "present" somewhere in the catalog.

The thing that would have worked is the thing we did not do. A locale you never declare falls back cleanly: `pt-PT` on a device with no `pt-PT.lproj` resolves to `pt` at the file level, exactly the inheritance you wanted. Declaring the incomplete variant is what turns that inheritance off and exposes the source language underneath.

## The Solution

We deleted `pt-BR` and now ship one Portuguese locale, plain `pt`, filled with Brazilian content. CLDR and Apple both resolve unqualified `pt` to Brazilian Portuguese, so this is the platform's own default, not a workaround. European Portuguese devices fall back to `pt` at the file level and get a fully translated app in a Brazilian register - a real, whole-file fallback rather than a key-by-key one that can leave holes.

The consolidation was mechanical: promote the fully populated table to `pt`, drop the `pt-BR` column from every key, and remove the `ptBR` case from the `Language` enum so the picker shows a single **Português**. The 505 rows still in European register did not block the change; we flagged them for a later Brazilian-register pass and shipped. Register drift is a copy-editing task you can schedule. English leaking into half your UI is a shipping defect you cannot.

### Make the "no fallback" rule fail the build

The dangerous property is that nothing complains: the build succeeds, the catalog is valid, the app runs. The failure appears only as rendered English on a Portuguese device. A rule that subtle cannot live in someone's memory, so we made completeness a test that reads the source catalog and fails when any key is present in some languages and missing in others. It does not mention Portuguese; it enforces the general invariant that caught us. Here it is, self-contained:

```swift
import Testing
import Foundation

@Suite("Portuguese is one complete locale")
struct PortugueseLocaleTests {
    private func loadStrings() throws -> [String: Any] {
        let url = URL(fileURLWithPath: #filePath)
            .deletingLastPathComponent()
            .appendingPathComponent("Localizable.xcstrings")
        let json = try #require(
            JSONSerialization.jsonObject(with: Data(contentsOf: url)) as? [String: Any]
        )
        return try #require(json["strings"] as? [String: Any])
    }

    @Test("No pt-BR table exists, and pt covers every translated key")
    func portugueseIsSingleAndComplete() throws {
        for (key, entry) in try loadStrings() {
            let locs = ((entry as? [String: Any])?["localizations"] as? [String: Any]) ?? [:]
            #expect(locs["pt-BR"] == nil, "\"\(key)\" reintroduced a pt-BR table")

            // Flag-gated keys stay untranslated in every language; any key that
            // carries a non-English row must also carry pt.
            let hasTranslation = locs.keys.contains { $0 != "en" }
            #expect(!hasTranslation || locs["pt"] != nil, "\"\(key)\" is missing pt")
        }
    }
}
```

Notice the test reads the `.xcstrings` file on disk, not the compiled bundle, so it catches a reintroduced variant or a dropped `pt` row before a build can repackage the damage. A green run is a proof that Portuguese is exactly one complete table.

### The server rejects `pt-BR` on purpose

The client is half the system. Hello Weather's forecast API takes a `lang` parameter, and the generous instinct after consolidating is to accept `pt-BR` on the wire and alias it to `pt` so no caller breaks. We return HTTP 400 instead, and the reason is not strictness for its own sake: the alias table is directional, and one of its uses runs backwards.

The map normalizes an inbound regional code to a simple internal one. Inverted, the same map turns an internal code back into a canonical regional code to send outbound to providers that speak BCP47. Ruby's `Hash#invert` is last-key-wins, and that is the trap. This runs as-is:

```ruby
# Forward: normalize an inbound regional code to a simple internal one.
canonical = { "en-US" => "en", "pt-PT" => "pt" }

# Inverted: turn an internal code back into the canonical regional one to
# send outbound to providers that speak BCP47.
canonical.invert.fetch("pt")   # => "pt-PT", correct for the region

# Now "helpfully" accept an inbound pt-BR alias in the SAME map:
canonical = { "en-US" => "en", "pt-PT" => "pt", "pt-BR" => "pt" }

# Two keys now map to "pt". invert is last-key-wins, so the later one survives:
canonical.invert.fetch("pt")   # => "pt-BR", every outbound request flips region
```

A one-line convenience on the inbound side silently changes the Portuguese code we send to third parties, and nothing in the inbound tests would show it. The fix is to keep the canonical map a clean bijection and put inbound-only aliases in a separate merged table the inversion never touches:

```ruby
module Units
  # Inbound aliases go in LANGUAGE_ALIASES, never here: .invert (last key wins)
  # drives OUTBOUND codes - "pt-BR"=>"pt" would flip outbound pt-PT to pt-BR.
  BCP47_TO_SIMPLE = {
    "en-US" => "en", "de-DE" => "de", "pt-PT" => "pt",
    "zh-CN" => "zh", "zh-TW" => "zh_TW"
    # ...one row per supported region
  }.freeze

  SIMPLE_TO_BCP47 = BCP47_TO_SIMPLE.invert.freeze

  # Inbound-only aliases the inversion must never see.
  LANGUAGE_ALIASES = BCP47_TO_SIMPLE.merge(
    "zh-Hans" => "zh", "zh-Hant" => "zh_TW", "no" => "nb"
  ).freeze

  def self.normalize_language(lang)
    LANGUAGE_ALIASES.fetch(lang.to_s, lang.to_s)
  end
end
```

Note that `pt-BR` appears in neither table, so `normalize_language` leaves it unchanged and the request-validation layer rejects it as an unknown code. Two tests pin both halves of the contract - the outbound code stays European, and `pt-BR` never resolves to something known:

```ruby
require "minitest/autorun"
require "set"

class UnitsLanguageTest < Minitest::Test
  KNOWN = Units::BCP47_TO_SIMPLE.values.to_set

  def test_outbound_portuguese_stays_european
    assert_equal "pt-PT", Units::SIMPLE_TO_BCP47.fetch("pt")
  end

  def test_pt_br_is_not_a_known_code
    # normalize_language passes it through unchanged; validation then 400s it.
    refute_includes KNOWN, Units.normalize_language("pt-BR")
  end
end
```

The first test is the one that fails loudly the moment someone adds `"pt-BR" => "pt"` to `BCP47_TO_SIMPLE`: the outbound code flips to `pt-BR` and the assertion breaks, on the inbound change, at the exact spot the mistake was made.

## Results

- **The ~46%-English failure mode is gone.** With one complete `pt` and no incomplete variant table beside it, no key can render the source language for a Portuguese user.
- **Portuguese maintenance halved.** One column to translate, review, and keep complete instead of two that drift against each other.
- **Brazilian users get a Brazilian-register app** aligned with how CLDR and Apple already resolve `pt`, and European devices still get a fully translated app rather than an English-riddled one.
- **The outbound provider code for Portuguese is provably stable**, with a single failing test standing between a well-meaning inbound alias and a corrupted outbound region code.

## Lessons Learned

- **A partial locale is worse than none.** A missing locale falls back cleanly to its base language at the file level; a present-but-incomplete one disables that inheritance and renders your source language for every missing key. Ship a locale complete or do not declare it.
- **One complete locale beats two incomplete ones.** Prefer the unqualified language code carrying your dominant market's content over a base plus a half-finished variant, because the platform already resolves the unqualified code that way and you stop fighting it.
- **Know which direction each lookup table is read in.** A map that is also inverted elsewhere encodes two contracts in one literal; an addition correct for the inbound read can be silently wrong for the outbound one. Keep the inverted map a clean bijection and give inbound-only aliases their own table.
- **Put the warning at the line where someone will trip.** The invert hazard lives in a comment directly above the map, the completeness invariant lives in a build-failing test, and the outbound code is pinned by an assertion that breaks on the exact edit that would corrupt it.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
