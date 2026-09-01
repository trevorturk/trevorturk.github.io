---
layout: post
title: "Server Owns the Sentence"
date: 2026-08-24 09:30:00 -0600
summary: "Localize codes at the edge instead of composing English sentences in the middle, own display punctuation in one server-side rule, and know the direction of every language alias table so an inversion doesn't flip your outbound codes."
tags: [ruby, api-design, localization, i18n]
---

## The Problem

A weather API returns two kinds of text, and only one of them is yours.

The first kind is prose an upstream data source already wrote: a summary sentence for the hour, a daily narrative. When the source supports the requested language, that text arrives localized and you pass it through. The companion post on [probing vendor language support](/probing-vendor-language-support/) is about knowing whether that prose is real or a silent English fallback.

The second kind is text you derive: an air-quality level name, a pressure-trend phrase, a condition-code description. It does not exist until your server composes it. A field that ships as `"Health effects possible."` is a finished English sentence with no seam left to translate at. The client receives prose, not data, and prose in the wrong language cannot be fixed downstream.

The trap is that the English path always works. The app looks fine, and the bill comes due six months later when a customer in another language reports half the forecast screen in English. By then the sentence is composed in a dozen call sites.

At [Hello Weather](https://helloweather.com) the derived fields are localized on the server, in the same request, pinned to the request's language, and the app renders them verbatim. This post is the server side of that arrangement: composing text that is translatable by construction, owning display punctuation in one place, and bridging two independently maintained language lists with an alias table whose direction you understand.

## The Solution

Three rules, each cheap, each with a failure mode that only shows up in a non-English request:

1. **Ship codes, not sentences.** Every descriptive field is a machine code plus, optionally, a localized `_name`/`_phrase` companion resolved through a helper pinned to the request's language. The server never composes an English sentence it cannot later translate.
2. **Bridge language lists in one direction each.** Derive the valid-language set from one source of truth, normalize inbound codes through an alias table, validate by inclusion, and reject the rest. Keep the outbound provider-code table separate from the inbound alias table, because one is the inverse of the other and inversion is lossy.
3. **Own display punctuation universally.** One rule, one concern, every output shape, every consumer. Strip, normalize, terminate (except where a language doesn't terminate), upcase the first letter. Clients render the result byte for byte.

Under all three sits one test discipline: default snapshots are English-only, so a language-dependent change has to assert a non-English language or it ships broken silently.

### Codes, Not Sentences

The architecture rests on one helper. In the class that derives attributes, `t()` resolves an I18n key against the language the request asked for, never `I18n.locale` and never a thread default. The language is a property of the parsed request, threaded through explicitly, so every derived string comes out in one language no matter how concurrently the server is composing. A pressure trend then ships as a stable, language-free code beside a localized label. The client switches on the code for logic and renders the label for display:

```ruby
require "i18n"

class DerivedAttributes
  class NotImplementedError < StandardError; end

  def initialize(language:, pressure_change:)
    @language = language
    @pressure_change = pressure_change
  end

  def to_h
    {
      pressure_trend: pressure_trend,            # stable, language-free
      pressure_trend_name: pressure_trend_name,  # localized for display
    }
  end

  private

  def t(key)
    I18n.t(key, locale: @language.to_sym)
  end

  def pressure_trend
    return "steady" if @pressure_change.abs < 1
    @pressure_change.positive? ? "rising" : "falling"
  end

  # Exhaustive case, raising else. Dynamic interpolation
  # (t("api.pressure.#{pressure_trend}")) would ship the literal string
  # "translation missing: ..." as content for any unclassified value.
  def pressure_trend_name
    case pressure_trend
    when "steady"  then t("api.pressure.steady")
    when "rising"  then t("api.pressure.rising")
    when "falling" then t("api.pressure.falling")
    else
      raise NotImplementedError, pressure_trend
    end
  end
end
```

The field pair is what to notice. A `de` request emits `{"pressure_trend": "falling", "pressure_trend_name": "Fallend"}`, and an `en` request emits `{"pressure_trend": "falling", "pressure_trend_name": "Falling"}`. The code is identical across languages, so client logic never parses prose. Only the companion is translated, at the one seam that holds the language.

The verbose `case` is the safety, not clutter to remove. Dynamic key lookup turns a missing translation into a shipped string: I18n returns `"translation missing: ..."`, which is content rather than an error, and it renders on a customer's screen. The exhaustive `case` with a raising `else` turns the same gap into a failed test, a bug you hear about from CI instead of from a customer. Keep the case structure when you localize; do not simplify it into an interpolation.

### One Language List, Two Directions of Aliasing

The valid-language set is derived once, at boot, from the framework's own locale registry, `Rails.application.config.i18n.available_locales`, so there is no second hardcoded list to drift against. Add a locale file and the language becomes valid. The 27 codes (`en cs da de el es fi fr hi hu id it ja ko nb nl pl pt ro ru sv th tr uk vi zh zh_TW`) are exactly the set of files on disk. Validation is then one line, `validates :language, inclusion: { in: LANGUAGE }`, and an unknown code is a clean `400`.

Between the wire and that validation sits normalization. Clients send codes in whatever shape their platform hands them: BCP47 (`de-DE`, `zh-TW`), script codes (`zh-Hans`, `zh-Hant`), legacy provider codes (`zh-tw`), or bare simple codes (`de`). All normalize to the internal simple key first. The subtlety is that two alias tables run in opposite directions, and folding one into the other is a bug invisible until it ships.

```ruby
# BCP47_TO_SIMPLE is a bijection — one canonical tag per simple code — so its
# inverse is lossless and safe to send back out to upstream providers.
BCP47_TO_SIMPLE = {
  "en-US" => "en", "de-DE" => "de", "pt-PT" => "pt",
  "zh-CN" => "zh", "zh-TW" => "zh_TW"
  # ... one canonical BCP47 tag per simple code
}.freeze

SIMPLE_TO_BCP47 = BCP47_TO_SIMPLE.invert.freeze

# Inbound-only, many-to-one conveniences live HERE, on top of the bijection,
# and never inside it. Hash#invert is last-key-wins, so folding a second tag
# for an existing simple code into BCP47_TO_SIMPLE would silently rewrite
# SIMPLE_TO_BCP47 and flip an outbound provider code you weren't looking at.
LANGUAGE_ALIASES = BCP47_TO_SIMPLE.merge(
  "zh-Hans" => "zh",
  "zh-Hant" => "zh_TW",
  "no"      => "nb",
  "zh-tw"   => "zh_TW",
  "zh-cn"   => "zh"
).freeze

def normalize_language(lang)
  LANGUAGE_ALIASES.fetch(lang.to_s, lang.to_s)
end

normalize_language("zh-Hant")  # => "zh_TW"
normalize_language("de")       # => "de"
normalize_language("pt-BR")    # => "pt-BR"  (no alias; fails validation → 400)
SIMPLE_TO_BCP47["pt"]          # => "pt-PT"
```

`BCP47_TO_SIMPLE` is a bijection precisely so that inverting it is lossless: one canonical BCP47 tag per simple code. The inbound-only aliases (`zh-Hans`, `no`, the lowercase variants) are many-to-one, so they cannot go in the invertible map without the inversion picking an arbitrary winner. An innocent-looking `"pt-BR" => "pt"` in the canonical map would flip the outbound Portuguese sent to providers from `pt-PT` to `pt-BR`. That is why the conveniences sit in a separate `merge` on top, and why `pt-BR` is deliberately not an alias at all: it is rejected like any unknown code. An alias table has a direction, and if you ever invert it, the un-inverted table must be a bijection.

The client keeps its own 27-case language list, and there is no automated cross-repo parity test between the two. Add a language on the client and not the server, and nothing in either repo's CI fails at the seam. The only guards are the server's alias table, which normalizes the client's `zh-Hans` into `zh`, and the inclusion validation, which returns `400` for anything unrecognized. That is a deliberate, slightly uncomfortable place to stand: two lists, one bridge, no compiler holding them together. It works because the bridge is small, centralized, and reviewed as the seam it is. The one place the coupling is made explicit is the client's [rendered-width validator](/rendered-width-validation/), which reads the server's locale files to measure server-composed strings against real device fonts. The two repos share no code, but they share the strings, and the width tool checks them in the font they will appear in.

### One Punctuation Rule, Owned Once

Once the server owns the words, it should own the punctuation too, or every client reinvents it and they disagree. Terminal punctuation, whitespace collapsing, and first-letter casing are display formatting, and display formatting is universal: the same rule for every output shape and every consumer. It lives in exactly one concern, with a single list of languages that take no Latin terminator:

```ruby
require "active_support/all" # String#upcase_first

module Punctuation
  UNTERMINATED_LANGUAGES = %w(ja zh zh_TW th).freeze
  DANDA_LANGUAGES = %w(hi).freeze

  def punctuate_summary(text, language:)
    return nil if text.nil?

    text = text.strip
    return text.upcase_first if UNTERMINATED_LANGUAGES.include?(language)

    if DANDA_LANGUAGES.include?(language)
      text = text.gsub("!", "।").sub(/\.+\z/, "।").squeeze("।")
      text += "।" unless text.empty? || text.end_with?("।")
    else
      text = text.gsub("!", ".").squeeze(".")
      text += "." unless text.empty? || text.end_with?(".")
    end
    text.upcase_first
  end

  def punctuate_alert(text, language:)
    return nil if text.nil?

    text = text.strip
    text += "." unless UNTERMINATED_LANGUAGES.include?(language) || text.empty? || text.end_with?(".")
    text.upcase_first
  end
end
```

The two methods are deliberately unequal. `punctuate_summary` does the full pass, with CJK and Thai appending nothing and the danda languages terminating with `।`, and the client deleted its own summary-cleanup code in favor of it. `punctuate_alert` only applies the terminator, with no exclamation-point normalization and no run-collapsing, because an alert is authoritative government prose and reshaping its punctuation would alter an official message. They are kept structurally separate so nobody can later unify them and quietly start rewriting alert text. One exemption completes the picture: strings authored entirely in the locale files, where a translator wrote the sentence and its terminal punctuation, are already display-ready and skip the pass. The rule punctuates what the server assembles; it does not second-guess what a human already finished.

### The Test That Catches None of This by Default

Every rule above has the same failure signature: it works in English and breaks in some other language. The default fixtures are English-only. An English summary is already terminated, upper-cased, and Latin-scripted, so a change that mangles the CJK path, breaks alias normalization for `zh-Hant`, or drops a `_name` companion sails through an English snapshot untouched.

The discipline is a one-line convention: when a change is language-dependent, assert a non-English language.

```ruby
require "test_helper"

class SummaryPunctuationTest < ActiveSupport::TestCase
  test "Japanese summaries carry no Latin period; English ones do" do
    en = Api::Weather.new(source: "mock", output: "hello_weather", lang: "en").to_h.deep_symbolize_keys
    ja = Api::Weather.new(source: "mock", output: "hello_weather", lang: "ja").to_h.deep_symbolize_keys

    assert_equal "Drizzle.", en.dig(:currently, :summary)
    assert_equal "Drizzle",  ja.dig(:currently, :summary) # no Latin period
  end
end
```

Without the `ja` assertion, the no-terminator rule for CJK could regress and every test would still be green. A non-Latin language in the assertion is not thoroughness for its own sake. It is the only assertion that exercises the branch English never touches.

## Results

- Every derived string is pinned to the request's language through one helper. No thread-global locale, no per-call-site guesswork, so a concurrently composed response never leaks a second language.
- Adding a locale file is the whole act of adding a language. The valid set is derived from the framework's locale registry, and inclusion validation returns `400` for anything else. The invertible provider table and the inbound merge are physically separate, and the `pt-BR` outbound flip never shipped.
- The app deleted its summary-massaging code. Punctuation has one owner, a structurally narrower alert method keeps government prose verbatim, and clients render bytes.
- The cost accepted: the client keeps its own language list with no parity test, on the bet that a small reviewed bridge is safer than a fragile cross-repo one. That bet has held.

## Lessons Learned

- **A missing translation is a bug, not a fallback string.** An exhaustive `case` with a raising `else` turns the gap into a red test instead of "translation missing" on a customer's screen.
- **Know the direction of every alias table.** If it will ever be inverted, the un-inverted form must be a bijection, and many-to-one conveniences live in a layer that is never inverted.
- **Keep a deliberately narrower rule as its own method.** When one output must stay verbatim, a separate function stops someone from DRYing the difference away.
- **Assert a non-Latin language for any change that touches localized output.** English fixtures exercise no terminator suppression, no script difference, and no alias normalization.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The title was shortened, the alias and punctuation sections each state their point once after the code instead of re-walking it, the bolded in-paragraph rules were unbolded, Results folds in the accepted trade-off, and Lessons Learned went from six bullets to the four the body does not already state. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
