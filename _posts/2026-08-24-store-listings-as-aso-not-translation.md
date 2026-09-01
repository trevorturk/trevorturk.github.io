---
layout: post
title: "Store Listings Are ASO, Not Translation"
date: 2026-08-24 09:50:00 -0600
summary: "Per-locale store listings kept in git and pushed to App Store Connect as reviewed diffs, written as App Store Optimization per market rather than a literal translation of the English."
tags: [workflow, localization, i18n, app-store]
---

## The Problem

The app shipped in 26 languages and the App Store did not move. Search ranking in Germany stayed flat, and the Japanese storefront still showed English. When we pulled the live listing down to see why, the app info and every recent version carried exactly one localization, and every other storefront fell back to it.

The store listing is not part of the app. It lives in App Store Connect, in its own data model: name, subtitle, keywords, description, promotional text, and the "What's New" notes on each version. None of it comes from the string catalog, so translating the binary never touches it. Localizing [Hello Weather](https://helloweather.com) into 26 languages meant standing up roughly 26 storefront listings that did not exist yet, not editing ones that did.

The tempting move treats that as a translation job: hand the English listing to a translator and paste the results into 26 fields. A store listing is a ranked, indexed marketing surface, and each storefront ranks it against search terms typed by people in a different market. A subtitle that is the best English phrasing, translated word for word, lands in a storefront optimized for nothing anyone there searches for. The words come back correct and the listing still loses, because correctness was never the metric.

This is the same shape as the [App Store pricing problem](/appstore-pricing/): a field that looks like one value is really a per-territory decision, and doing it by hand in the web UI is slow, unreviewable, and wrong in ways you cannot see.

## The Solution

Treat store metadata as an App Store Optimization program run per storefront. Three parts:

1. **A per-locale file in version control is the source of truth** for every listing field, one file per storefront.
2. **A CLI pulls, diffs, and pushes** those files against App Store Connect, so every change to a live listing is a reviewed diff rather than a form edit nobody saw.
3. **Two human gates stay manual and nothing else does.** The English master is signed off before any fills are drafted, and every push goes out as a reviewed diff.

The listings are model-drafted, which is only safe because both gates and the whole change history ride git. Everything a human needs to catch a bad listing is a `git diff` or a `push` preview away.

### The listing is marketing copy, not a string table

App strings are *translated*; the store listing is *rewritten per market*. Miss that distinction and you waste the highest-value field on the page. The subtitle is, after the name, the second-most-weighted field the App Store indexes for search, so a literal rendering of an English tagline throws that weight away. Each storefront instead gets a subtitle and keyword field chosen for the terms people there actually type. The name is localized where a market expects it. The description carries no search weight and is pure conversion, so it is re-flowed from the approved English.

The source of truth is one YAML file per storefront under `config/appstore/metadata/`, 28 of them: `en-CA.yml` as the English master, then `de-DE.yml`, `ja.yml`, and the rest. Each holds the app-info and version fields for that storefront, with the limits sitting next to the copy so they shape it instead of surprising it at push time:

```yaml
# config/appstore/metadata/de-DE.yml — one storefront.
# Placeholder copy; the CLI checks each field against these limits before a push.
app_info:
  name: "PlaceholderApp"                    # 30 chars, submission-gated, localizable
  subtitle: "Das Wetter, klar und schnell"  # 30 chars, keyword-bearing, chosen for this market
  privacy_policy_url: "https://example.com/privacy"
version:
  keywords: "wetter,vorhersage,regenradar,pollen"  # 100 BYTES not chars; commas, no spaces
  description: "Eine klare, schnelle Wettervorhersage."  # 4,000 chars, no search weight, pure conversion
  promotional_text: "Neu: Pollen und Regenradar in jeder Stadt."  # 170 chars, editable anytime
  whats_new: "Schnelleres Radar und kleinere Fehlerbehebungen."   # 4,000 chars, per-release, localized
  support_url: "https://example.com/support"
  marketing_url: "https://example.com"
```

`keywords` is measured in bytes, so a multibyte language gets far fewer keywords per storefront than English does. Putting the file in git is the quiet win. A listing becomes a reviewable artifact with a history: who changed the German subtitle and when, a diff between two versions, a revert of a bad one. The web UI offers none of that.

### Every change to a live listing is a reviewed diff

A listing edited in a web form leaves no history, no review, and no way to tell whether the repo and the store agree. The CLI's three verbs have the same shape as `git`, in the order a release runs them:

```bash
bin/appstore metadata pull                 # overwrite local files from the live listing
bin/appstore metadata diff de-DE fr-FR     # show local vs. live for two storefronts
bin/appstore metadata push                 # preview every pending change (writes nothing)
bin/appstore metadata push --apply         # write the reviewed changes to App Store Connect
```

`pull` overwrites the local files from what is live, so a `git status` afterward shows whether the repo and the store have drifted. `diff` shows how the local files differ. `push` previews by default and writes only with `--apply`.

The manager behind those verbs is small. It maps YAML keys to App Store Connect attributes and, on `--apply`, updates the storefronts that exist and creates the ones that do not. The review trick is in `diff`: a short field prints old and new values inline, but a long field prints only a character count, because a 4,000-character description dumped in full twice is not reviewable. The block runs against a stubbed client on its own. The create path in `push` is what stood up the roughly 26 net-new listings:

```ruby
require "yaml"

FIELDS      = %i[name subtitle keywords description promotional_text whats_new].freeze
LONG_FIELDS = %i[description whats_new].freeze

# Stands in for the App Store Connect REST client. In production each method
# calls the ASC API; here it returns canned rows and logs writes so the block
# runs on its own.
class StubConnectClient
  def initialize
    @live = {
      "en-CA" => { id: "en", name: "PlaceholderApp", subtitle: "Weather, simply",
                   keywords: "weather,forecast,rain", description: "A clean forecast.",
                   promotional_text: "", whats_new: "First release." }
    }
  end

  def localizations = @live
  def locale_id(locale) = @live.dig(locale, :id)

  def update(id:, attributes:)  = puts("  PATCH localizations/#{id} → #{attributes.keys.join(', ')}")
  def create(locale:, attributes:) = puts("  POST localizations locale=#{locale} → #{attributes.keys.join(', ')}")
end

class MetadataManager
  def initialize(local:, client: StubConnectClient.new)
    @local  = local            # { locale => { field => value } }
    @client = client
  end

  # Overwrite local files from what is live, so a later `git status` reveals drift.
  def pull = @client.localizations.transform_values { |row| row.reject { |k, _| k == :id } }

  # Short fields show old → new; long fields show only a char count, so a
  # 4,000-char description never floods the review.
  def diff
    live = @client.localizations
    @local.each_with_object({}) do |(locale, fields), changes|
      lines = FIELDS.filter_map do |field|
        was = live.dig(locale, field).to_s
        now = fields[field].to_s
        next if was == now

        if LONG_FIELDS.include?(field)
          "#{locale} #{field}: #{was.length} → #{now.length} chars"
        else
          "#{locale} #{field}: \"#{was}\" → \"#{now}\""
        end
      end
      changes[locale] = lines unless lines.empty?
    end
  end

  # Preview by default; write only with apply: true.
  def push(apply: false)
    live = @client.localizations
    diff.each do |locale, lines|
      lines.each { |line| puts((apply ? "PUSH " : "would push ") + line) }
      next unless apply

      changed = @local[locale].select { |f, v| v.to_s != live.dig(locale, f).to_s }
      if (id = @client.locale_id(locale))
        @client.update(id: id, attributes: changed)
      else
        @client.create(locale: locale, attributes: @local[locale])   # net-new storefront
      end
    end
  end
end

local = {
  "en-CA" => { name: "PlaceholderApp", subtitle: "The forecast, your way",
               keywords: "weather,forecast,radar", description: "A clean, fast forecast.",
               promotional_text: "New: radar in every city.", whats_new: "Faster radar." },
  "de-DE" => { name: "PlaceholderApp", subtitle: "Das Wetter, klar und schnell",
               keywords: "wetter,vorhersage,regenradar", description: "Eine klare Vorhersage.",
               promotional_text: "Neu: Regenradar.", whats_new: "Schnelleres Radar." }
}

MetadataManager.new(local: local).push(apply: false)   # de-DE has no live row: previews as net-new
```

Run it and `de-DE` previews as a full create while `en-CA` shows only its changed fields inline. The two human gates sit around this machinery. The first exists because translated copy is keyed off the English: approve the master once, and the 26 fills are drafted against final wording. The second gate is the `push` preview itself, read and approved before `--apply` touches a live storefront.

### Field limits are release-cadence rules, not just size caps

Each limit also decides *when* a field can change, so encoding it up front is cheaper than a rejected submission. Promotional text (170 characters) is the only field editable on a live listing at any time, so it carries a seasonal or between-release line without a submission. Name and subtitle are submission-gated, so a subtitle rotation has to ride a release train. Description, keywords, and What's New open and close with it. Copy written against those rules ships on the right cadence instead of waiting for a window that already closed.

### Release notes and screenshots ride the same rails

Two more surfaces are localizable, and both fold into this system. The first is release notes. The in-app changelog is already fully localized: every entry's date and body go through the same `localized(_:)` helper, so the "what changed" screen renders in the reader's language. A trimmed but complete version of that view:

```swift
import SwiftUI

// Resolves against the app's own language setting, not the device language.
// See the localization write-up for how the helper is wired.
func localized(_ key: String.LocalizationValue) -> String {
    String(localized: key)
}

struct ChangelogEntry: Identifiable {
    let date: String
    let body: String
    var id: String { date }
}

let changelog: [ChangelogEntry] = [
    ChangelogEntry(
        date: localized("August 18, 2026"),
        body: localized("""
        - Hello Weather now speaks 27 languages! Choose yours in Settings → Language.
        - Your location's weather now updates automatically when you travel to a new city.
        """)
    )
]

struct ChangelogView: View {
    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 20) {
                ForEach(changelog) { entry in
                    VStack(alignment: .leading, spacing: 6) {
                        Text(entry.date).font(.headline)
                        Text(entry.body)
                    }
                }
            }
            .padding()
        }
    }
}
```

Notice that both the date and the body run through `localized(_:)`, so the entry is copy like any other rather than a hardcoded English string. The store's "What's New" is that same content on a different surface, so it lives in the metadata YAML and pushes on the same rails. At release time the tooling sets the primary-locale What's New straight from the changelog entry, whose `- Verb …` wording already fits the field. One release note, written once, reaches the in-app screen and 28 storefronts.

The second surface is screenshots. A store tile's headline is a localized layer composited over the captured screenshot at build time, never baked into the design asset. A designer builds the frame, ground, and typography once; a headline rewrite or a new language re-renders the text layer without going back to design.

One sequencing decision made all of this safe. The languages shipped quietly first, behind an opt-in picker with a translation-feedback button, and the marketing that points people at a storefront waited until customers had exercised the language in the wild. The [support inbox](/support-inbox-as-telemetry/) is the QA channel for that wait.

## Results

- **28 per-locale listing files under version control**, so a listing can be diffed, reviewed, and rolled back. The 26 storefronts that had been falling back to one primary-language listing were stood up through the CLI's create path.
- **Every push to a live listing is a reviewed diff.** `pull` makes drift between the repo and App Store Connect visible, and `--apply` is opt-in against a dry-run default.
- **The release note is written once and lands in two places.** The in-app changelog and the per-locale What's New draw from one source, so they can no longer disagree.
- **A paid post-editing vendor was retired** once the model plus the written rules matched its quality. What stays manual is the two gates: signing off the English master and reading the push preview.

## Lessons Learned

- **Translate what is read; rewrite what is ranked.** A field scored by search is an optimization problem per market, and a faithful translation can be correct and still lose.
- **Gate on the source text before fanning out.** Every derived value is keyed to the exact source, so an edit after the fills exist discards the whole batch.
- **Measure a limit in the platform's unit.** A byte cap on keywords favors native terms that fit over a translated English list that will not.
- **Declared is not launched.** Market a storefront only after real users have exercised its language, and treat the support inbox as the test suite until then.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the flat German ranking and the English Japanese storefront instead of on how much text the app carries, the title dropped from twelve words to six, and the market-versus-translation, source-gate, and byte-limit arguments are each stated once, in the body or in Lessons Learned but not both. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
