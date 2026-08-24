---
layout: post
title: "QA Pages Generated From Repo Data: A Type Registry, a Provenance SHA, and Self-Proving Coverage"
date: 2026-08-24 08:30:00 -0600
summary: "How to turn a throwaway localization review page into a durable artifact type defined by two committed files, stamped with the commit its data came from, and incapable of silently hiding a surface it forgot to render."
tags: [workflow, ai-agents, localization, tooling]
---

## The Problem

[Hello Weather](https://helloweather.com) carries far more text than a typical weather app, and the text is part of the layout rather than decoration on top of it. Before a release, someone has to look at how 24 date formats and a dozen stat cards render across 27 languages — on a chart rail, a stat card, a watch complication. That is thousands of tiny strings nobody flips through by hand, so the natural move is to generate a web page and send the link to whoever signs off.

The first rounds worked that way, and the link turned out to be a bad source of truth. It has no committed definition and nobody can diff it. By the next round nobody could tell which commit's data a page had rendered, whether the thresholds it flagged still matched the code, or whether it had quietly left a language out.

The worse failure was omission. A hand-built page renders from a list maintained by hand. Add a new date surface, forget to add it to the list, and the page renders cleanly with that surface missing — nothing fails, because nothing on the page knows a surface exists that it isn't drawing. The reviewer signs off on a spread that looks complete and isn't.

So we stopped treating the page as the artifact and treated a pair of committed files — versioned, diffable, reproducible, honest about their own coverage — as the artifact instead.

## The Solution

A QA page becomes a named **type**: two files in the repo.

1. **A spec** (`types/<name>.md`): YAML frontmatter with the commands that regenerate the data and the template path, then prose stating every threshold and an append-only Approvals log.
2. **A template** (`templates/<name>.html`): presentation only. It draws meters, tints cells, and lays out a table, holding no policy and no data.

The rendered page is generated on demand and never committed. The repo record is spec plus template plus the SHA the last approval was granted against, and that trio regenerates the exact approved page on demand.

The workflow is four steps: run the `data` commands read-only to a scratchpad; inject the JSON and a provenance object into the template's two placeholders and publish; iterate with the reviewer, folding live tweaks back into the committed template; and on approval, record the date, the data SHA, and the decision, then open a PR. The commit is the deliverable; the link is a convenience.

Three ideas make the artifact durable: the two-file split, the provenance stamp, and self-proving coverage.

### Why Split Policy From Presentation Into Two Files

A review page mixes two concerns that change for different reasons. The threshold that decides whether a surface is "too roomy and needs a per-language rule" is policy; the gradient that tints a cell is presentation. When they share a file, a visual tweak can move a policy number by accident, buried in layout code.

So the type is two files. Here is a spec — frontmatter that regenerates the page, then prose stating the thresholds and logging approvals:

```yaml
---
name: stat-card-qa
title: "Stat Card QA — 12 Cards × 27 Languages"
favicon: "📊"
data:
  - "CARDS_JSON=<scratchpad>/cards.json swift generate/stat-card-widths.swift"
template: templates/stat-card-qa.html
---

# Stat Card QA

Every stat card slot, rendered with the single widest string it can ever show,
measured in the system UI font against the design grid's point budget.

## Thresholds

- **OVER** — wider than the slot's budget at the default text size. The only
  gate; a red cell also carries the word OVER.
- **MARGIN** — within 95% of budget. Informational; tighten if a longer string
  could appear in that slot.
- **NARROW** — would clip on the narrowest legacy phone (375pt). Informational.
- **XXL** — over budget only at the largest Dynamic Type size. Informational.

Slot point budgets: title 106, subtitle 106, description 142. A slot passes at
or below 95% of its budget.

## Approvals

- **2026-08-18 — round closed.** All 12 cards × 27 languages × every slot at
  data SHA `a1b2c3d`. Standing state: 0 OVER rows. A drift tripwire was added so
  a new card cannot escape the sweep.
```

`name` ties the spec to its template, `data` is the exact regeneration command, and `title` and `favicon` stay stable across republishes so the tab keeps its identity and the page is findable by title, not a saved link. Everything below the frontmatter is policy: a reviewer can question the 95% pass bar in one Markdown file, and a designer can restyle the meters with no way to move a threshold, because the thresholds are not in the file they edit.

### Why a Rendered Page Needs a Provenance Stamp

Six weeks after a page is approved, the only durable question is which commit's data the reviewer signed off on. Without an answer, the approval is folklore, against a page since regenerated from newer data. So provenance is a property of a *rendering*, injected per render, never baked in.

The template ships with two placeholders and no data. The pipeline fills one with the generated JSON and the other with a provenance object holding at least the short main SHA. The JavaScript that receives them derives the page's enumeration from the data, renders anything the catalog forgot, stamps the provenance line, and banners on any gap:

```javascript
// Injected at generation time. The template ships with the two placeholders
// only — never with data, never with a baked-in SHA.
const DATA = /*__DATA__*/;   // { languages: { en: { intent: { cell, width } }, ... } }
const PROV = /*__PROV__*/;   // { sha: "a1b2c3d" }

// The hand-maintained catalog: how to group and label intents. It controls how
// things are shown, never whether they are shown.
const GROUPS = [
  { name: "Hourly bar graph", intents: ["hourlyChartHour", "hourlyChartWeekday"] },
  { name: "Headers & rows",   intents: ["dailyHeader", "dailyDay"] },
  // one entry per known surface
];

// The data is the authority. Derive the real enumeration from what the
// generator produced, not from the catalog above.
const langs = Object.keys(DATA.languages).sort();
const dataIntents = [...new Set(
  Object.values(DATA.languages).flatMap(entry => Object.keys(entry))
)];
const catalogIntents = GROUPS.flatMap(group => group.intents);

// Three kinds of coverage gap, all derived, none trusted to a human list.
const unclassified    = dataIntents.filter(i => !catalogIntents.includes(i));
const missingFromData = catalogIntents.filter(i => !dataIntents.includes(i));
const holes = langs.flatMap(lang =>
  dataIntents.filter(i => !DATA.languages[lang][i]).map(i => `${lang}.${i}`));

// The page states, on its own face, which commit it drew and how it measured.
document.getElementById("prov").textContent =
  `Source: HelloWeatherTests/snapshots/ @ main ${PROV.sha}` +
  ` · ${dataIntents.length} intents × ${langs.length} languages` +
  ` · strings measured in your system UI font.`;

// Anything the catalog forgot still renders — in a section of its own, never
// dropped, so a forgotten surface is impossible to miss.
if (unclassified.length) {
  document.getElementById("unclassified").innerHTML =
    `<h2>Unclassified intents</h2>` +
    `<p>Present in the data but missing from the catalog — classify in GROUPS.</p>` +
    unclassified.map(i => `<div class="intent">${i}</div>`).join("");
}

// A coverage mismatch is a visible failure state, not a silent omission.
if (unclassified.length || missingFromData.length || holes.length) {
  document.getElementById("banner").innerHTML =
    `<div class="banner">Coverage mismatch —` +
    ` unclassified: ${unclassified.join(", ") || "none"};` +
    ` in catalog but absent from data: ${missingFromData.join(", ") || "none"};` +
    ` language holes: ${holes.slice(0, 10).join(", ") || "none"}.</div>`;
}
```

When the approval is recorded, the SHA in the Approvals log and the SHA on the page are the same, so the approved artifact reproduces byte for byte.

That last clause is not cosmetic: a width verdict is only true in the face that draws the glyphs, so the generator measures in the app's own font, never a substituted webfont. This Swift script reads the blessed snapshot files and measures every string with the platform's text engine:

```swift
import AppKit
import Foundation

let snapshotsDir = "HelloWeatherTests/snapshots/date_format_snapshot_tests"
let outPath = ProcessInfo.processInfo.environment["DATES_JSON"] ?? "dates.json"

func width(_ string: String) -> Double {
    let font = NSFont.systemFont(ofSize: 17, weight: .regular)
    let attributed = NSAttributedString(string: string, attributes: [.font: font])
    return Double(ceil(attributed.size().width))
}

var languages: [String: [String: [String: Any]]] = [:]

let files = try FileManager.default
    .contentsOfDirectory(atPath: snapshotsDir)
    .filter { $0.hasSuffix(".snap.txt") }
    .sorted()

for file in files {
    // filename shape: date_format__<lang>.snap.txt
    guard let mark = file.range(of: "__"),
          let end = file.range(of: ".snap.txt") else { continue }
    let language = String(file[mark.upperBound..<end.lowerBound])

    let content = try String(contentsOfFile: snapshotsDir + "/" + file, encoding: .utf8)
    var intents: [String: [String: Any]] = [:]
    for line in content.split(separator: "\n").dropFirst() {
        let parts = line.components(separatedBy: " | ")
        guard parts.count == 2 else { continue }
        let cell = String(parts[1])
        intents[parts[0]] = ["cell": cell, "width": width(cell)]
    }
    languages[language] = intents
}

let json = try JSONSerialization.data(
    withJSONObject: ["languages": languages], options: [.sortedKeys, .prettyPrinted])
try json.write(to: URL(fileURLWithPath: outPath))
print("wrote \(outPath): \(languages.count) languages")
```

The generator renders nothing; it produces measurements in the font the template claims. Injection reads the short HEAD SHA into a provenance object and substitutes both placeholders through Python rather than `sed`, because the JSON is full of `/` and `&` that `sed` would mangle:

```bash
# Generate the data, build a provenance object, inject both into the template.
DATES_JSON=out/dates.json swift generate/date-matrix.swift
SHA=$(git rev-parse --short HEAD)

DATA=$(cat out/dates.json) PROV="{\"sha\":\"$SHA\"}" python3 - <<'PY' > out/date-format-qa.html
import os, pathlib
html = pathlib.Path("templates/date-format-qa.html").read_text()
html = html.replace("/*__DATA__*/", os.environ["DATA"])
html = html.replace("/*__PROV__*/", os.environ["PROV"])
pathlib.Path("/dev/stdout").write_text(html)
PY
```

The SHA is read at injection time, so the page can never claim a commit it wasn't generated from.

### Why the Page Proves Its Own Coverage

The omission failure from the top produces the worst outcome a review can have: a confidently-wrong sign-off. The defense is to never let the page trust its own hand-maintained list of what to render.

The `GROUPS` catalog above only groups and labels intents. The set of intents to *cover* is derived from the generated data, which in the date type traces back to the test suite's `allCases` over every date-format intent. So silent omission is impossible: add a new intent, and the next regeneration shows it in the unclassified section with the banner announcing the catalog is stale. The reviewer cannot approve a quietly incomplete page, because an incomplete page is visibly incomplete. The trade-off is that a stale catalog now costs a visible banner, not an invisible hole.

### Three Types, One Contract

The same three ideas host three measurement problems, and the differences show where a single metric lies.

**date-format-qa** uses two measures because one misses a case. A compact rail has a hard character cap (weekday 3, chart hour 5) enforced by a contract test, and going over is the only broken state; the ratio to English is informational, flagging roomy surfaces at 1.6× and 2.0×. But a capped rail also flags at a 1.05× ink ratio even when the string fits its character count, because a CJK or Thai glyph can sit inside a five-character budget while out-inking the English it replaced — a gap a count cannot see. The rulebook behind those caps has [its own post](/date-format-rulebook/).

**stat-card-qa** measures the widest string each slot can ever show against a fixed point budget, with OVER the only gate. Because it draws server strings from a second checkout, its provenance line stamps that checkout's head too and must say when it was absent, or the page under-reports. The slot budgets are [a post of their own](/designing-for-the-narrowest-slot/).

**wind-label-qa** renders 27 languages across 8 bearings at true weight inside the real bar geometry, with verdicts FITS, SCALE (fits only by shrinking), and TRUNC (clips even at the scale floor). Every abbreviation is click-to-edit with live re-measure, so a reviewer can try shorter forms in the browser and see which fit before touching the catalog.

## Results

- **Three named types in production**, each a spec plus a template regenerated from repo data on demand — no rendered HTML committed, no stale pages accumulating, and the repo record reproduces the approved page exactly.
- **The date-format round closed against a named data SHA** at zero contract failures, with every judgment call recorded in the type file so the next session sees what was decided.
- **The stat-card round covered all twelve cards across twenty-seven languages and every slot** with no truncating rows, plus a drift tripwire that stops a new card escaping the sweep.
- **The wind-label round found exactly one truncating language**, fixed by compacting four abbreviations, with the accepted scaling tail on the smallest phones attached to the artifact, not lost.
- **Forgotten surfaces stopped being invisible** — an un-rendered surface is now a banner and an unclassified section, not a hole nobody notices.

## Lessons Learned

- **Make committed files the artifact's identity, not the URL** — a link has no diff or history, so anything you need to reproduce or review later must live in files you can diff.
- **Separate policy from presentation into different files** — when a threshold and a gradient share a file, a visual tweak can move a policy number unnoticed.
- **Stamp every rendering with the SHA of its inputs** — an approval against a commit is reproducible and one against a link is not, so inject provenance per render.
- **Derive coverage from the data, not a hand-kept list** — a page that builds its enumeration from its inputs and banners on any gap cannot hide a forgotten surface, the failure a review most needs to prevent.
- **Measure in the font that ships** — a width verdict is only true in the face that draws the glyphs, so a substituted webfont makes the measurements fiction.
- **Pair every status color with a word** — a verdict in color alone disappears in grayscale and for a colorblind reviewer, so the color is decoration and the word is the verdict.
- **Add a second measure when one number has a blind spot** — a character cap cannot see ink width and a wide glyph exploits that gap, so measure both.

None of this is specific to weather. Any team producing visual review artifacts can define the artifact as committed files, generate on demand, stamp with provenance, and let the data prove its own coverage.

---

## How This Post Was Made

**Prompt:** "dispatch research into web/ios about how to ios/web handles localization, including developer tooling, snapshot testing, space-constrained date formats, artifact generation for review, email tooling for support workflows with translation bugs, anything else you can find that's relevant to localization work in ios/web. I'd like an ELI5 explainer that's very brief that I can share via email … I'd also like you to dispatch research into the blog repo to make sure we're covering all of the localization related topics as individual blog posts … then agents to draft the new blog posts …"

Research by eight Claude agents across the iOS, web, and blog repos (string catalog, date rulebook, width and snapshot tooling, QA artifacts, API localization, support tooling, cross-repo sync, and a coverage audit of the existing posts); this draft was written by a dedicated agent from that research plus the underlying source, tests, and skill files, then reviewed before publishing. A second pass rewrote each section to lead with the product reason before the mechanism and replaced trimmed fragments with self-contained code examples.
