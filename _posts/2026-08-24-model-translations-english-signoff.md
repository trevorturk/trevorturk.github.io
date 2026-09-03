---
layout: post
title: "Every Feature Is a Localization Event"
date: 2026-08-24 09:00:00 -0600
summary: "Apple's String Catalog shows the raw key when a translation is missing, so a half-translated feature ships broken. That one fact means every PR translates its own copy into 26 languages, we sign off the English first, a pipeline edits a 6 MB catalog without rewriting the whole file, and we ran a paid-editor pilot, measured it, and cancelled it."
tags: [localization, ai-agents, workflow]
model: "Claude"
last_edited: 2026-09-03
last_edited_by: "Claude Fable 5.1"
---

## The Problem

Apple's String Catalog is the file that holds an iOS app's translations. Each entry is keyed by its English source string, with one row per language. If the Vietnamese row is missing, the app doesn't fall back to the English we shipped. It shows the key. For a plain label, that's English text in the middle of a Vietnamese screen. For a plural form, or a key where we overrode the English text, it's the raw key: the `%lld minutes` incident put "Rising in 1 minutes." on screen. We checked this on a device. Most localization stacks fall back to the source language, so an incomplete translation is a cosmetic gap. Here it's a broken screen in production. We learned that the expensive way with a half-filled Portuguese variant, which is a story of its own.

So every key is either translated everywhere or broken somewhere. Add a button in English and translate it next sprint, and the button is broken in 26 languages for a sprint. [Hello Weather](https://helloweather.com) has more text than most weather apps: stat cards, radar legends, chart labels, watch and widget labels, and hundreds of lines of settings. One person maintains all of it, with no translation agency and a language model doing the translating. So the rule is blunt:

> **Every feature is a localization event.** Any PR that adds or changes user-visible copy fills all 26 non-English languages in the same PR, because there is no key-level fallback and a half-translated feature renders raw keys.

## The Solution

A PR checklist enforces the rule. Everything else in the workflow exists to make translating in the same PR cheap enough to do every time and safe enough to trust:

- Sign off the English before writing any translations
- A pipeline that edits the catalog without rewriting the whole file
- QA in tiers and stages, calibrated on one language first
- A pilot with a paid editor, where we judged every changed row
- Model routing: judgment on the strongest model, mechanical work on a cheaper one

### The gate: same PR, fast lane, never auto-merged

A feature with English-only keys passes review in an English simulator, merges, and shows raw keys the next time a non-English user opens that screen. Nothing in the build catches it. So we wrote the rule down twice: once in the architecture rules that must never be broken, and once in the PR checklist that runs before every push. Here's the checklist, condensed. A "fill" is a translated value for one key in one language, and the order of the items matters:

```markdown
### Localization gate (every PR that touches user-visible copy)

- [ ] New/changed copy is extractable — `Text("…")` or the `localized(_:)`
      helper, never string concatenation — and the build's extraction step
      lists every new key.
- [ ] Short, polysemous keys ("Rate", "Start", "Fair") carry a translator
      comment or a distinct key naming the rendering surface.
- [ ] English source text is signed off *before* any fills are written.
- [ ] All 26 non-English languages are filled in *this* PR (single `pt`).
- [ ] Any `en` row that differs from the source text has `state: translated`.
- [ ] Placeholder parity holds for every filled value.
- [ ] Catalog-only PR? Validators and tests are green; do not auto-merge.
```

Extraction comes first because a string only gets into the catalog if the build's extraction step sees it. A string built by concatenation never becomes a key, so it can never be translated. The last item is the one exception. A PR that touches only catalog values and process files, no code, gets a fast lane: once the validators are green, we merge it quickly. It still waits for a person to merge it, and it never changes the English to make a translation fit.

### Sign off the English first

When the model drafts copy, we sign off the English first and only then write the 26 translations. The reason is mechanical. Each translation is keyed to the exact English string. Change the button label after the translations are in, and all 26 rows hang off a key that no longer exists. Extraction drops them and the raw key shows again. An English edit after the translations costs 26 times what it costs before them.

So the order is fixed: draft the English, sign it off, then fan out the translations. The same order keeps the [agent fan-out](/agent-fanout-isolation-contract/) safe, because agents never work from source text that's still changing.

### Editing a 6 MB catalog without corrupting it

The catalog is a 6 MB JSON file. Xcode owns it and rewrites it in its own style: its own formatting, its own key order (the `localizedStandardCompare` sort), and its own escaping rules. The obvious way to bulk-edit it is `json.load`, change some values, `json.dump`. That rewrites the whole file in the script's style instead. A forty-value change turns into a diff of thousands of lines, and an escaped placeholder can get mangled on the way through without anyone noticing.

So we edit a handful of strings one at a time with exact text replacements. A bulk change goes through a pipeline with one rule: don't touch the real file until you've proven the change is right. Apply the replacements to a copy in memory, validate the copy, write it, then read the file back and check that it's the same bytes. The pipeline is a procedure written in the localization skill, not a checked-in script. Condensed into one, it looks like this.

```python
"""Apply bulk String Catalog fills without letting an unowned serializer
touch the file: simulate in memory, validate, write only if the bytes match."""

import json
from dataclasses import dataclass

from placeholders import placeholder_multiset  # see the next block


@dataclass(frozen=True)
class Replacement:
    old: str  # exact block as the file serializes it today
    new: str  # same indentation/order/escaping, new value


def apply_once(text: str, repl: Replacement) -> str:
    count = text.count(repl.old)
    if count != 1:
        raise ValueError(f"expected exactly 1 occurrence, found {count}")
    return text.replace(repl.old, repl.new, 1)


def validate(before: dict, after: dict, touched: set[str]) -> None:
    """after may differ from before only at `touched` keys, must keep the key
    count, and must preserve each touched value's placeholder multiset. The
    key IS the English source, so parity compares a value against its key."""
    if len(before["strings"]) != len(after["strings"]):
        raise ValueError("key count changed")

    for key, entry in after["strings"].items():
        if key not in touched:
            if entry != before["strings"][key]:
                raise ValueError(f"untouched key changed: {key!r}")
            continue
        for lang, loc in entry["localizations"].items():
            value = loc["stringUnit"]["value"]
            if placeholder_multiset(value) != placeholder_multiset(key):
                raise ValueError(f"placeholder parity broke: {key!r} / {lang}")


def apply_fills(path: str, replacements: list[Replacement], touched: set[str]) -> None:
    with open(path, encoding="utf-8") as f:
        original = f.read()

    simulated = original
    for repl in replacements:
        simulated = apply_once(simulated, repl)  # occurrence count pinned to 1

    validate(json.loads(original), json.loads(simulated), touched)  # also proves it parses

    with open(path, "w", encoding="utf-8") as f:
        f.write(simulated)

    with open(path, encoding="utf-8") as f:
        written = f.read()
    assert written == simulated, "written bytes differ from the validated simulation"
```

The `assert` at the end matters because it ties the validation to the file on disk. The file we validated and the file we shipped are the same bytes.

The placeholder check the pipeline calls has to allow one thing and forbid another. A German translation can and should reorder a sentence, so `%1$@ starting in %2$lldm` may become `In %2$lldm beginnt %1$@`. The check accepts that reorder and rejects only a dropped or duplicated placeholder. It compares which placeholders appear and how many times, not where they sit. Get that backwards and you either block correct translations or ship a crash from a bad format string.

```python
"""placeholders.py — order-independent placeholder parity for format strings."""

import re
from collections import Counter

_SPECIFIER = re.compile(
    r"%(?:\d+\$)?[#0\- +']*\d*(?:\.\d+)?(?:hh|h|ll|l|q|L|z|t|j)?[@diouxXeEfgGaAcsp%]"
)


def placeholder_multiset(value: str) -> Counter:
    """Multiset of format specifiers, with positional indices normalized away
    so that a reordered translation compares equal to its source.

    A reorder keeps the same multiset:
    >>> a = placeholder_multiset("%1$@ starting in %2$lldm")
    >>> b = placeholder_multiset("In %2$lldm beginnt %1$@")
    >>> a == b
    True

    A dropped placeholder does not:
    >>> a == placeholder_multiset("Beginnt gleich")
    False
    """
    specifiers = []
    for match in _SPECIFIER.finditer(value):
        token = re.sub(r"^%\d+\$", "%", match.group(0))  # %2$lld -> %lld
        if token != "%%":
            specifiers.append(token)
    return Counter(specifiers)
```

### QA in tiers and stages, calibrated on German

A mistranslated one-word chart legend is a visible bug. A long FAQ paragraph that reads a little stiffly is not. So we split the catalog into two tiers by the screen that owns each key. Tier 1 is the core a human can review: forecast, radar, settings, alerts, widgets, watch. Tier 2 is long-form text that only the model reviews: FAQ, changelog, in-app news. The tier lists come from the build's string-extraction metadata, not from a hand-kept list, so we can check them against the real screens. On top of the tiers, QA runs in stages, cheapest first. Scripted checks go first: placeholder parity, translations far longer or shorter than the English, translations identical to the English, CJK punctuation, and glossary consistency. Then bilingual review sheets grouped by screen. Then the model reviews each language for two things, fluency and register.

We calibrated the QA on German first, over 1,092 rows, before running it on the other languages. The baseline was 1 P0, roughly 10 P1, and about 25 P2, where P0 is the most serious. The one P0 was a key with two meanings. "Rate" is a noun (a rate of change) and a verb ("rate the app"), and the model had translated it in the app-review sense inside a precipitation chart legend. Only a reviewer who knows where the key renders can catch that. So short keys with more than one meaning now get a translator comment or a distinct key when they're written, and that habit has its own rule.

### The pilot: one language, a paid editor, every change judged

The plan had been to send the tier 1 core, all 26 languages, to a paid vendor for post-editing (a human editor correcting machine translations). We didn't want to assume that spend would improve quality, so we ran a pilot first: one language, one paid editor, and a hard rule for what came back. We would compare every changed row against the shipped value and accept or reject it with a written reason. Editors aren't automatically right, so an acceptance needs a reason as much as a rejection does.

The editor changed 172 of 897 rows. Some changes were real wins: two grammar errors the model had made (a wrong adjective ending and a wrong participle), the standard German word for a barometer trend, and typography details like the proper ellipsis character. The same 172 changes also added new errors: the wrong word for watch complications (a P0, twice), a wrong moon-phase name, two wrong pollen nouns, a brand name that must never be translated, and terms that drifted from the platform's official OS wording. The editor introduced more errors than they found.

That doesn't mean the vendor was bad. It means we only knew the edit made things worse because we judged every changed row. The normal way to take a vendor deliverable is to import it whole, on trust. Done that way, it would have lowered quality while feeling like an upgrade. The record that showed us this is a table with one row per change:

```markdown
| Key / surface            | Editor's change            | Verdict | Reason                                         |
|--------------------------|----------------------------|---------|------------------------------------------------|
| `STEADY` (pressure trend) | → „BESTÄNDIG"             | accept  | barometer convention                           |
| `…` (trailing ellipsis)  | `...` → `…`                | accept  | platform typography convention                 |
| complication label       | → „Probleme"               | reject  | wrong sense for a watch complication (P0)      |
| `Skies` (icon-set name)  | Skies → a German noun      | reject  | brand / icon-set name, never translated        |
```

Then we measured the model's own work on the same rows: roughly a 0.3% hard-error rate over 897 rows, which was already good enough to launch. We cancelled the remaining paid orders and dropped paid post-editing. The scope file we'd written to drive the vendor work now drives the model's own QA passes instead.

### Model routing: judgment on the strongest model, mechanics on a smaller one

Register calls, review verdicts, and judging any outside edit stay on the strongest model. Translations, scripted validation, and cross-reference audits fan out to a pinned mid-tier model. We chose the mid-tier on purpose, because the largest models turn a bounded translation into a rewrite (the argument of the [middle-model post](/delegate-to-the-middle-model/)). The payoff shows at scale. The quality pass after the pilot ran ten audit agents plus one top-model judge across the 25 remaining languages in a day, and it restored seven passages that had been silently dropped upstream.

## Results

- 27 languages shipped with same-day store approval, and a test fails the PR on any key that is translated in some languages but not all 26.
- We dropped paid post-editing on evidence: the pilot put the model plus checks at roughly a 0.3% hard-error rate, and the vendor scope file now drives the model's own QA.
- A forty-value catalog change is now a forty-value diff, and the file we reviewed is the same bytes as the file we shipped.
- The cost: every PR that touches copy carries 26 translations, catalog-only PRs still wait for a person to merge, and tier 2 long-form text gets model review only.

## Lessons Learned

- **Know your fallback before you design the workflow.** If the stack falls back to the source language, you can translate later. If it shows raw keys, you have to translate in the same PR.
- **Don't load and re-save a file that another tool owns.** Make exact text replacements on a copy, validate the copy, and check that the bytes on disk match it.
- **Check the set of placeholders, not the order.** A good translation reorders them. Reject only a dropped or duplicated one.
- **Judge every row of an outside edit before trusting it.** Imported whole, a deliverable can lower quality while feeling like an upgrade.
- **The checks make model translation safe, not the model.** With the checks in place, the model's translations are trustworthy. Without them, no translator is, human or model.
