> **This is a mirror.** The canonical, living edition of this paper is published at
> **[isimplifyme.com/whitepapers/the-aeo-standard](https://isimplifyme.com/whitepapers/the-aeo-standard)** — it revises there first; this mirror follows.
> Joseph W. Elstner, Founder & Principal Architect, iSimplifyMe · Published 2026-07-10 · License: [CC BY 4.0](LICENSE)

## What this white paper is

The AEO Standard is iSimplifyMe's 100-point scoring system for Answer Engine Optimization — the discipline of structuring content so AI answer engines like ChatGPT, Perplexity, Gemini, and Claude can extract, cite, and recommend it. This page offers the **print-ready PDF edition** of the standard; the living document, with its changelog and current revision, is openly published at [the standard's canonical home in the Lab](https://isimplifyme.com/labs/aeo-standard).

The PDF exists for the ways paper still wins: sharing with a team, annotating in review, and reading the whole rubric end to end without a browser. The content matches the web edition at the version printed on its cover.

## What's inside

The white paper carries the complete standard, not a summary. In fourteen pages of reading you get:

- **The two gating rules** — hidden machine-only content and fabricated authority — that reject a page outright, regardless of its point total.
- **All seven scoring sections with point values:** Substance & Originality (25), RAG/Retrieval Readiness (20), Atomic Answer Blocks (15), Structured Data & Schema (10), Semantic HTML & Headings (10), Internal Linking & Fan-Out (10), and SEO Meta & Technical (10) — every check marked mechanical or judgment.
- **The atomic answer block specification** — the 40–60-word, self-contained, visible answer format that answer engines can lift and cite.
- **Verdict thresholds and the scorecard format** a conforming scorer reports.
- **The common failure patterns** we see most across production audits, in rough order of frequency.
- **The versioning policy and license** — CC BY 4.0, versioned like software, so a score always references a specific revision.

## Who it's for

The standard is written for the people who have to make citation-worthiness operational: content leads deciding what "publishable" means, technical SEOs adapting to answer engines, and the engineers wiring quality gates into a publishing pipeline. If a generic model could have written your page, no amount of markup will make it citable — the standard scores both halves and says so plainly.

## How to use it

Score something. The fastest way to understand the rubric is to watch it grade a page you care about: the [free AEO scanner](https://isimplifyme.com/tools/aeo-scanner) runs any URL in the browser, and the `aeo-scan` CLI preview on npm checks the core mechanical signals from a terminal. When the gap between your score and the publish threshold looks like architecture rather than copyediting, [that is the work we do](https://isimplifyme.com/services/aeo-infrastructure).
