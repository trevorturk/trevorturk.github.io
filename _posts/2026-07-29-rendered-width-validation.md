---
layout: post
title: "Measure the String Before You Translate It"
date: 2026-07-29 08:30:00 -0600
summary: "Standalone Swift validators that render every localized string at its real font and compare it to a computed layout budget, turning UI truncation in 22 languages into a committed work-list, and then into a server-side content fix that needed no app update."
tags: [swift, ios, localization, i18n, tooling]
---

## The Problem

Customer QA reported cut-off text in the small stat cards of the [Hello Weather](https://helloweather.com) iOS app, in Spanish. The obvious fix was to shorten the Spanish strings, which is the same loop every localized app runs: ship a screen that fits in English, translate it into 25 more languages, wait for a support ticket, fix one string, ship an app update, repeat.

The loop is slow because nothing in the toolchain knows how wide a string will be. A localization catalog stores text. A SwiftUI layout computes widths at runtime, on device, in a specific font at a specific Dynamic Type size. The two facts never meet until a human looks at a screenshot.

The truncation is also systemic rather than incidental. If `Air Quality` overflows a stat card title in Vietnamese, it overflows in the widget too, because it is the same catalog key in the same slot class. And when the string comes from the server, a level name or an advisory phrase, a one-word fix requires an app release that has nothing to do with the app. So instead of fixing the Spanish strings, we went looking for a way to know ahead of time every string in every language that would not fit.

## The Solution

Three parts, none of which require running the app:

1. **A committed JSON registry of layout constraints**: which catalog keys render in which width-constrained slots, and what the budget is for each
2. **A dependency-free validator script** that compiles and runs standalone, reads the registry plus the localization catalog, and measures
3. **A committed markdown report** that is the work-list, the diff, and the regression baseline at once

The validator has a lifecycle. It starts in audit mode, reporting findings and exiting 0, while there is a backlog. It flips to gate mode, with a non-zero exit, once the backlog is cleared.

We built it twice. The first generation counted characters. The second measured rendered widths. Both were useful, and the second exists because of what the first could not see.

## Generation 1: A Character-Budget Registry

The registry is a plain JSON file, committed to the repo:

```json
{
  "_readme": "Width budgets for catalog keys rendering in width-constrained slots (stat cards, chart legends, widget rows, complication labels). budget = max Character count for every language value except cjkExempt. scales = per-language values within one group must stay pairwise distinct.",
  "cjkExempt": ["ja", "ko", "zh-Hans", "zh-Hant"],
  "keys": {
    "Air Quality": { "slots": ["statTitle"],         "budget": 17 },
    "Cloudy":      { "slots": ["chartLegend"],       "budget": 9  },
    "AQI":         { "slots": ["complicationLabel"], "budget": 6  },
    "Actual":      { "slots": ["miniTitle"],         "budget": 9  }
  },
  "scales": {
    "uvLegend":       ["Low", "Mid", "High", "Max"],
    "pressureLegend": ["Low", "Normal", "High"],
    "visibilityLegend": ["Good", "Fair", "Poor"]
  }
}
```

The registry holds 74 keys across six slot classes and 14 scale groups, and it encodes two things a linter could not infer. A key is constrained because of where it renders, and one key can render in several places, so each entry names its slots. That makes the budget reviewable: someone can ask "is 17 characters really the stat title budget?" without reading layout code. Scale groups are sets of labels that appear together in one chart legend, and their rule is not about length at all. Within a scale, every language's values must be pairwise distinct. A translator working key-by-key has no way to know that two English words map to the same natural word in their language.

The scale rule found two shipping bugs on the first run:

- The Czech cloud legend rendered the same word for both `Cloudy` and `Overcast`
- The Russian UV legend rendered the same word for both `High` and `Max`

Two identically labeled swatches, different colors, in a chart that shipped. No width measurement would have caught those.

The validator is about 90 lines of Foundation. It parses the registry and the localization catalog as plain JSON, with no app dependency and no test target:

```swift
for scale in scales.keys.sorted() {
    for language in checkedLanguages {
        var seen: [String: String] = [:]
        for key in scales[scale] ?? [] {
            guard let translated = value(key, language) else { continue }
            if let previous = seen[translated] {
                findings.append("FINDING: within-scale duplicate in \(scale) " +
                                "\(language): \"\(previous)\" and \"\(key)\" " +
                                "both \"\(translated)\"")
            } else {
                seen[translated] = key
            }
        }
    }
}
```

A bash wrapper compiles it to a temp directory and runs it, so the whole tool is `./tools/validate-compact-strings` with no build system involved:

```bash
#!/usr/bin/env bash
set -euo pipefail
script_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
tmp_dir="$(mktemp -d "${TMPDIR:-/tmp}/compact-string-validation.XXXXXX")"
trap 'rm -rf "$tmp_dir"' EXIT

swiftc -parse-as-library "$script_dir/validate-compact-strings.swift" -o "$tmp_dir/validator"
TOOLS_DIR="$script_dir" "$tmp_dir/validator"
```

First run: 211 findings across 26 languages. That number is a work-list, ordered and diffable, not a failure.

## Generation 2: Measuring What Actually Renders

Character counts are a bad proxy. `Ω` and `l` are both one character. Cyrillic is wider than Latin at the same count. And a character budget cannot express the difference between a 13pt regular description and an 11pt semibold uppercased title in the same card.

The second validator measures real rendered widths with AppKit text measurement on the desktop, with macOS SF Pro standing in for iOS SF Pro:

```swift
static func width(_ string: String, _ size: CGFloat, weight: NSFont.Weight = .regular) -> CGFloat {
    let font = NSFont.systemFont(ofSize: size, weight: weight)
    return ceil(NSAttributedString(string: string, attributes: [.font: font]).size().width)
}
```

The budget side is the part to copy. Instead of a hand-picked number, it re-derives the layout from the grid formula the SwiftUI view uses:

```swift
static let deviceWidth: CGFloat = 375        // smallest supported width
static let gridOuterPadding: CGFloat = 32
static let gridSpacing: CGFloat = 10
static let gridMinimumColumn: CGFloat = 165  // adaptive grid minimum
static let cardPadding: CGFloat = 32
static let iconAllowance: CGFloat = 36
static let headroom: CGFloat = 0.95          // proxy-font margin

static var descriptionBudget: CGFloat {
    let available = deviceWidth - gridOuterPadding
    let columns = floor((available + gridSpacing) / (gridMinimumColumn + gridSpacing))
    let column = (available - (columns - 1) * gridSpacing) / columns
    return column - cardPadding
}

static var titleBudget: CGFloat { descriptionBudget - iconAllowance }
static var passBar: CGFloat { descriptionBudget * headroom }
```

That yields 134.5pt for descriptions and 98.5pt for titles and subtitles. Because the formula mirrors the view, a change to spacing or column minimum is a one-line change in the tool, not a re-guess of every budget. The 5% headroom accounts for the proxy font. Desktop SF Pro is not the real thing, so results land in three buckets rather than two: `OK`, `MARGIN` (inside the 5% band, needs device verification), and `OVER`.

### Worst-Case Format Arguments

Most constrained strings are format templates, not literals. `Sunrise at %@.` has no width until you fill it in. Measuring the template is meaningless, and measuring it with a convenient argument is worse, because it silently passes. So the tool synthesizes the widest legal argument for each placeholder:

```swift
// Widest clock string for this locale (both 12h and 24h are measured)
static func worstTime12(_ language: String) -> String {
    formattedDate(language, pattern: "h:mma", hour: 12, minute: 59)
        .lowercased(with: locale(language))
}

// Widest noun that can fill a precip template
static func worstPrecipNoun(_ language: String) -> String {
    ["Rain", "Snow", "Sleet", "Hail", "Precip"]
        .compactMap { catalogValue($0, language) }
        .max(by: { width($0, 13) < width($1, 13) }) ?? "Rain"
}
```

The catalog reader does the same for plurals. When an entry has plural variations rather than a single string unit, it returns the widest variant, not the `other` case:

```swift
if let plural = (localization["variations"] as? [String: Any])?["plural"] as? [String: Any] {
    let values = plural.values.compactMap {
        (($0 as? [String: Any])?["stringUnit"] as? [String: Any])?["value"] as? String
    }
    return values.max(by: { width($0, 13) < width($1, 13) })
}
```

The last category of argument matters most. Many of the widest strings in the app are not in the app at all. Level names ("Very Unhealthy"), advisory phrases ("Health effects possible."), wind bearings, and composed pollen phrases are all returned by our API, localized on the server. So the tool reads the server repo's locale files directly, converting YAML to JSON in the wrapper and passing a directory path in:

```bash
for yml in "$web_dir"/config/locales/*.yml; do
  lang="$(basename "$yml" .yml)"
  ruby -ryaml -rjson -e 'puts JSON.generate(YAML.safe_load(File.read(ARGV[0])))' \
    "$yml" > "$web_json_dir/$lang.json"
done
web_head="$(git -C "$web_dir" rev-parse --short HEAD)"
```

The report records the server checkout's commit SHA and warns when it differs from that repo's main branch, so a stale baseline announces itself. If the checkout is missing, the tool degrades to client-key coverage with a warning instead of failing. Rows built from synthesized rather than real values (temperatures, precip amounts, wind units) are tagged `[estimate]`, so a reader knows which findings are inferences.

### The Committed Report

The tool writes a markdown file that is checked in:

```markdown
## Summary

- Rows measured: 1620 (27 languages)
- Over budget at default type size: **483**
- Inside margin (127.8-134.5pt band): 78
- Over budget at the xxLarge cap: 674

| Card | Slot | Language | Width | Verdict | Source | Rendered |
|---|---|---|---|---|---|---|
| AQI | description | de | 242/134pt | OVER | `server:aqiLevelPhrase` | Gesundheitliche Auswirkungen moeglich. |
| AQI | description | en | 144/134pt | OVER | `server:aqiLevelPhrase` | Health effects possible. |
| AQI | subtitle | it | 134/98pt | OVER | `server:aqiLevelName` | Molto Insalubre |
```

Committing generated output feels wrong until you use it once. It buys three things:

- **A work-list.** Sorted worst-first per card, it tells the copy pass what to fix and in what order.
- **A diff.** Re-run the tool on a branch and `git diff` shows exactly which rows moved. That is the review artifact for a translation PR.
- **A baseline.** The report is the definition of "no worse than before."

It also surfaced things nobody was looking for. Stat card *titles* truncate today: Vietnamese `Chất lượng không khí` renders at 141pt in a 98pt slot, with eight more languages over on the same key. English itself fails 9 rows. And a `.capitalized` call in the client was title-casing Spanish level names mid-sentence.

## The Server Loop

Of the 483 over-budget rows, 140 came from server-owned strings. Because those phrases live in the API's locale files rather than the app bundle, fixing them is a content change that deploys: no App Store review, no version gate, no waiting for users to update. A truncation bug became a copy edit.

The server-side pass shortened 300+ locale values across 22 languages under one rule:

> **The dual-surface rule:** a shortened value must still read as natural prose on the app's detail screen and on the web product - not merely fit the card.

This is what stops "make it fit" from degrading into telegraphese. A pressure trend name has to work as a standalone card label and inside a sentence:

```yaml
# before -> after, es
pressure:
  trend_ext_name:
    falling-quickly: "Cae rápido"   # was "Bajando rápido"
    falling: "Bajando"
```

`falling-quickly` and `falling` are adjacent steps in the same scale, so the shortened value also has to stay lexically distinct from its neighbor. That is the generation-1 scale rule showing up on the server side.

Verification closed the loop: point the client's width tool at the server branch and re-run. Server-string findings dropped from 140 to 74.

The remaining 74 are adjudicated keeps, rows where no natural short form exists, recorded explicitly:

- Composed two-item pollen phrases stay over in ~10 languages, because the joined nouns alone approach the budget. That one moves back to the client as a layout change (show the dominant type only).
- Thai wind bearings were deliberately left unabbreviated, because the local convention writes them out and abbreviating damages the prose surface.
- Indonesian air quality names depart from the official band terminology to fit an 18pt subtitle, a documented trade rather than an oversight.

Writing keeps down in the same artifact as the findings is what makes the report safe to gate on later. A row that stays over budget forever is fine as long as somebody decided that on purpose.

## Results

- 1,620 rows measured at real rendered widths across 27 languages, 483 of them over budget at the default type size, after a first character-budget run of 211 findings that also caught two live legend bugs in shipping charts
- 300+ server locale values shortened across 22 languages and deployed as content, with zero app updates for the server-owned half of the fix
- Server-string findings down from 140 to 74, every remainder adjudicated and recorded
- The trade-off: desktop SF Pro stands in for the device font, so rows in the `MARGIN` band still need verification on a device

## Lessons Learned

- **Ship the cheap proxy first, then replace it.** Character counts are dependency-free and found two real bugs. Real widths came second, once the proxy's blind spots were known.
- **Derive budgets from the layout formula, not from taste.** Hard-coded budgets rot the moment somebody touches the view. Copying the grid arithmetic makes a spacing change a one-line edit.
- **A format template has no width.** Every placeholder needs a worst-case argument: widest plural variant, widest clock format for the locale, widest enumerated noun, longest real server value.
- **Audit first, gate later, record the keeps.** A validator that fails on day one with 211 findings gets disabled on day two. Exit 0 while the backlog exists, write down each row left over on purpose, then make the flip its own change.
- **Where a string lives decides how its bugs get fixed.** Level names and advisory phrases in the API instead of the app bundle looked like a normalization choice. It made a whole class of UI-fit bugs fixable by deploy.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and tools, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the Spanish QA report instead of a generic loop, the title is one clause, the registry and report sections each say their part once, Results holds what changed and the trade-off, and Lessons Learned went from eight rules to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
