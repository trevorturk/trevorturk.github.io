---
layout: post
title: "Every Feature Is a Localization Event: Model Fills, English Sign-Off, and a Pilot That Retired Paid Post-Editing"
date: 2026-08-24 09:00:00 -0600
summary: "Apple's String Catalog has no key-level fallback, so a half-translated feature ships broken. That one fact forces same-PR translation into 26 languages, English sign-off before the fills, a pipeline that edits a 6 MB catalog without corrupting it, and a paid-vendor pilot we measured and cancelled."
tags: [localization, ai-agents, workflow]
---

## The Problem

[Hello Weather](https://helloweather.com) carries far more text than a typical weather app: descriptive stat cards, radar legends, labeled chart rails, watch and widget labels, hundreds of lines of settings. That text is part of the design, not decoration on top of it. And it is maintained by one person with no translation agency, with a language model doing the actual translating.

The constraint that shapes everything else is what Apple's String Catalog does when a language is missing a string. Most localization stacks fall back to the source language, so an incomplete translation is a cosmetic gap. The String Catalog does not. We verified it on a device: a missing language row renders the raw internal key, not English — a key like `feels_like_short` with no Vietnamese value renders as literal broken text, not readable English. There is no key-level fallback. (We learned this the expensive way with a half-filled Portuguese variant — the subject of a [companion post](/one-portuguese/).)

Translation is therefore all-or-nothing, per key, at ship time. Add a button in English and translate it next sprint, and next sprint the button is broken in 26 languages in production. So the rule is blunt:

> **Every feature is a localization event.** Any PR that adds or changes user-visible copy fills all 26 non-English languages in the same PR, because there is no key-level fallback and a half-translated feature renders raw keys.

## The Solution

### The gate: same-PR fill, fast-tracked but never unattended

Without a gate, the failure is concrete: a feature merges with English-only keys and 26 languages render raw keys the next time a non-English user opens that screen, invisible in a review whose simulator is in English. So the rule is enforced twice: an architecture invariant states it as never-violate, and a pull-request checklist step runs before every push.

The ordering of its items is the point:

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

Extraction is checked before fills for a reason: a key only enters the catalog if the build's string extraction sees it, so a string built by concatenation never becomes a key and can never be translated.

The last item is the one carve-out: a PR touching only catalog values and process files, no code, gets a fast lane — validators green, merged quickly. But it never auto-merges, and it never changes English source text to make a translation fit.

### English sign-off before the fills

The ordering rule that saves the most work is counterintuitive: when the model drafts copy, the English is signed off first, and only then are the 26 fills written.

The reason is mechanical, not aesthetic. Each translated value is keyed to the exact English source string, so rewriting the English button label after the fills are in attaches all 26 rows to a key that no longer exists — extraction drops them and the raw key renders again. Editing the English later is not free; it is 26 times more expensive than editing it now.

So the sequence is fixed: draft English, sign off English, fan out the fills. This is the discipline that makes the [agent fan-out](/agent-fanout-isolation-contract/) safe — you never fan out over source text still in flux.

### Editing a 6 MB catalog without corrupting it

The catalog is a 6 MB JSON file that Xcode owns and rewrites, with a specific serialization style, key order (Xcode's own `localizedStandardCompare` sort), and escaping rules. The obvious way to bulk-edit it — load it, mutate the dictionary, write it back — is exactly wrong: a `json.load` / `json.dump` round-trip re-serializes in the script's style, so a forty-value change produces a diff touching thousands of lines and can silently mangle an escaped placeholder.

So a handful of strings are edited one at a time with exact anchored replacements, and a bulk fill goes through a validated-replacement pipeline with one invariant: never mutate the real file speculatively. Simulate in memory, prove it correct, then write — requiring the bytes on disk to equal the validated copy.

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

The `assert` at the end is the load-bearing line: the file you validated and the file on disk are provably the same bytes, so validation cannot drift from what shipped.

Placeholder parity, the check the pipeline calls, has to allow one thing while forbidding another. A German translation can and should reorder a sentence, so `%1$@ starting in %2$lldm` may legitimately become `In %2$lldm beginnt %1$@`. The validator must accept that reorder and reject only a dropped or duplicated placeholder — parity is about the multiset of specifiers, not the order. Get it backwards and you block correct translations or ship a format-string crash.

```python
"""placeholders.py — order-independent placeholder parity for format strings."""

import re
from collections import Counter

_SPECIFIER = re.compile(
    r"%(?:\d+\$)?[#0\- +']*\d*(?:\.\d+)?(?:hh|h|ll|l|q|L|z|t|j)?[@dioux XeEfgGaAcsp%]"
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

### Tiering, staged QA, and a calibration run

Not every key deserves the same scrutiny: a one-word chart legend that mistranslates is a visible, embarrassing bug; a long FAQ paragraph that reads slightly stiffly is not. So the catalog is tiered mechanically by owning screen — tier 1 the human-reviewable core (forecast, radar, settings, alerts, widgets, watch), tier 2 the model-only long-form (FAQ, changelog, in-app news). Derived from the build's string-extraction metadata rather than hand-maintained, the tier lists stay verifiable against the actual screens. On top of the tiers runs a staged QA, cheapest-first: deterministic validators (placeholder parity, length-ratio outliers, target-equals-English, CJK punctuation, glossary consistency), then bilingual review sheets bucketed by screen, then per-language model review on two lenses, fluency and register.

It was calibrated on German first, over 1,092 rows, before running across the fleet, setting the baseline: 1 P0, roughly 10 P1, about 25 P2. The single P0 is the archetypal localization bug — a polysemous key. "Rate" is both a noun (a rate of change) and a verb ("rate the app"), and the model had translated it in the app-review sense inside a precipitation chart legend — a sense error only a reviewer who knows the rendering context can see. It is why the workflow now disambiguates short polysemous keys at authoring time, a habit with [its own post](/rate-start-fair-polysemy/).

### The pilot: one language, a paid editor, every row adjudicated

The plan had been to send the whole catalog to a paid post-editing vendor. Whether that spend actually bought quality was a question we refused to answer by assumption, so we ran a deliberate pilot: one language, sent to a paid editor, with a hard rule on the way back. Every changed row would be adjudicated against the shipped value, accepted or rejected with a written reason — editors are not automatically right, and every acceptance needs an argument as much as every rejection.

The editor changed 172 of 897 rows. The genuine wins: two morphological errors the model had made (a wrong compound form, a wrong participle), the correct barometric-trend convention, and platform typography niceties like the proper trailing-ellipsis style. But the same 172 changes introduced new errors too: a wrong term for watch complications (a P0, twice), a wrong moon-phase name, two wrong pollen nouns, a brand name that must never be translated, and drift from the platform's official OS terminology. Counted honestly, the editor introduced more errors than they found — the edit was net-negative.

The verdict that matters is not that the vendor was bad. It is that we only knew the edit was net-negative because we adjudicated every changed row. Consumed the normal way — imported wholesale on trust — it would have lowered quality while feeling like an upgrade. The gate, not the editor's credentials, produced the signal, and the record that made it visible is a table, one row per change:

```markdown
| Key / surface            | Editor's change            | Verdict | Reason                                         |
|--------------------------|----------------------------|---------|------------------------------------------------|
| `sunrise` (stat card)    | Aufgang → Sonnenaufgang    | accept  | matches the OS-term glossary                   |
| `…` (trailing ellipsis)  | `...` → `…`                | accept  | platform typography convention                 |
| complication label       | → „Probleme"               | reject  | wrong sense for a watch complication (P0)      |
| `Skies` (icon-set name)  | Skies → Himmel             | reject  | brand / icon-set name, never translated        |
```

Then we measured the model's own baseline against those same rows: roughly a 0.3% hard-error rate over 897 rows — already launch quality. So the remaining paid orders were cancelled and paid post-editing retired; the scope file prepared to drive the vendor work now aims the model's own QA passes instead.

### Model routing: judgment up, mechanics out

Model routing mirrors the general policy: register calls, review verdicts, and the adjudication of any external edit stay on the strongest model, while fills, deterministic validation, and cross-reference audits fan out to a pinned mid-tier agent — deliberately not the largest one, because the largest models inflate a bounded fill into a rewrite (the argument of the [middle-model post](/delegate-to-the-middle-model/)). The payoff shows at scale: the quality pass after the pilot fanned ten audit agents plus one top-model adjudicator across the 25 remaining languages in a day, restoring seven passages silently dropped upstream — throughput that exists only because the mechanical work is delegated and the judgment concentrated.

## Results

- **The catalog ships complete in every language, or a test stops the PR** — no half-translated state can leak, and the no-fallback behavior would expose it immediately if it did.
- **English changes stopped being expensive surprises.** Locking the source before the fills turned "redo 26 languages" into a one-line edit at the cheap moment.
- **Bulk catalog edits stopped producing thousand-line diffs** — the pipeline reduces a forty-value fill to a forty-value diff, with a byte-identity guarantee that the reviewed file is the shipped file.
- **A paid contract was cancelled on evidence, not vibes** — the pilot showed the model plus gates already at launch quality (~0.3% hard-error rate), and the vendor scope was repurposed to aim the model's own QA.
- **27 languages shipped with same-day store approval.**

## Lessons Learned

- **Know your fallback before you design the workflow.** A stack that falls back to the source language lets you translate lazily; one that renders raw keys does not, which makes same-PR fills the only correct workflow rather than a nicety.
- **Sign off the source before you fan out.** Translated rows are keyed to the exact English string, so rewriting the English after the fills discards all 26 languages.
- **Never round-trip a file through a serializer you don't own.** A `json.load` / `json.dump` cycle re-serializes in your style, churning the diff and risking escaping; generate exact replacement blocks, validate a simulated copy, and require byte-identity before writing. And check the set, not the order: a good translation reorders placeholders, so accept the reorder and reject only a dropped or duplicated one, or you block correct work and ship crashes.
- **Adjudicate every row of any external edit before trusting it.** The professional pass introduced more errors than it caught, visible only because we checked each changed row; a deliverable consumed wholesale can lower quality while feeling like an upgrade.
- **The gates make model translation safe, not the model.** It is launch-quality here because extraction rules, same-PR fills, validators, tiered review, polysemy disambiguation, and per-row adjudication surround it. Build the gates and the model is trustworthy; skip them and no translator, human or model, is.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
