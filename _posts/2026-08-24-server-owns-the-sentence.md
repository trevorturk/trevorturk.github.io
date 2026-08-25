---
layout: post
title: "Server Owns the Sentence: Codes Not Prose, Universal Punctuation, and One Alias Table Between Two Language Lists"
date: 2026-08-24 09:30:00 -0600
summary: "Localize codes at the edge instead of composing English sentences in the middle, own display punctuation in one server-side rule, and know the direction of every language alias table so an inversion doesn't flip your outbound codes."
tags: [ruby, api-design, localization, i18n]
---

## The Problem

A weather API returns two kinds of text, and only one of them is yours.

The first kind is prose an upstream data source already wrote: a summary sentence for the hour, a daily narrative. When the source supports the requested language, that text arrives localized and you pass it through. Our companion post on [probing vendor language support](/probing-vendor-language-support/) is about knowing whether that prose is real or a silent English fallback.

The second kind is text *you* derive: an air-quality level name, a pressure-trend phrase, a condition-code description. This text does not exist until your server composes it, and the moment it composes in English you have created a localization debt you may never be able to pay. A field that ships as `"Health effects possible."` is a finished English sentence. There is no seam left to translate it at. The client receives prose, not data, and prose in the wrong language cannot be fixed downstream.

The trap is subtle because the English path always works. The app looks great, and the bill comes due six months later when a customer in another language reports half the forecast screen is in English. By then the sentence is load-bearing in a dozen call sites.

At [Hello Weather](https://helloweather.com) the derived fields are localized on the server, in the same request, pinned to the request's language, and the app renders them verbatim. This post is the server-composition side of localization: composing text that is translatable by construction, owning display punctuation in exactly one place, and bridging two independently-maintained language lists with an alias table whose *direction* you understand.

## The Solution

Three rules, each cheap, each with a failure mode that only shows up in a non-English request:

1. **Ship codes, not sentences.** Every descriptive field is a machine code plus, optionally, a localized `_name`/`_phrase` companion resolved through a helper pinned to the request's language. The server never composes an English sentence it cannot later translate.
2. **Bridge language lists in one direction each.** Derive the valid-language set from one source of truth, normalize inbound codes through an alias table, validate by inclusion, and reject the rest. Keep the outbound provider-code table separate from the inbound alias table, because one is the *inverse* of the other and inversion is lossy.
3. **Own display punctuation universally.** One rule, one concern, every output shape, every consumer. Strip, normalize, terminate (except where a language doesn't terminate), upcase the first letter. Clients render the result byte-for-byte.

The connective tissue under all three is a test discipline: **default snapshots are English-only, so a language-dependent change has to assert a non-English language or it ships broken silently.**

### Codes, Not Sentences

The whole architecture rests on one helper. In the class that derives attributes, `t()` resolves an I18n key against the language the request asked for — never `I18n.locale`, never a thread default. The language is a property of the parsed request, threaded through explicitly, so every derived string comes out in one language no matter how concurrently the server is composing. A pressure trend then ships as a stable, language-free code beside a localized label, and the client switches on the code for logic and renders the label for display:

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

The field pair is what to notice. A `de` request emits `{"pressure_trend": "falling", "pressure_trend_name": "Fallend"}`; an `en` request emits `{"pressure_trend": "falling", "pressure_trend_name": "Falling"}`. The code is identical across languages, so client logic never parses prose; only the companion is translated, at the one seam that holds the language.

The verbose `case` is the safety, not clutter to remove. Dynamic key lookup turns a missing translation into a *shipped string*: I18n returns `"translation missing: ..."`, which is not an error but content, and it renders on a customer's screen. The exhaustive `case` with a raising `else` turns the same gap into a failed test — a bug you hear about from CI, not from a customer reading "translation missing" in their forecast. Keep the case structure when you localize; do not "simplify" it into an interpolation.

### One Language List, Two Directions of Aliasing

The valid-language set is derived once, at boot, from the framework's own locale registry — `Rails.application.config.i18n.available_locales` — so there is no second hardcoded list to drift against. Add a locale file and the language becomes valid; the 27 codes (`en cs da de el es fi fr hi hu id it ja ko nb nl pl pt ro ru sv th tr uk vi zh zh_TW`) are exactly the set of files on disk. Validation is then one line — `validates :language, inclusion: { in: LANGUAGE }` — and an unknown code is a clean `400`.

Between the wire and that validation sits normalization. Clients send codes in whatever shape their platform hands them: BCP47 (`de-DE`, `zh-TW`), script codes (`zh-Hans`, `zh-Hant`), legacy provider codes (`zh-tw`), or bare simple codes (`de`). All normalize to the internal simple key first. The subtlety: there are two alias tables running in opposite directions, and folding one into the other is a bug invisible until it ships.

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

Read the comment twice. `BCP47_TO_SIMPLE` is a bijection precisely so inverting it is lossless: one canonical BCP47 tag per simple code. The inbound-only aliases (`zh-Hans`, `no`, the lowercase variants) are many-to-one, so they cannot go in the invertible map without the inversion picking an arbitrary winner. Adding an innocent-looking `"pt-BR" => "pt"` to the canonical map would flip the *outbound* Portuguese sent to providers from `pt-PT` to `pt-BR`. That is why the conveniences sit in a separate `merge` on top, and why `pt-BR` is deliberately not an alias at all: it is rejected like any unknown code. **The transferable rule: an alias table has a direction, and if you ever invert it, the un-inverted table must be a bijection.**

Now the honest part. The client keeps its *own* 27-case language list, and there is **no automated cross-repo parity test between the two.** Add a language on the client and not the server, and nothing in either repo's CI fails at the seam. The only guards are the server's alias table (which normalizes the client's `zh-Hans` into `zh`) and the inclusion validation (which returns `400` for anything unrecognized). That is a deliberate, slightly uncomfortable place to stand: two lists, one bridge, no compiler holding them together. It works because the bridge is small, centralized, and reviewed as the seam it is. The one place the coupling is made explicit is the client's [rendered-width validator](/rendered-width-validation/), which reads the server's locale files to measure server-composed strings against real device fonts — the two repos share no code, but they share the strings, and the width tool checks them in the font they'll appear in.

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

`punctuate_summary` does the full pass: strip, turn exclamation points into periods, collapse runs of terminators, append a period unless the string is blank or already terminated, and upcase the first character. CJK and Thai append nothing and let native terminators through; the danda languages terminate with `।`. The client deleted its own summary-cleanup code and renders whatever the server sends.

Notice that alerts get a *deliberately narrower* method. `punctuate_alert` only applies the terminator — no exclamation-point normalization, no run-collapsing. That is on purpose: an alert is authoritative government prose, and reshaping its punctuation would alter an official message. The two methods are kept structurally separate **so nobody can later "unify" them and quietly start rewriting alert text.** The separation is the feature. One exemption completes the picture: strings authored entirely in the locale files, where a translator wrote the sentence and its terminal punctuation, are already display-ready and skip the pass. The rule punctuates what the server assembles; it does not second-guess what a human already finished.

### The Test That Catches None of This by Default

Every rule above has the same failure signature: it works in English and breaks in some other language. And the default fixtures are English-only. An English summary is already terminated, upper-cased, and Latin-scripted, so a change that mangles the CJK path — or breaks alias normalization for `zh-Hant`, or drops a `_name` companion — sails through an English snapshot untouched.

The discipline is a one-line convention with teeth: **when a change is language-dependent, assert a non-English language.**

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

Without the `ja` assertion, the no-terminator rule for CJK could regress and every test would still be green. A non-Latin language in the assertion is not thoroughness for its own sake; it is the only assertion that exercises the branch English never touches.

## Results

- **One helper** pins every derived string to the request's language: no thread-global locale, no per-call-site guesswork, so a concurrently-composed response never leaks a second language.
- **The valid-language set is one line** derived from the framework's locale registry, with single-line inclusion validation returning `400` for anything else. Adding a locale file is the whole act of adding a language.
- **The `pt-BR` outbound-flip bug never shipped** because the invertible provider table and the inbound merge are physically separate, with the hazard written into a comment at the table.
- **The client kept its own language list** and the team accepted no parity test, betting a small reviewed bridge is safer than a fragile cross-repo one. That bet has held.
- **The app deleted its summary-massaging code.** Punctuation has one owner; a structurally narrower alert method keeps government prose verbatim, and clients render bytes.

## Lessons Learned

- **Localize codes at the edge, not sentences in the middle.** The moment your server composes an English sentence, you have made content that cannot be translated afterward. Ship a machine code plus a localized companion, resolved at the one seam that holds the language — because the English path always looks fine, so this has to be a rule, not a judgment call.

- **A missing translation is a bug, not a fallback string.** Dynamic key interpolation turns an unclassified value into "translation missing" on a customer's screen; an exhaustive `case` with a raising `else` turns the same gap into a red test. Verbose and loud beats terse and silent.

- **Know the direction of every alias table.** If a table will ever be inverted, the un-inverted form must be a bijection, and your many-to-one conveniences must live in a separate layer that never gets inverted — because `Hash#invert` is last-key-wins, and the day it picks the wrong winner it changes an outbound value you weren't even looking at.

- **Two lists bridged by one small table can be the right call.** There is no compiler enforcing that the client and server language lists agree, and that is survivable precisely because the bridge is centralized and reviewed with care. Don't add a fragile cross-repo test to fake a guarantee you don't have; make the bridge small enough to hold in your head.

- **Own display punctuation once, and resist unifying the exceptions.** One rule, one concern, every output — but when a second rule (alerts) is deliberately narrower, keep it as its own method so nobody DRYs the difference away, because the difference is the point.

- **Snapshot a non-Latin language or the drift ships silently.** English fixtures exercise no terminator suppression, no script difference, no alias normalization; a single `lang=ja` assertion is worth more than a hundred English rows for any change that touches localized output.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
