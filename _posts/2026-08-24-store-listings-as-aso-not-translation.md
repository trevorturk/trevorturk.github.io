---
layout: post
title: "Store Listings Are ASO, Not Translation"
date: 2026-08-24 09:50:00 -0600
summary: "We keep one App Store listing per storefront in git, push changes as reviewed diffs, and write each one for its market's search terms instead of translating the English."
tags: [workflow, localization, i18n, app-store]
---

## The Problem

The app soft-launched in 26 new languages, and the App Store pages didn't follow. We'd left the listing in English on purpose. When we looked at a few storefronts, the Japanese one showed an English description with no subtitle at all, and Germany and France seemed to have one-word subtitles. Then we pulled the live listing down. The app info and every recent version had one localization, English. The one-word subtitles were the store's own category labels. Every other storefront was falling back to the English.

The store listing isn't part of the app. It lives in App Store Connect as its own set of fields: name, subtitle, keywords, description, promotional text, and the "What's New" notes on each version. None of that comes from the app's string catalog, so translating the app doesn't touch it. Localizing [Hello Weather](https://helloweather.com) into 26 languages meant creating roughly 26 listings that didn't exist yet, not editing ones that did.

The obvious approach is to treat that as a translation job: hand the English listing to a translator and paste the results into 26 sets of fields. But the store indexes the listing for search, and each storefront ranks it against what people in that country type. A subtitle that works well in English, translated word for word, ends up matching nothing anyone there searches for. The words can be right and the listing still ranks badly, because the store scores search matches, not accuracy.

We'd hit the same problem with [App Store pricing](/appstore-pricing/). A field that looks like one value is really a decision per territory, and editing it by hand in the web UI is slow, nobody reviews it, and you can't see what's wrong.

## The Solution

We treat the store listing as App Store Optimization, meaning copy written to rank in each storefront's search, and we run it per storefront. Three parts:

1. **One file per storefront in git** holds every listing field for that storefront.
2. **A command-line tool pulls, diffs, and pushes** those files to App Store Connect, so every change to a live listing is a diff someone read, not a form edit nobody saw.
3. **Two steps stay manual.** We sign off the English master before the model drafts the per-market listings, and we read every push preview before applying it.

A model drafts the listings. That's safe because the change history is in git and a person reads both gates. To catch a bad listing, you run `git diff` or the `push` preview.

### The listing is marketing copy, not a string table

App strings get translated. The store listing gets rewritten for each market. The subtitle is the second-most-weighted search field after the name, so a word-for-word translation of the English tagline wastes it. Each storefront gets a subtitle and keyword list chosen for the terms people there actually type. The name stays the brand name in every storefront so far; whether to localize it is still an open question. The description carries no search weight and only has to persuade someone who's already on the page, so we rework it from the approved English.

Each storefront has one YAML file under `config/appstore/metadata/`, 28 in all: `en-CA.yml` is the English master, then `de-DE.yml`, `ja.yml`, and the rest. Each file holds the app-info and version fields for that storefront. The field limits live in the tool's validator, which runs before any push, so we find out about an over-long field before submission instead of at it. The files themselves have no comments, because `pull` regenerates them. The limits are written in as comments here so you can read them:

```yaml
# config/appstore/metadata/de-DE.yml — one storefront.
# Placeholder copy; the CLI validates each field against these limits before a push.
locale: de-DE
app_info:
  name: "PlaceholderApp"                    # 2–30 chars, submission-gated
  subtitle: "Das Wetter, klar und schnell"  # 30 chars, keyword-bearing, chosen for this market
  privacy_policy_url: "https://example.com/privacy"
  privacy_choices_url:
version:
  whats_new: "Schnelleres Radar und kleinere Fehlerbehebungen."   # 4,000 chars, per-release, localized
  description: "Eine klare, schnelle Wettervorhersage."  # 4,000 chars, no search weight, pure conversion
  keywords: "wetter,vorhersage,regenradar,pollen"  # 100 BYTES not chars; commas, no spaces
  promotional_text: "Neu: Pollen und Regenradar in jeder Stadt."  # 170 chars, editable anytime
  support_url: "https://example.com/support"
  marketing_url: "https://example.com"
```

`keywords` is measured in bytes, so a language with multibyte characters fits far fewer keywords than English does. Keeping the file in git is the part that matters most. A listing gets a history: who changed the German subtitle and when, a diff between two versions, a way to revert a bad one. The web UI has none of that.

### Every change to a live listing is a reviewed diff

The tool has three commands, shaped like `git`, listed here in the order a release runs them:

```bash
bin/appstore metadata pull                 # overwrite local files from the live listing
bin/appstore metadata diff de-DE fr-FR     # show local vs. live for two storefronts
bin/appstore metadata push                 # preview every pending change (writes nothing)
bin/appstore metadata push --apply         # write the reviewed changes to App Store Connect
```

`pull` overwrites the local files with what's live, so `git status` afterward shows whether the repo and the store have drifted apart. `diff` shows how the local files differ from live. `push` previews by default and only writes with `--apply`.

The manager behind those commands is small. It maps YAML keys to App Store Connect attributes and checks every field against the limits. On `--apply`, it updates the storefronts that exist and creates the ones that don't. `diff` prints a short field's old and new values inline, but for a long field it prints only the character counts, because nobody can review a 4,000-character description printed twice. The block below is simplified and runs on its own against a stub client. The real manager splits app-info and version fields across two App Store Connect resources, which we've left out. The create path in `push` is how we stood up the roughly 26 new listings:

```ruby
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
  # `apply` is set once from the CLI flag; every verb previews unless it is true.
  def initialize(local:, client: StubConnectClient.new, apply: false)
    @local  = local            # { locale => { field => value } }
    @client = client
    @apply  = apply
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

  # Preview by default; write only when constructed with apply: true.
  # The real manager runs the length and byte validator on each locale first.
  def push
    live = @client.localizations
    diff.each do |locale, lines|
      puts "Would update #{locale} (new locale)" unless live.key?(locale) || @apply
      lines.each { |line| puts((@apply ? "Updated " : "Would update ") + line) }
      next unless @apply

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

MetadataManager.new(local: local, apply: false).push   # de-DE has no live row: previews as net-new
```

Run it and `de-DE` previews as a new locale with every field listed, while `en-CA` shows only the fields that changed. The two manual gates sit on either side of this. The first is signing off the English master, because every per-market listing is drafted from it. Approve the English once, and the 26 per-market listings are drafted against final wording. The second gate is reading the `push` preview before `--apply` touches a live storefront.

### Each field limit also says when the field can change

Each field has a rule about *when* it can change, not just how long it can be. Promotional text (170 characters) is the only field you can edit on a live listing at any time, so that's where a seasonal line or a between-release note goes. Name and subtitle can only change with a submission, so a new subtitle has to wait for a release. Description, keywords, and What's New change with a release too. If you know that before writing, you put each line in a field that can change when you need it to.

### Release notes and screenshots go the same way

Two more things need localizing: release notes and screenshots. Release notes already go through this system. The in-app changelog is localized: every entry's date and content go through the same `localized(_:)` helper, so the "what changed" screen shows up in the reader's language. Here's a trimmed version of that view, with the Markdown rendering and styling left out:

```swift
import SwiftUI

// Resolves against the app's own language setting, not the device language.
// See the localization write-up for how the helper is wired.
func localized(_ resource: LocalizedStringResource) -> String {
    var resource = resource
    let language = UserDefaults.standard.string(forKey: "language") ?? "en"
    resource.locale = Locale(identifier: language)
    return String(localized: resource)
}

private struct ChangelogEntryData: Identifiable {
    var date: String
    var version: String
    var content: String

    var id: String { "\(date)-\(version)" }

    static let all: [ChangelogEntryData] = [
        ChangelogEntryData(date: localized("August 18, 2026"), version: "v26.6.5", content:
        localized("""
        - Hello Weather now speaks 27 languages! Choose yours in Settings → Language.
        - Your current location's weather now updates automatically when you travel to a new city.
        """)
        )
    ]
}

struct ChangelogView: View {
    var body: some View {
        ScrollView(.vertical, showsIndicators: false) {
            LazyVStack(spacing: 20) {
                ForEach(ChangelogEntryData.all) { entry in
                    VStack(alignment: .leading, spacing: 6) {
                        Text(entry.date).font(.headline)
                        Text(entry.content)
                    }
                }
            }
        }
        .navigationTitle("Changelog")
    }
}
```

Notice that the date goes through `localized(_:)` too, not just the content, so the whole entry is translatable copy rather than a hardcoded English string. The store's "What's New" is the same text in a different place, so it lives in the metadata YAML and pushes with everything else. At release time the tooling copies the changelog entry straight into the primary locale's What's New; the `- Verb …` wording already fits the field. We write one release note and it reaches the in-app screen and 28 storefronts.

Screenshots are designed but not built as of September 2026. When you add a language to a listing, its screenshots default to the primary language's, so screenshots never held up the push. The plan keeps design and localization separate. A designer builds the frame, background, and typography once. The headline on each store image is a localized text layer laid over the captured screenshot at build time, not baked into the design file, so a new headline or a new language re-renders the text without going back to the designer.

One ordering decision made the rest safe. The languages shipped quietly first, behind an opt-in picker with a translation-feedback button. Marketing that points people at a storefront waited until real customers had used the language. The store's language list and the string catalog only say which languages are *declared*, not which ones work, so we announce a language only after people have used it. During that wait, the [support inbox](/support-inbox-as-telemetry/) is where the bugs come in.

## Results

- **28 listing files in git**, one per storefront, so a listing can be diffed, reviewed, and rolled back. The 26 storefronts that had been falling back to English were created through the tool's create path.
- **Every push to a live listing is a diff someone read.** `pull` shows drift between the repo and App Store Connect, and `push` is a dry run unless you pass `--apply`.
- **The release note is written once and lands in two places.** The in-app changelog and the per-locale What's New come from the same text, so they can't disagree anymore.
- **We dropped a paid post-editing vendor** once the model, working from the written rules, matched its quality.

## Lessons Learned

- **Rewrite any field the store ranks by search, one market at a time.** Translating it faithfully gives you correct words that don't match what people there search for.
- **Sign off the source text before drafting the other languages.** Every per-market listing is drafted from it, so an edit afterward means reworking all of them.
- **Check limits in the platform's unit.** The keyword cap is in bytes, so native terms that fit beat a translated English list that doesn't.
- **A declared language isn't a launched one.** Market a storefront only after real users have used its language, and treat the support inbox as the test suite until then.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the flat German ranking and the English Japanese storefront instead of on how much text the app carries, the title dropped from twelve words to six, and the market-versus-translation, source-gate, and byte-limit arguments are each stated once, in the body or in Lessons Learned but not both. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The opening now describes what the storefront check and the first `pull` actually found (an English-only listing, category labels mistaken for subtitles) instead of an unsupported flat German ranking; the limits are described as living in the CLI's validator rather than in the YAML files, which carry no comments, and the YAML excerpt gained the real `locale` and `privacy_choices_url` keys and the 2–30 name range; the Ruby excerpt now takes `apply` in the constructor and prints "Would update" like the real manager, and is labeled as simplified; the Swift excerpt uses the real `LocalizedStringResource` helper, the `ChangelogEntryData` record with `date`, `version`, and `content`, and the changelog's exact wording; the name is described as unlocalized so far, the screenshot compositor as planned rather than built, and the source-gate lesson as reworking the fills rather than discarding them.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 1, run after the pilot (#67) merged. The opening now tells what we found in one step per sentence, "surface" is gone from four places, App Store Optimization is defined where it first appears, the per-market listings are called that throughout instead of "fills", "the CLI" is "the tool", and the Results bullet that repeated the two manual gates was cut. Judgment calls: two `###` headings ("release-cadence rules", "ride the same rails") were reworded in plain words, "ground" in the screenshot plan became "background", and the Lessons bullets lost their maxim form. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"
