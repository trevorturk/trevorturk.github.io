---
layout: post
title: "Rate, Start, Fair: Disambiguating Short Keys Before 26 Languages Mistranslate Them"
date: 2026-08-24 08:40:00 -0600
summary: "The one-word English keys are the ones that mistranslate at scale, and the fix is upstream at authoring time: translator context in the key, a per-language register table, two glossaries, and a distinctness test before commit."
tags: [localization, i18n, ai-agents, workflow]
---

## The Problem

A weather app is mostly one- and two-word labels: chart legends, stat-card titles, compact widget and complication slots. That is the text that breaks in translation, because a short key carries no context, and a model translating hundreds in bulk has nothing but the word to go on.

At [Hello Weather](https://helloweather.com) the String Catalog holds 1,324 keys across 27 languages — more than 32,000 translated rows. One of them is the single word `Rate`. It renders as a chart legend for a precipitation rate, and nothing in the key says so. A human translator with a screenshot would never mistranslate it; a translator or a model working from a flat list will.

An early quality pass found `Rate` translated as the *rate-the-app* verb in all 26 languages. The German row read „Bewerten" — the App Store review prompt, not a measure of how hard it is raining. It was a correct translation of the wrong sense. The same pass found `Start` rendered as an imperative button in 12 languages when it labeled a start *time*, and `Upgrade` as update-the-app in 9 when it meant the paid tier.

This polysemy failure is worse than ordinary translation error for three reasons. **It is invisible in review:** every row passes spellcheck, a fluency read, and a native speaker handed the word without the screen — the string is grammatical and idiomatic, only the sense is wrong. **It scales with language count, not string count:** one ambiguous English key becomes 26 confident wrong answers, and the more complete the localization, the more places the mistake ships. **There is no fallback to catch it:** a missing row renders the raw English key, obvious on sight, but a present-but-wrong-sense row renders fluent local text that looks finished. The failure mode that survives review is the one that looks done.

The instinct is to fix these downstream, in a QA pass over the finished translations — the expensive place. The right place is before a single translation exists.

## The Solution

The rule is one sentence.

> **Disambiguate a short key at authoring time, in the same PR that introduces it — never in a later QA pass over 26 finished translations.**

When you add a key whose sense is not obvious from its own text, you do one of two things before anyone translates it: attach a translator comment stating the rendering context, or split it into a distinct key. Both encode the missing fact — *this `Rate` is a precipitation rate, not the review prompt* — where the person or model producing the other languages will see it.

Everything else in this post is the machinery that makes a disambiguated key translate *consistently* once its sense is fixed: a register table, two glossaries, a distinctness check, plural handling, and a typography table. Disambiguation decides what the string means; the rest decides that all 26 languages say it the same way.

### Disambiguate at the source

The polysemy class is small and recognizable: `Rate`, `Start`, `Fair`, `Upgrade`, `Match`, `Clear` — one-word keys that are a verb and a noun at once, or a weather term and a UI verb. `Fair` is the sharpest: an air-quality band, a visibility level, and the everyday adjective, each wanting a different word in most languages. The code-review checklist names the class so a reviewer stops on it:

> Short/ambiguous keys are disambiguated for translators (comment or distinct key): the polysemy class ("Rate", "Start", "Fair", "Upgrade") mistranslates in dozens of languages.

The cheaper fix is a translator comment: an argument on the same `Text` that draws the label, so the rendered output is unchanged but the note ships to translators:

```swift
import SwiftUI

struct PrecipLegendView: View {
    var body: some View {
        HStack {
            // Ambiguous: every translator sees only the word "Rate".
            Text("Rate")

            // Same pixels, but the comment ships to translators and
            // fixes the sense.
            Text("Rate", comment: "Precipitation rate; chart legend label, not the rate-the-app verb")

            // Stronger fix: a distinct key, for when the same English
            // word renders in two senses in the same app.
            Text("Precip rate", comment: "Chart legend label for rainfall intensity")
        }
    }
}
```

`Text("Rate", comment:)` renders the same label as `Text("Rate")`; the only difference is what a translator sees. Reach for a distinct key when the same English word carries two senses in the app, as `Fair` does for air quality and visibility, where Czech and Spanish need two words for one English one.

The comment travels into the String Catalog as a sibling of the translations, so every language row inherits the same context:

```json
{
  "sourceLanguage" : "en",
  "strings" : {
    "Rate" : {
      "comment" : "Precipitation rate; chart legend label, not the rate-the-app verb.",
      "localizations" : {
        "de" : {
          "stringUnit" : { "state" : "translated", "value" : "Intensität" }
        },
        "fr" : {
          "stringUnit" : { "state" : "translated", "value" : "Intensité" }
        },
        "ja" : {
          "stringUnit" : { "state" : "translated", "value" : "強度" }
        }
      }
    }
  }
}
```

The `comment` field is the mechanism: the one place the authoring context reaches all 26 translators at once, steering German, French, and Japanese to the rainfall-intensity sense rather than the review-prompt verb.

### One register per language, written down

Once a key's sense is fixed, the next inconsistency is register: formal or informal address, applied *the same way every time*. Nothing reads more machine-generated than a German screen that says „du" in one sentence and „Sie" in the next.

The fix is not judgment per string. It is one decision per language, recorded in a table and applied everywhere:

| Register | Languages |
|---|---|
| Informal singular | de du, fr tu, it tu, es tú, pt você (Brazilian), nl je/jij, da/sv/nb du, fi sinä, hu te, ro tu, pl ty |
| Formal / polite | ru вы, uk ви, cs vy, el εσείς, tr siz, hi आप, id Anda |
| Politeness-system | ja です/ます, ko 해요체, th polite-neutral (no ครับ/ค่ะ), vi bạn, zh neutral (您 in prose) |

The buckets are not arbitrary. Informal-singular is the register a weather app earns in German, French, and the Nordic languages — it is what Apple's own weather surfaces use. Formal is correct for Russian, Czech, Turkish, and Hindi, where informal address to a stranger reads as rude. The CJK and Thai languages have politeness *systems* rather than a formal/informal binary, and the table records the specific level to hold (Japanese friendly-polite です/ます, not the stiff でございます). **Pick one register per language and record it; do not decide register per string** — a reviewer, a model, and a later audit then consult the same row.

### Two glossaries: match the platform, fix the domain

Terminology is the third lever, and it splits into two problems.

The first is OS vocabulary. When your copy tells a user to enable something in Settings, the word for "Notifications" or "Lock Screen" must match the exact word Apple's own localized OS uses, or the instruction points at a label that does not exist on the user's screen. We mined Apple's shipped `.lproj` and `.loctable` strings — over 10,000 per language — into a per-language OS-term glossary:

| Lang | notifications | complications | lock_screen | settings |
|---|---|---|---|---|
| de | Mitteilungen | Komplikationen | Sperrbildschirm | Einstellungen |
| ja | 通知 | コンプリケーション | ロック画面 | 設定 |
| ru | Уведомления | Расширения | Экран блокировки | Настройки |

The rule is *match verbatim when the copy directs the user into system UI* — while our own Appearance/Light/Dark labels stay product vocabulary, deliberately *not* forced to Apple's terms because they name our feature, not a screen the user will go tap.

The second glossary is the weather domain: pollen categories, AQI and visibility level names, alert phenomena, moon phases. These are terminology-of-record where client and server must agree — a level name the API returns has to be lexically identical to the one the app renders beside it — and where the distinctions are science, not style: a tropical alert term must not drift to an extratropical one, a waxing crescent must not become a waxing gibbous. Abbreviations localize to the target's established short form — English `AQI` becomes German `LQI` — while a hard never-translate list keeps brand names, `API`, `UV`, URLs, and placeholders in English everywhere. **Reuse terminology from a glossary; invent a term only once, and write it down the first time.**

### Distinctness, plurals, and typography

Three smaller rules finish the set, each catching a class of bug fluency review misses.

**Scale distinctness.** Two keys that render together in one color-coded scale may never share a translated value in any language. This collision check caught two live shipping bugs the first time we ran it (covered in [measuring strings before you translate them](/rendered-width-validation/)): Czech rendered one word for both `Cloudy` and `Overcast` in a cloud legend, Russian one word for both `High` and `Max` in a UV legend — two identically-labeled swatches, different colors, in a shipped chart. A translator working key-by-key cannot see that two English words landed on one native word; only a check that compares the whole scale can, and it is small enough to state in full:

```swift
import Testing

// A "scale" is a set of keys that render together as one color-coded
// legend; within a scale, no two keys may collapse to the same word
// in any single language.
struct Scale {
    let name: String
    let keys: [String]
}

// Stand-in for the catalog: key -> (language -> translated value).
typealias Catalog = [String: [String: String]]

func withinScaleCollisions(_ scales: [Scale], in catalog: Catalog) -> [String] {
    var collisions: [String] = []
    for scale in scales {
        var seen: [String: [String: String]] = [:]  // language -> value -> key
        for key in scale.keys {
            guard let rows = catalog[key] else { continue }
            for (language, value) in rows {
                let normalized = value.lowercased()
                if let other = seen[language]?[normalized] {
                    collisions.append(
                        "\(scale.name)/\(language): \"\(value)\" shared by \(other) and \(key)")
                } else {
                    seen[language, default: [:]][normalized] = key
                }
            }
        }
    }
    return collisions.sorted()
}

@Test func scaleValuesStayDistinctPerLanguage() {
    let scales = [
        Scale(name: "cloudLegend", keys: ["Cloudy", "Overcast"]),
        Scale(name: "uvLegend", keys: ["High", "Max"]),
    ]
    let catalog: Catalog = [
        "Cloudy":   ["en": "Cloudy",   "cs": "Zataženo", "de": "Bewölkt"],
        "Overcast": ["en": "Overcast", "cs": "Zataženo", "de": "Bedeckt"],
        "High":     ["en": "High",     "ru": "Высокий",  "de": "Hoch"],
        "Max":      ["en": "Max",      "ru": "Высокий",  "de": "Max"],
    ]

    #expect(withinScaleCollisions(scales, in: catalog) == [
        "cloudLegend/cs: \"Zataženo\" shared by Cloudy and Overcast",
        "uvLegend/ru: \"Высокий\" shared by High and Max",
    ])
}
```

The fixture is the two real bugs, and the assertion pins exactly the collisions the detector should find. In the live suite the catalog and scales load from `Localizable.xcstrings` and the width registry; the logic is this one function, which flags any language where two keys of a scale collapse to one word.

**Plural variations.** Counted strings need the forms the language actually distinguishes — CLDR one/few/many/other — not just singular and plural; our catalog carries 179 such keys. And a key whose only localizations are plural variations still needs an explicit English row in the `translated` state, or it renders the raw key ("Rising in 1 minutes.") because the source language was never given a finished form.

**Typography.** The last layer is punctuation and spacing, and the lesson is to mine the platform rather than argue style guides. Apple's shipped strings produced a per-language table: Traditional Chinese uses a midline ellipsis `⋯` (U+22EF), not `…`; Turkish writes the percent sign *before* the number (`%50`); French puts a non-breaking space before `: ; ! ?`. Where a national style guide disagreed, Apple won, because Apple's word is what the user's other apps already show. The one override is width: in a compact slot, **width beats orthography**.

## Results

- **The polysemy class is caught upstream now, not in QA.** `Rate` (mis-sensed in all 26 languages), `Start` (imperative in 12), and `Upgrade` (update-the-app in 9) became translator-comment or key-split fixes at the source — one line each instead of 26 downstream corrections apiece.
- **Register and terminology became lookups, not per-string judgment.** One register table across all 27 languages, plus two glossaries — OS terms matched verbatim into system UI, weather-domain terms kept in client/server agreement — so a reviewer and a model consult the same source every time.
- **The distinctness check found two live legend bugs** (Czech Cloudy/Overcast, Russian High/Max) that no fluency review had caught, and now runs over every scale in the catalog.
- **The residual error rate is low and concentrated.** With sense and register pinned before translation, a later review finds mostly preference, not the confident wrong answers the short keys used to produce.

## Lessons Learned

- **Disambiguate short keys at the source, in the same PR.** A one-word key that is both noun and verb becomes 26 confident wrong answers the moment it is translated, and a comment or a key split is the only fix that reaches the person or model doing the other languages.

- **The dangerous error is the fluent one.** A missing translation renders the raw key and is obvious; a wrong-sense translation renders idiomatic local text and looks finished. Design review to catch meaning, not grammar — grammar already passes.

- **Pick one register per language and write it down.** Register drift within a language is the clearest machine-translation tell there is, so make it a table lookup, not a per-string judgment.

- **Mine the platform for terminology and typography instead of arguing style guides.** Apple ships tens of thousands of localized strings per language on every device; where a national style guide disagrees, consistency with the apps your users already use wins.

- **Check the whole scale, not the key.** Two labels that share a color-coded chart must stay lexically distinct in every language, and only a check that compares them together sees the collision — fluency review reads keys one at a time. It is the same discipline as an [adversarial review round](/adversarial-review-rounds/): a named lens beats a vague second look.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
