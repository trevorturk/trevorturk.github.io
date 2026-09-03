---
layout: post
title: "Server Owns the Sentence"
date: 2026-08-24 09:30:00 -0600
summary: "Send codes with translated labels instead of English sentences, put display punctuation in one server rule, and keep the inbound language alias table separate from the outbound one so inverting it can't flip a provider code."
tags: [ruby, api-design, localization, i18n]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

A weather API returns two kinds of text, and only one of them is yours.

The first kind is prose the data vendor already wrote: a summary sentence for the hour, a narrative for the day. When the vendor supports the language we asked for, that text arrives translated and we pass it through. The companion post on [probing vendor language support](/probing-vendor-language-support/) covers how to tell whether that prose is really translated or a silent English fallback.

The second kind is text we write ourselves: the name of an air-quality level, a phrase for the pressure trend, a description for a condition code. It doesn't exist until our server writes it. If the server sends `"Health effects possible."`, that's a finished English sentence, and there's no seam left where anyone could translate it. The client gets prose instead of data, and prose in the wrong language can't be fixed downstream.

The trap is that the English path always works. The app looks fine. Six months later a customer in another language reports that half the forecast screen is in English, and by then the sentence is being built in a dozen call sites.

At [Hello Weather](https://helloweather.com) we translate the derived fields on the server, in the same request, in the language the request asked for, and the app shows them as they arrive. This post is the server side of that: writing text so it can be translated, owning punctuation in one place, and bridging two separately maintained language lists with an alias table whose direction we understand.

## The Solution

Three rules. Each is cheap, and each has a failure that only shows up in a non-English request:

1. **Ship codes, not sentences.** Every descriptive field is a machine code, plus an optional translated `_name` or `_phrase` label looked up in the request's language. The server never writes an English sentence it can't translate later.
2. **Bridge language lists in one direction each.** Build the set of valid languages from one source, normalize incoming codes through an alias table, accept what's in the set, and reject the rest. Keep the outbound provider-code table separate from the inbound alias table, because one is the inverse of the other and inverting a table loses information.
3. **Own display punctuation everywhere.** One rule in one module, for every output shape and every consumer. Strip, normalize, add a terminator (except in languages that don't use one), and capitalize the first letter. Clients show the result as it arrives.

Under all three sits one testing habit: the default fixtures are English-only, so a language-dependent change has to assert a non-English language, or it ships broken and nobody notices.

### Codes, Not Sentences

Everything rests on one helper. In the class that derives attributes, `t()` looks up an I18n key in the language the request asked for, never `I18n.locale` and never a thread default. The language is a property of the parsed request and gets passed down explicitly, so every derived string comes out in one language no matter how many requests the server is handling at once. A pressure trend then ships as a stable code that's the same in every language, next to a translated label. The client switches on the code for logic and shows the label:

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

  # The real helper also memoizes per locale; the lookup is the point.
  def t(key, **options)
    I18n.t(key, locale: @language.to_sym, **options)
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

Notice the pair of fields. A `de` request gets `{"pressure_trend": "falling", "pressure_trend_name": "Fällt"}`, and an `en` request gets `{"pressure_trend": "falling", "pressure_trend_name": "Falling"}`. The code is the same in both, so client logic never parses prose. Only the label is translated, at the one place that knows the language.

The long `case` is the safety, not clutter. With a dynamic key lookup, a missing translation becomes a shipped string: I18n returns `"translation missing: ..."`, which is content rather than an error, and it shows up on a customer's screen. The exhaustive `case` with a raising `else` turns the same gap into a failed test, so we hear about it from CI instead of from a customer. Keep the `case` when you localize; don't collapse it into an interpolated key.

### One Language List, Two Directions of Aliasing

We build the set of valid languages once, at boot, from the framework's own locale registry, `Rails.application.config.i18n.available_locales`, so the request model doesn't keep a second hardcoded list that could drift. That registry is set in one place, `config/application.rb`, and its 27 codes (`en cs da de el es fi fr hi hu id it ja ko nb nl pl pt ro ru sv th tr uk vi zh zh_TW`) match the locale files on disk one for one. Adding a language is a locale file plus one entry in that list. Validation is one line, `validates :language, inclusion: { in: LANGUAGE }`, and an unknown code gets a `400`.

Between the wire and that validation sits normalization. Clients send whatever code their platform gives them: BCP47 tags (`de-DE`, `zh-TW`), script codes (`zh-Hans`, `zh-Hant`), old provider codes (`zh-tw`), or bare codes (`de`). All of them get normalized to our internal simple key first. The catch is that we have two alias tables running in opposite directions, and folding one into the other is a bug you won't see until it ships.

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

`BCP47_TO_SIMPLE` is one-to-one on purpose (the comment calls it a bijection): one canonical BCP47 tag per simple code, so inverting it loses nothing. The inbound-only aliases (`zh-Hans`, `no`, the lowercase variants) are many-to-one. If they went in the invertible map, the inversion would pick an arbitrary winner. A harmless-looking `"pt-BR" => "pt"` in the canonical map would flip the Portuguese we send to providers from `pt-PT` to `pt-BR`. So the conveniences sit in a separate `merge` on top, and `pt-BR` isn't an alias at all; it's rejected like any unknown code. An alias table has a direction. If you're ever going to invert one, the table you invert has to be one-to-one.

The client keeps its own 27-case language list, and there's no automated test that the two lists match. Add a language on the client and not the server, and nothing in either repo's CI fails. The only guards are the server's alias table, which turns the client's `zh-Hans` into `zh`, and the inclusion check, which returns `400` for anything else. That's a slightly uncomfortable place to stand: two lists, one bridge, and no compiler holding them together. It works because the bridge is small, in one place, and reviewed carefully. The one place the coupling is explicit is the client's [rendered-width validator](/rendered-width-validation/), which reads the server's locale files and measures server-composed strings in real device fonts. The two repos share no code, but they share the strings, and the width tool checks them in the font they'll be shown in.

### One Punctuation Rule, Owned Once

Once the server owns the words, it should own the punctuation too. Otherwise every client reinvents it and they disagree. Terminal punctuation, whitespace collapsing, and capitalizing the first letter are display formatting, and display formatting is the same rule for every output shape and every consumer. It lives in one module, with one list of languages that don't take a Latin period:

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

The two methods are unequal on purpose. `punctuate_summary` does the full pass: Japanese, Chinese, and Thai get no terminator, Hindi gets a danda (`।`, the Devanagari full stop), and the client deleted its own cleanup code for server summaries because of it. `punctuate_alert` only adds the terminator, with no exclamation-point rewriting and no run-collapsing, because an alert is official government text and reshaping its punctuation would change an official message. They stay as separate methods so nobody can merge them later and quietly start rewriting alerts. One exemption: the level names and phrases that come straight from the locale files (`"Breathe easy"`, `"Feels muggy"`) are labels a translator finished, not sentences the server built, so they get `upcase_first` at most and no terminator. The one locale-file string that does take the full pass is the icon summary, because it ships in a `summary` slot that clients show as a sentence.

### The Test That Catches None of This by Default

Every rule above fails the same way: it works in English and breaks in some other language. The default fixtures are English-only. An English summary already ends in a period, starts with a capital, and uses Latin script, so a change that breaks the no-period path for Japanese, breaks alias normalization for `zh-Hant`, or drops a `_name` label passes an English snapshot untouched.

The habit is one line: when a change depends on language, assert a non-English language.

```ruby
require "test_helper"

class Api::SummaryPunctuationTest < ActiveSupport::TestCase
  # The real test loops over every output shape; one is shown.
  test "lang=ja applies the no-append rule end-to-end in every output" do
    en = Api::Weather.new(source: "mock", output: "hello_weather", lang: "en").to_h.deep_symbolize_keys
    ja = Api::Weather.new(source: "mock", output: "hello_weather", lang: "ja").to_h.deep_symbolize_keys

    assert_equal "Drizzle.", en.dig(:currently, :summary)
    assert_equal "Drizzle",  ja.dig(:currently, :summary) # no Latin period
  end
end
```

Without the `ja` assertion, the no-period rule for Japanese and Chinese could break and every test would stay green. A non-Latin language in the assertion isn't thoroughness for its own sake. It's the only assertion that runs the branch English never reaches.

## Results

- Every derived string is in the request's language, through one helper. No thread-global locale and no per-call-site guesswork, so a response composed while other requests are running never leaks a second language.
- Adding a language is a locale file and one registry entry. The valid set comes from the framework's locale registry, and inclusion validation returns `400` for anything else. The invertible provider table and the inbound merge are separate, and the `pt-BR` outbound flip never shipped.
- The app deleted its summary-cleanup code for server text. Punctuation has one owner, a narrower alert method keeps government text as written, and clients show what they get.
- The cost: the client keeps its own language list with no parity test, on the bet that a small reviewed bridge is safer than a fragile cross-repo one. As of August 2026, that bet has held.

## Lessons Learned

- **A missing translation is a bug, not a fallback string.** An exhaustive `case` with a raising `else` turns the gap into a red test instead of "translation missing" on a customer's screen.
- **Know the direction of every alias table.** If you'll ever invert it, the table you invert must be one-to-one, and many-to-one conveniences go in a layer that's never inverted.
- **Keep a deliberately narrower rule as its own method.** When one output must stay as written, a separate function stops someone from merging the difference away.
- **Assert a non-Latin language for any change that touches translated output.** English fixtures never exercise terminator suppression, script differences, or alias normalization.
