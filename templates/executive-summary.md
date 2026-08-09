<!-- file: templates/executive-summary.md -->
<!-- version: 1.0.0 -->
<!-- guid: d66beaa1-7efa-4dc2-ae01-6083b9238695 -->
<!-- last-edited: 2026-08-09 -->

# Executive summary template

An executive summary explains completed work to someone who **makes decisions
about it but does not read code** — funding, staffing, risk acceptance. It is
not a release note, not a status report, and not a changelog. Those already
exist and are written for engineers.

The audience question is always the same: *was this worth doing, and what do I
now need to decide?*

Summaries live in `docs/executive-summaries/` in the repository the work
happened in. This document is the org-wide template; consuming repositories
reference it rather than keeping their own copy.

## Which shape to use

| Shape | Use when | Length |
| --- | --- | --- |
| **Single change** | One incident, defect, or self-contained piece of work | 30–120 lines |
| **Roundup** | A month, or a planned wave spanning many pull requests | 100–600 lines |

Most summaries are the single-change shape. Reach for the roundup only when
one-section-per-change would bury the story.

## Rules that apply to both shapes

1. **Plain language. No jargon, no tool names, no file paths, no code.** Say
   "the automated tests that drive a real browser", not "the Playwright suite".
   Where a technical term is genuinely unavoidable, define it in the sentence
   that uses it.
2. **Lead with impact, not activity.** "The library could not be sorted at all"
   beats "refactored the sort control". The reader is buying outcomes, not
   commits.
3. **Anchor every claim to evidence** — a pull request number, a commit, or a
   measured figure. Cite inline as `#123`. Never invent hours-saved or
   cost-avoided numbers; use the counts you actually have. A claim with a number
   behind it is checkable; one without is a feeling.
4. **Never overclaim.** Work that is merged but not yet rolled out is "the
   capability shipped, rollout pending". Work on an unmerged branch is in
   flight and does **not** belong under `**Shipped:**`. A stakeholder who later
   discovers one claim was optimistic will discount every other claim in the
   file.
5. **Say what was actually verified, and how.** Distinguish "the tests pass"
   from "it was run against real data" from "not yet verified". Never let a
   reader assume a stronger check than the one performed. If the strongest
   evidence is a passing test suite, say so plainly — a green suite is evidence
   about the tests, not about the system.
6. **Be honest about what went wrong, including self-inflicted damage.** A
   summary that only lists wins is marketing and gets read as such. The most
   valuable entries are usually the defects found — especially the ones the
   project caused itself, because those are precisely what justify the cost of
   the work that found them.
7. **Say what is still open**, including what the work deliberately does not
   cover. This is what makes the document worth trusting.
8. **Do not pad.** If a period was quiet, say it was quiet and say why. A short
   honest summary is more credible than a long one inflated to look busy. A
   pull-request count is a poor proxy for value in either direction: a month
   that produced only a design can be the month that made the next one possible,
   and a month that produced one careless line can cost more than it delivered.

Formatting: wrap prose at 80 columns, and bold the load-bearing sentence in a
long paragraph so the document can be skimmed. Carry the standard four-line
file header — unlike changelog and TODO fragments, executive summaries are
ordinary documents and the header rule applies.

## Naming, and the update convention

```text
docs/executive-summaries/<YYYY-MM-DD>-<short-slug>-executive-summary.md
```

The date is the period the summary covers — for a roundup, its start or end,
chosen consistently within a repository — **not** the day it was last touched.
Put the real write date in `last-edited:`.

**Updating an existing summary is the norm, not the exception.** When work
continues on the same subject, edit the existing file rather than adding a
second one: bump `version:` (minor for new material, patch for corrections),
refresh `last-edited:`, and extend the `**Shipped:**` range. **The filename
never changes.** A reader should be able to follow one subject in one file.

## Scaffold — single change

````markdown
<!-- file: docs/executive-summaries/YYYY-MM-DD-slug-executive-summary.md -->
<!-- version: 1.0.0 -->
<!-- guid: GENERATE-A-NEW-UUID -->
<!-- last-edited: YYYY-MM-DD -->

# A title that states the problem in human terms

## What was wrong

What the person on the other end actually experienced, and why it mattered.
Open here even when the work was preventative — "nothing was visibly broken,
and that was the problem" is a legitimate opening.

## What happened

What was done and what it cost. If a fix uncovered further problems, say how
many, and separate the ones already fixed from the ones now waiting on a
decision.

## What this means going forward

What is now true, what is still open, and anything the reader must decide.
State the limits of the work plainly — what it does not cover, and why.
````

## Scaffold — roundup

````markdown
<!-- file: docs/executive-summaries/YYYY-MM-DD-slug-executive-summary.md -->
<!-- version: 1.0.0 -->
<!-- guid: GENERATE-A-NEW-UUID -->
<!-- last-edited: YYYY-MM-DD -->

# Executive Summary: <period or wave name>

**Shipped:** PRs [#A–#B](https://github.com/OWNER/REPO/pulls?q=is%3Apr+is%3Amerged+merged%3AYYYY-MM-DD..YYYY-MM-DD),
covering YYYY-MM-DD through YYYY-MM-DD (N merged; X files changed,
+ins/−del lines. Note anything excluded, such as routine dependency bumps,
and why)
**Prepared:** YYYY-MM-DD
**Related docs:** links to deeper write-ups and sibling roundups, linked
rather than repeated here.

One or two sentences on scope, including what is deliberately out of scope,
and whether the document is grouped by theme or told as one story.

## Executive Summary

- **Theme in bold.** Two to four sentences a non-engineer can act on, with
  pull request numbers as evidence (#A, #B). One bullet per theme, not one per
  pull request.

**Highest-risk items** — what a stakeholder most needs to know, because each
one touched safety, could have destroyed data, or went undetected. Include
anything still unresolved or awaiting a decision.

- **#PR** — the defect in one sentence, in plain language, with its
  consequence stated.

**Verification note:** exactly what was checked and how — tests, automated
checks, a real run against production data — and explicitly what was *not*
checked.

## What changed, in plain terms

### 1. Theme name

**What it was:** the situation before, in plain language.

**Why it mattered:** the consequence in terms the reader cares about — cost,
risk, downtime, someone's time. Not the technical consequence.

**The fix:** what was done, with pull request numbers, how it was verified,
and the measured result if there is one.

## What did not get done

Scope deliberately deferred, blocked, or abandoned, and what each is waiting
on. Naming this is what stops the next reader assuming it was finished.

## Cost and effort notes

Optional. Volume delivered, defects caught before they reached production,
work that had to be redone and why, or time lost to external causes such as an
upstream outage. Anything that helps a reader judge whether the spend was
sound.
````

## A note on writing a set of these

Roundups covering consecutive periods are worth more read together than
separately, and it is worth saying so explicitly in each one. A quiet month
whose single mistake creates the next month's workload, and a design month
whose output is a document rather than code, are both badly misread in
isolation. Cross-link siblings, and where one period's cost was created by
another's decision, say which.
