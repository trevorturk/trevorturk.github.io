---
layout: post
title: "Measure the String Before You Translate It"
date: 2026-07-29 08:30:00 -0600
summary: "A small Swift script measures every localized string at its real font and compares it to the room the layout gives it. The report it writes became a work-list across 27 languages, and most of the fixes shipped as server content with no app update."
tags: [swift, ios, localization, i18n, tooling]
---

## The Problem

Customer QA reported cut-off text in the small stat cards of the [Hello Weather](https://helloweather.com) iOS app, in Spanish. The obvious fix was to shorten the Spanish strings. That's the loop every localized app runs: ship a screen that fits in English, translate it into 26 more languages, wait for a support ticket, fix one string, ship an app update, repeat.

The loop is slow because nothing in our tools knows how wide a string will be. The localization catalog stores text. The SwiftUI layout works out widths at runtime, on a device, in one font at one Dynamic Type size. The two never meet until a person looks at a screenshot.

The same string breaks in more than one place. If `Air Quality` overflows a stat card title in Vietnamese, it overflows in the widget too, because it's the same catalog key in the same kind of slot. And some strings come from the server, like air quality level names and advisory phrases. Fixing one word there meant an app release for a change that wasn't in the app. So instead of fixing the Spanish strings, we went looking for a way to find every string in every language that wouldn't fit, before anyone reported it.

## The Solution

Three parts, and none of them needs the app running:

1. **A JSON file, checked in,** that lists which catalog keys render in tight slots and how much room each one gets
2. **A Swift script with no dependencies** that reads that file and the localization catalog and measures every string
3. **A markdown report, also checked in,** that works as the work-list, the diff, and the baseline

The tool starts in audit mode: it reports what it finds and exits 0, because there's a backlog to work through. Once the backlog is gone, it flips to gate mode and fails the run instead.

We built it twice. The first version counted characters. The second measures rendered widths, because counting characters missed things.

## Generation 1: A Character-Budget Registry

The registry is a plain JSON file in the repo:

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

The registry holds 74 keys across six slot types and 14 scale groups. It records two things no linter could work out on its own. The first is where each key renders. A key has a budget because of the slot it lands in, and one key can land in several, so each entry names its slots. A reviewer can then ask "is 17 characters really the stat title budget?" without reading layout code. The second is the scale groups. A scale is a set of labels that appear together in one chart legend, and its rule has nothing to do with length: within a scale, every language's values must all be different from each other. A translator working one key at a time can't see that two English words map to the same word in their language.

The scale rule found two shipping bugs on the first run:

- The Czech cloud legend rendered the same word for both `Cloudy` and `Overcast`
- The Russian UV legend rendered the same word for both `High` and `Max`

Two swatches with the same label and different colors, in a chart that had shipped. Measuring widths would never have caught those.

The tool is about 90 lines of Foundation. It reads the registry and the localization catalog as plain JSON, with no dependency on the app and no test target:

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

A bash wrapper compiles it into a temp directory and runs it, so the whole thing is one command, `./tools/validate-compact-strings`, with no build system involved:

```bash
#!/usr/bin/env bash
set -euo pipefail
script_dir="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
tmp_dir="$(mktemp -d "${TMPDIR:-/tmp}/compact-string-validation.XXXXXX")"
trap 'rm -rf "$tmp_dir"' EXIT

swiftc -parse-as-library "$script_dir/validate-compact-strings.swift" -o "$tmp_dir/validator"
TOOLS_DIR="$script_dir" "$tmp_dir/validator"
```

The first run gave 211 findings across 26 languages. That number is a work-list, not a failure.

## Generation 2: Measuring What Actually Renders

Character counts are a rough stand-in for width. `Ω` and `l` are both one character. Cyrillic is wider than Latin at the same count. And a character budget can't tell a 13pt regular description from an 11pt semibold uppercased title in the same card.

The second version measures real rendered widths with AppKit on the Mac, with macOS SF Pro standing in for iOS SF Pro:

```swift
static func width(_ string: String, _ size: CGFloat, weight: NSFont.Weight = .regular) -> CGFloat {
    let font = NSFont.systemFont(ofSize: size, weight: weight)
    return ceil(NSAttributedString(string: string, attributes: [.font: font]).size().width)
}
```

The budget is the part to copy. Instead of a hand-picked number, the tool works it out from the same grid formula the SwiftUI view uses:

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

That gives 134.5pt for descriptions and 98.5pt for titles and subtitles. The formula matches the view, so if someone changes the spacing or the column minimum, the tool changes in one line and nobody re-guesses every budget. The 5% headroom is there because the Mac font isn't the real one. So each row lands in one of three buckets instead of two: `OK`, `MARGIN` (inside the 5% band, so check it on a device), and `OVER`.

### Worst-Case Format Arguments

Most of the strings in tight slots are format templates, not fixed text. `Sunrise at %@.` has no width until you fill it in. Measuring the template tells you nothing. Measuring it with whatever value is handy is worse, because it passes and you learn nothing. So the tool builds the widest value that could really appear for each placeholder:

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

The catalog reader does the same for plurals. When an entry has plural forms instead of one string, it returns the widest form, not the `other` case:

```swift
if let plural = (localization["variations"] as? [String: Any])?["plural"] as? [String: Any] {
    let values = plural.values.compactMap {
        (($0 as? [String: Any])?["stringUnit"] as? [String: Any])?["value"] as? String
    }
    return values.max(by: { width($0, 13) < width($1, 13) })
}
```

The last kind of placeholder matters most. Many of the widest strings in the app aren't in the app at all. Level names like "Very Unhealthy", advisory phrases like "Health effects possible.", wind bearings, and pollen phrases all come from our API, translated on the server. So the tool reads the server repo's locale files directly. The wrapper converts them from YAML to JSON and passes the directory in:

```bash
for yml in "$web_dir"/config/locales/*.yml; do
  lang="$(basename "$yml" .yml)"
  ruby -ryaml -rjson -e 'puts JSON.generate(YAML.safe_load(File.read(ARGV[0])))' \
    "$yml" > "$web_json_dir/$lang.json"
done
web_head="$(git -C "$web_dir" rev-parse --short HEAD)"
```

The report records the commit SHA of the server checkout and warns when it differs from that repo's main branch, so you can tell when the baseline is stale. If the checkout is missing, the tool checks only the app's own keys and prints a warning instead of failing. Temperatures, precip amounts, and wind units have no catalog value, so the tool invents one, and those rows are tagged `[estimate]` so a reader knows which findings are guesses.

### The Committed Report

The tool writes a markdown file, and we check it in:

```markdown
## Summary

- Rows measured: 1620 (27 languages)
- Over budget at default type size: **483**
- Inside margin (127.8-134.5pt band): 78
- Over budget at the xxLarge cap: 674

| Card | Slot | Language | Width | Verdict | Source | Rendered |
|---|---|---|---|---|---|---|
| AQI | description | de | 242/134pt | OVER | `server:aqiLevelPhrase` | Gesundheitliche Auswirkungen möglich. |
| AQI | description | en | 144/134pt | OVER | `server:aqiLevelPhrase` | Health effects possible. |
| AQI | subtitle | it | 134/98pt | OVER | `server:aqiLevelName.capitalized` | Molto Insalubre |
```

Checking in generated output feels wrong until you've used it once. It gives us three things:

- **A work-list.** It's sorted worst-first per card, so whoever is shortening strings knows what to fix and in what order.
- **A diff.** Run the tool on a branch and `git diff` shows which rows moved. That's what a reviewer reads on a translation PR.
- **A baseline.** The report defines "no worse than before."

It also turned up things nobody was looking for. Stat card *titles* truncate today. Vietnamese `Chất lượng không khí` renders at 141pt in a 98pt slot, and eight more languages are over on the same key. English fails 9 rows of its own. And a `.capitalized` call in the app was title-casing Spanish level names in the middle of a sentence.

## The Server Loop

Of the 483 over-budget rows, 140 came from server strings. Those live in the API's locale files, not in the app, so fixing them is a deploy: no App Store review, no version check, no waiting for users to update. Each of those 140 was a copy edit.

The server pass shortened 300+ locale values across 22 languages under one rule:

> **The dual-surface rule:** every value must still read as natural prose on the iOS detail screens and the web product, not just fit the card.

The rule keeps "make it fit" from turning every phrase into a clipped fragment. A pressure trend name has to work on its own as a card label and inside a sentence:

```yaml
# before -> after, es
pressure:
  trend_ext_name:
    falling-quickly: "Cae rápido"   # was "Bajando rápido"
    falling: "Bajando"
```

`falling-quickly` and `falling` are neighbors in the same scale, so the shorter value also has to stay a different word from the one next to it. That's the scale rule from the first version, now applied on the server.

To check the pass, we pointed the app's width tool at the server branch and ran it again. Server-string findings dropped from 140 to 74.

The remaining 74 are rows we decided to keep, because there's no natural short form. Each one is written down with its reason:

- Two-item pollen phrases stay over in ~10 languages, because the two nouns joined together already come close to the budget. That one got fixed on the server the same day, by naming only the dominant pollen type in the phrase, which cleared all 17 over-budget pollen rows.
- Thai wind bearings stay spelled out, because that's the local convention and abbreviating them reads badly in a sentence.
- Indonesian air quality names don't use the official band terms, because those don't fit an 18pt subtitle. That trade is written down, so it doesn't look like a mistake.

We can gate on the report later because the keeps sit in the same file as the findings. A row can stay over budget forever as long as someone decided that on purpose.

## Results

- 1,620 rows measured at real rendered widths across 27 languages, with 483 over budget at the default type size. The first character-count run had 211 findings and caught two legend bugs in shipping charts
- 300+ server locale values shortened across 22 languages and deployed, with no app update for that half of the fix
- Server-string findings down from 140 to 74, and each of the 74 recorded with a reason
- The trade-off: the Mac font stands in for the device font, so rows in the `MARGIN` band still need a check on a device
- Since then: the tool flipped to gate mode on 2026-07-31, with the floor moved from 375pt to 390pt, the narrowest iPhone still sold. In August 2026 the standalone tools moved into the unit-test target as `WidthReportTests`, run through `bin/width-reports`, and now measure with the real iOS font in the app-hosted test host. The recorded baselines matched the Mac results row for row across 1,755 rows, so the `MARGIN` caveat went away without ever having changed a verdict

## Lessons Learned

- **Ship the cheap stand-in first, then replace it.** Character counts need no dependencies and found two real bugs. Real widths came second, once we knew what counting missed.
- **Work budgets out from the layout formula, not by guessing.** A hard-coded budget goes stale the moment someone touches the view. If the tool copies the grid arithmetic, a spacing change is a one-line edit.
- **A format template has no width.** Every placeholder needs its widest realistic value: the widest plural form, the widest clock string for the locale, the widest noun from the list, the longest real server value.
- **Audit first, gate later, write down the keeps.** A check that fails on day one with 211 findings gets turned off on day two. Exit 0 while there's a backlog, record each row left over on purpose, then flip to gate mode in its own change.
- **Where a string lives decides how its bugs get fixed.** Putting level names and advisory phrases in the API instead of the app looked like a tidy-up. It meant a whole class of layout bugs could be fixed with a deploy.

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and tools, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the Spanish QA report instead of a generic loop, the title is one clause, the registry and report sections each say their part once, Results holds what changed and the trade-off, and Lessons Learned went from eight rules to five. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The registry (74 keys, six slots, 14 scales), both validators, the 211 and 1,620/483/78/674 counts, the 134.5pt and 98.5pt budgets, the legend bugs, the 140-to-74 server drop, and the 308-value, 22-language server pass all matched the iOS and web repositories at the commits the post describes. Corrected: the language count in the opening (26 other languages, not 25) and the summary (the work-list spans 27); two report rows restored to the committed baseline ("möglich", and the subtitle source `server:aqiLevelName.capitalized`); the dual-surface rule now uses the PR's wording; the pollen residual now records what happened (fixed on the server the same day by rendering only the dominant type, not a client layout change); and Results gains a note that the gate flipped on 2026-07-31 at a 390pt floor and that the standalone tools were later folded into `WidthReportTests` measuring on the real iOS font.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 2, run after batch 1 (#68) merged. The prose was redrafted from a short plain-language explanation of the work: "we" and contractions throughout, one fact per sentence, and ordinary words in place of "proxy", "adjudicated keeps", "telegraphese", "lexically distinct", "prose surface", and "surfaced"; the validator, tool, and script are now one thing, "the tool", except where the file name or the later `WidthReportTests` move names it. Judgment calls: "zero app updates" became "no app update"; the quoted examples of server strings ("Very Unhealthy", "Health effects possible.") stay because they define the terms; and the summary was rewritten as two sentences with the same facts. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"
