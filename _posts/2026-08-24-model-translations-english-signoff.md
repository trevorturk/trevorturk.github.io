---
layout: post
title: "Every Feature Is a Localization Event"
date: 2026-08-24 09:00:00 -0600
summary: "Apple's String Catalog has no key-level fallback, so a half-translated feature ships broken. That one fact forces same-PR translation into 26 languages, English sign-off before the fills, a pipeline that edits a 6 MB catalog without corrupting it, and a paid-vendor pilot we measured and cancelled."
tags: [localization, ai-agents, workflow]
---

## The Problem

A key like `feels_like_short` with no Vietnamese value does not render in English. It renders as `feels_like_short`. We verified this on a device: Apple's String Catalog has no key-level fallback, so a missing language row shows the raw internal key as literal broken text. Most localization stacks fall back to the source language, which makes an incomplete translation a cosmetic gap. Here it is a broken screen in production. We learned this the expensive way with a half-filled Portuguese variant, a story of its own.

That one fact makes translation all-or-nothing, per key, at ship time. Add a button in English and translate it next sprint, and for a sprint the button is broken in 26 languages. [Hello Weather](https://helloweather.com) has more surface for this than most apps: descriptive stat cards, radar legends, labeled chart rails, watch and widget labels, hundreds of lines of settings. All of it is maintained by one person with no translation agency, with a language model doing the translating. So the rule is blunt:

> **Every feature is a localization event.** Any PR that adds or changes user-visible copy fills all 26 non-English languages in the same PR, because there is no key-level fallback and a half-translated feature renders raw keys.

## The Solution

The rule is enforced by a PR checklist. Everything else in the workflow exists to make the same-PR fill cheap enough to be routine and safe enough to trust:

- English sign-off before the fills are written
- A catalog pipeline that never round-trips the file through a serializer it does not own
- Tiered, staged QA calibrated on one language first
- A paid-vendor pilot with every changed row adjudicated
- Model routing that keeps judgment on the strongest model and fans the mechanics out

### The gate: same-PR fill, fast-tracked but never unattended

A feature with English-only keys passes review in an English simulator, merges, and renders raw keys the next time a non-English user opens that screen. Nothing in the build catches it. So the rule is written twice: an architecture invariant lists it as never-violate, and a pull-request checklist runs before every push. The order of the checklist items matters:

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

Extraction comes first because a key only enters the catalog if the build's string extraction sees it. A string built by concatenation never becomes a key, so it can never be translated. The last item is the one carve-out. A PR touching only catalog values and process files, no code, gets a fast lane: validators green, merged quickly. It still never auto-merges, and it never changes English source text to make a translation fit.

### English sign-off before the fills

When the model drafts copy, the English is signed off first, and only then are the 26 fills written. The reason is mechanical. Each translated value is keyed to the exact English source string. Rewrite the button label after the fills are in, and all 26 rows hang off a key that no longer exists. Extraction drops them and the raw key renders again. An English edit after the fills costs 26 times what it costs before them.

So the sequence is fixed: draft English, sign off English, fan out the fills. The same ordering is what makes the [agent fan-out](/agent-fanout-isolation-contract/) safe. Agents never fan out over source text still in flux.

### Editing a 6 MB catalog without corrupting it

The catalog is a 6 MB JSON file that Xcode owns and rewrites, with its own serialization style, key order (Xcode's `localizedStandardCompare` sort), and escaping rules. The obvious bulk edit, `json.load`, mutate, `json.dump`, re-serializes the whole file in the script's style. A forty-value change becomes a diff touching thousands of lines, and an escaped placeholder can be silently mangled on the way through.

So a handful of strings are edited one at a time with exact anchored replacements, and a bulk fill goes through a validated-replacement pipeline with one invariant: never mutate the real file speculatively. Simulate in memory, prove it correct, then write, and require the bytes on disk to equal the validated copy.

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

The `assert` at the end is what gives the validation its meaning. The file you validated and the file on disk are the same bytes, so the check cannot drift from what shipped.

Placeholder parity, the check the pipeline calls, has to allow one thing and forbid another. A German translation can and should reorder a sentence, so `%1$@ starting in %2$lldm` may legitimately become `In %2$lldm beginnt %1$@`. The validator accepts that reorder and rejects only a dropped or duplicated placeholder. Parity is about the multiset of specifiers, not their order. Get it backwards and you either block correct translations or ship a format-string crash.

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

A one-word chart legend that mistranslates is a visible bug. A long FAQ paragraph that reads a little stiffly is not. So the catalog is tiered by owning screen: tier 1 is the human-reviewable core (forecast, radar, settings, alerts, widgets, watch), tier 2 the model-only long-form (FAQ, changelog, in-app news). The tier lists are derived from the build's string-extraction metadata rather than hand-maintained, so they stay verifiable against the actual screens. On top of the tiers runs a staged QA, cheapest first: deterministic validators (placeholder parity, length-ratio outliers, target-equals-English, CJK punctuation, glossary consistency), then bilingual review sheets bucketed by screen, then per-language model review on two lenses, fluency and register.

The QA was calibrated on German first, over 1,092 rows, before running across the fleet. The baseline: 1 P0, roughly 10 P1, about 25 P2. The single P0 was a polysemous key. "Rate" is both a noun (a rate of change) and a verb ("rate the app"), and the model had translated it in the app-review sense inside a precipitation chart legend. Only a reviewer who knows the rendering context can see that error, which is why short polysemous keys are now disambiguated at authoring time, a habit that now has its own rule.

### The pilot: one language, a paid editor, every row adjudicated

The plan had been to send the whole catalog to a paid post-editing vendor. Whether that spend bought quality was a question we refused to answer by assumption, so we ran a pilot: one language, one paid editor, and a hard rule on the way back. Every changed row would be adjudicated against the shipped value and accepted or rejected with a written reason. Editors are not automatically right, and an acceptance needs an argument as much as a rejection does.

The editor changed 172 of 897 rows. The genuine wins: two morphological errors the model had made (a wrong compound form, a wrong participle), the correct barometric-trend convention, and platform typography niceties like the proper trailing-ellipsis style. The same 172 changes also introduced new errors: a wrong term for watch complications (a P0, twice), a wrong moon-phase name, two wrong pollen nouns, a brand name that must never be translated, and drift from the platform's official OS terminology. Counted honestly, the editor introduced more errors than they found. The edit was net-negative.

The verdict is not that the vendor was bad. It is that we only knew the edit was net-negative because we adjudicated every changed row. Imported wholesale on trust, the normal way to consume a vendor deliverable, it would have lowered quality while feeling like an upgrade. The gate produced the signal, not the editor's credentials, and the record that made it visible is a table, one row per change:

```markdown
| Key / surface            | Editor's change            | Verdict | Reason                                         |
|--------------------------|----------------------------|---------|------------------------------------------------|
| `sunrise` (stat card)    | Aufgang → Sonnenaufgang    | accept  | matches the OS-term glossary                   |
| `…` (trailing ellipsis)  | `...` → `…`                | accept  | platform typography convention                 |
| complication label       | → „Probleme"               | reject  | wrong sense for a watch complication (P0)      |
| `Skies` (icon-set name)  | Skies → Himmel             | reject  | brand / icon-set name, never translated        |
```

Then we measured the model's own baseline against the same rows: roughly a 0.3% hard-error rate over 897 rows, already launch quality. The remaining paid orders were cancelled and paid post-editing retired. The scope file prepared to drive the vendor work now aims the model's own QA passes instead.

### Model routing: judgment up, mechanics out

Register calls, review verdicts, and the adjudication of any external edit stay on the strongest model. Fills, deterministic validation, and cross-reference audits fan out to a pinned mid-tier agent, deliberately not the largest one, because the largest models inflate a bounded fill into a rewrite (the argument of the [middle-model post](/delegate-to-the-middle-model/)). The payoff shows at scale. The quality pass after the pilot fanned ten audit agents plus one top-model adjudicator across the 25 remaining languages in a day and restored seven passages silently dropped upstream. That throughput exists only because the mechanical work is delegated and the judgment concentrated.

## Results

- 27 languages shipped with same-day store approval, and the catalog ships complete in every language or a test stops the PR.
- Paid post-editing was retired on evidence: the pilot put the model plus gates at roughly a 0.3% hard-error rate, and the vendor scope was repurposed to aim the model's own QA.
- A forty-value catalog fill is now a forty-value diff, with a byte-identity guarantee that the reviewed file is the shipped file.
- The cost: every PR that touches copy carries 26 fills, catalog-only PRs still wait for a human to merge, and tier 2 long-form text gets model review only.

## Lessons Learned

- **Know your fallback before you design the workflow.** A stack that falls back to the source language lets you translate lazily. One that renders raw keys makes same-PR fills the only correct workflow.
- **Never round-trip a file through a serializer you don't own.** Generate exact replacement blocks, validate a simulated copy, and require byte-identity before writing.
- **Check the set of placeholders, not the order.** A good translation reorders them. Accept the reorder and reject only a dropped or duplicated one.
- **Adjudicate every row of an external edit before trusting it.** A deliverable consumed wholesale can lower quality while feeling like an upgrade.
- **The gates make model translation safe, not the model.** Build the gates and the model is trustworthy. Skip them and no translator, human or model, is.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.

**Rewrite (2026-09-01):** Part of an archive-wide rewrite. The owner asked, "with Fable 5.1, supposedly the writing quality is much better, I'm wondering if we should do a pass on all of the blog posts we have so far to improve them. should we start with the latest one?" and, after a pilot on the worktrees post, "I like the rewrite in any case and we have a lot of Fable capacity at the moment, should we go for it and dispatch an initial round of research to improve our skills, agents.md, etc and then dispatch sub-agents to rewrite each post? this could be done in a single PR, I think." Four Claude Fable 5.1 agents surveyed the archive to settle the voice and structure rules now in the blog-post-generator skill, and one agent rewrote this post under them. The post now opens on the raw key rendering on a device instead of on how much text the app carries, the title is one clause, each mechanism section gives its reason once, Results is what changed and what it cost, and Lessons Learned dropped the bullet that repeated a section heading. Code blocks, dates, numbers, links, and headings are unchanged, and no facts were added.
