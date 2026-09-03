---
layout: post
title: "Never Trust the Vendor's Language List"
date: 2026-07-29 08:40:00 -0600
summary: "We sent real requests to ten upstream data sources in all 27 supported languages and found four different ways a vendor fails that its documentation doesn't mention."
tags: [api-design, localization, i18n, ruby, testing]
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

---

## How This Post Was Made

**Prompt 1:** "it's been a while since we added any blog posts, see recent work in the ~/Code/helloweather projects, dispatch opus agents to search for interesting stuff that we've done since the last blog post, perhaps one or more agents per repo, then review and consider and come up with a proposed list of blog posts we might consider."

**Prompt 2:** "draft posts for [the approved shortlist] -- create one pr for the repo main / skills update we just did, then one pr per post for the approved list"

Research by one Claude agent per repo mining git history since the previous post; this draft was written by a dedicated agent from that research plus the underlying commits and skill files, then reviewed before publishing.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The title lost its subtitle, the post now opens on the one-line exclusion list nobody had checked, the failure-mode commentary was split into shorter sentences, Results dropped the bullets that restated the Solution, and Lessons Learned went from seven rules to four. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.

**Fact check (2026-09-01):** The owner asked, "1) dispatch research into the ~/Code/helloweather repos to validate the posts' content, for example checking the StoreKit code we shared is correct. 2) fix the "Pre-existing oddities" using your judgement, and feel free to make "judgment calls" as you see fit -- this is a blog meant to be authored by AI and is expected to lean on AI model judgement calls, advancements in model capabilities may prompt future editing/rewriting sessions, and for each one I'll want them to be driven autonomously." One Claude Fable 5.1 agent checked this post's code excerpts, numbers, dates, and quoted rules against the source repositories. The probe date, the four failure modes, the nine-language exclusion list, the three remapped codes, and the `language` field and its test all matched the 2026-07-20 source-language PR and the api-localization skill. Corrections: the source count is eleven, with ten carrying the docs-derived declarations that make the 270-cell matrix; that matrix dated from the end of 2025, so "for years" became "seven months"; "every source's declaration was wrong" became "nine of the ten needed a correction," since one English-only source was left unchanged; the excerpts now use `supported_source_languages` and `else super`, the names the code took the day after the probe when `supported_languages` was renamed and the `language_code` helper was deleted; and the condition-code map excerpt gained the `nil` entry and error message from the real code.

**Rewrite (2026-09-03):** Plain-register pass, pilot for issue #66, after a reader said the posts read like AI. Archive batch 3, run after batch 2 (#69) merged. Prose was redrafted from a short plain-language explanation of the post: "we" and contractions throughout, the bolded rule in The Solution unbolded and reworded ("the same text as the English one" instead of "byte-for-byte"), "byte-inequality" and "wire codes" replaced with ordinary words, the "localization opportunity" line and the "aphorism" closers in Lessons Learned cut, and Mode 1's commentary split so each sentence carries one fact. Judgment calls: the Results bullet's "English-only for years" was left as written, since the 2026-09-01 fact check scoped its "seven months" correction to the docs matrix, not to those two sources; the ISO 639-1 reference stays because it's the standard's name. Code blocks, the table, headings, dates, numbers, and links are unchanged. Prompts, verbatim:

**Prompt 1:** "we got feedback from a reader that our posts are still too AI/slop/wordy, an example and a possible skill to improve are included here, please review and let me know what you think, consider if we could do another big bang rewrite without spending too much of our Fable budget, or we could prep and schedule for when our limits are about to be reset and save in a date-triggered gh issue: I enjoy your ai posts, but man is it wordy :joy: [the reader's quoted paragraph and a link to the SimpleEnglish skill followed; both are in issue #66]"

**Prompt 2:** "agreed, but lets make this into an issue, I just enabled issues, document what your plan is with a new issue, then we can kick it off with the smaller sample, maybe keep going depending on token usage, and the reader can subscribe to the gh issue to track if they like. as usual, please include this prompting in the issue so people can follow along to see "how the sausage is made" if they're interested. oh, and sorry, I think what I'm looking for is less about word counts, and more about "ai speak" as in, here's a bit more slack chatter about this with the reader: I'm kicking off a blog rewrite thing, not 100% sure if I want to do a big bang today tho b/c Fable budgets [10:38 AM]but I'll report back READER [10:39 AM] I'll be curious. Will it be "byte for byte identical" ??? :joy:"

**Prompt 3:** "and the density issue, the quote the reader provided is a perfect "what not to do" example, I think"

**Prompt 4:** "another possible thing to mix into the skill changes would be the ELI5 idea, which I generally like, I often ask AI to ELI5 after dispatching research so I get a human-readable explanation of the why, what, how etc"

**Prompt 5:** "go ahead and kick off the pilot PR"

**Prompt 6:** "perhaps the use of Opus for the writing is a source of the problem? I'm finding Opus to be a bad writer, and Fable 5.1 to be much better. the reader reports: Also I think it's funny that the ai suggestions are still bad. "extracting from the source is what makes the slice trustworthy" Should just be "The slice is trustworthy because it's directly extracted from the source." -- and the "Not every slice can be copied straight out of the source PR" rewrite paragraph is better, but perhaps still somewhat verbose/ai-slop-ish? I wonder if we can do just a bit better, but this does seem like a promishing direction. consider and report back with a recommendation."

**Prompt 7:** "agreed except I wouldn't worry about the word count at all. "wordy" isn't the same thing as "word count" and I think the reader (and my) issue is more to do with the AI style of speaking, which is why we're looking at the ELI5 and SimpleEnglish skill adaptations."

**Prompt 8:** "merge it and start the first batch of ten, then I can check usage, and then we can keep going -- just to check, are you saying the total spend would be ~6M tokens?"

**Prompt 9:** "usage looks fine, merge it and run batch 2"

**Prompt 10:** "usage is fine, please continue -- one more thing -- at the end (or perhaps with future batches?) I'd like to change the "How This Post Was Made" sections in all posts to not have the prompt in the post itself, rather, the prompts should be moved into PR body if editable, or comments, then the "How This Post Was Made" can have the last edit date and a link to the Pull Requests / Prompts -- then there's less cruft at the end for readers that just want to copy paste a post into their agent -- wdyt?"
