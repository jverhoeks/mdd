# 016 — Deciding which prose rules to adopt, and the interface for doing it

**Status:** Open. Surveys prior art and frames a design space; proposes no
implementation. Supersedes the method proposed in
[R15](R15-ai-tell-detection-and-prose-evaluation.md) §"What would actually
settle it" only in the sense of widening it — R15's minimal-rewrite pairing
survives as one mechanism among several.

This note records a second attempt at [S06](../spec/S06-documentation-site.md)'s
deferred question — which Vale rules should this repository enable — and the
reason the attempt failed in the same way R15 did. It then surveys what anyone
else has built for the underlying problem, which turns out to be almost
nothing, and frames the interface design space that the failure exposed.

The immediate trigger was an attempt to enable more `write-good` rules and to
evaluate the Google style package. The measurement worked. The judgement did
not, and the way it broke is the useful part.

## The measurement

Vale 3, `write-good` v0.4.1 and Google v0.7.1, all rules enabled at
`MinAlertLevel = suggestion`, over `docs/guide/` and `docs/articles/` —
17,657 words of hand-written prose. 25 rules produced findings; 1,332 findings
in total.

| Rule | Findings | Rule | Findings |
|---|---:|---|---:|
| `write-good.E-Prime` | 522 | `Google.WordListCase` | 17 |
| `Google.Contractions` | 177 | `Google.Will` | 13 |
| `Google.Passive` | 141 | `Google.OxfordComma` | 12 |
| `write-good.Passive` | 141 | `Google.ExcessiveClaims` | 12 |
| `Google.EmDash` | 85 | `Google.Anthropomorphism` | 11 |
| `Google.Semicolons` | 59 | `write-good.Weasel` | 10 |
| `write-good.TooWordy` | 44 | `Google.Parens` | 7 |
| `Google.Acronyms` | 25 | `Google.WordList` | 6 |
| `Google.HeadingPunctuation` | 20 | remaining 8 rules | ≤3 each |
| `write-good.ThereIs` | 18 | | |

Five rules account for 984 findings, 74% of the total. `write-good.Illusions`
produced zero.

Two incidental findings worth keeping:

- **`site/src/content/docs/index.mdx` is not linted.** `.vale.ini` declares
  only an `[*.md]` section, so the `.mdx` file matches no section and silently
  receives no rules. Adding `[*.mdx]` requires the `mdx2vast` binary, which is
  a separate decision.
- **Vale's JSON output carries everything an interface needs.** Each alert has
  `Match` (the flagged text), `Span` (column offsets within the line), the
  message with its `%s` already substituted, plus `Action` (the auto-fix hook)
  and `Link` (rule documentation). A triage harness that reads only
  `Check`/`Line`/`Message` — as this session's first attempt did — throws away
  the span it needs to highlight anything.

### Densities of the contested constructions

Counted directly rather than via a rule, because the complaint about these is
about rate, not about any individual occurrence:

| Corpus | Words | Em-dashes /1k | Semicolons /1k |
|---|---:|---:|---:|
| `docs/guide/` | 11,023 | 2.7 | 1.9 |
| `docs/articles/` | 6,634 | 10.9 | 5.6 |
| `docs/articles/why-mdd-has-its-own-ir.md` | 1,607 | 14.3 | 2.5 |
| `docs/articles/near-lossless.md` | 1,632 | 10.4 | 8.6 |

The em-dash and semicolon problem is concentrated in the synthesised articles,
not the guide — a four-fold difference in both. R15's contraction count is the
mirror image: 0 contractions in 9,730 words of guide, 1 in 6,473 words of
articles.

## Why the judgement failed

Twenty rules were each given to a separate agent, with the rule's definition,
the findings it produced, and instructions to inspect the flagged prose in
context. The fan-out worked: 20 agents, 394k tokens, under four minutes, one
structured verdict each.

The prompt they were given contained this constraint:

> This repository has ONE writer with an established, deliberate voice: long
> sentences, em dashes with spaces, semicolons, "does not" over "doesn't". Do
> not recommend rewriting that voice.

That premise is false, and R15 §"Why the evaluation could not answer the
question" had already diagnosed it: *"Very little of this repository's prose
was written by a human unaided… There is no human-written control group in the
corpus."* Encoding the corpus as a voice to be protected guarantees that any
rule firing often reads as noise. **Counting alerts against an AI-written
baseline measures how much a corpus resembles itself** — and this attempt did
it again, one layer further in, by writing the premise into the instructions
rather than into the analysis.

The maintainer's actual preferences, stated after seeing the results, invert
most of the outcome:

- **Em-dashes.** Over-used and wanted gone from the guide. `Google.EmDash` is
  the wrong rule: it objects to *spacing* around the dash, not to the dash.
- **Contractions.** Wanted. `Google.Contractions` was rejected during the run
  for enforcing "doesn't" over "does not"; that rejection was backwards.
- **Semicolons.** Some are fine; there are far too many. Neither on nor off is
  the right answer.
- **Passive voice.** Wanted flagged, and `write-good.Passive` already does it.
- **E-Prime.** Rejection confirmed. The rule flags every form of "to be".

This splits the twenty verdicts cleanly, and the split is the transferable
result:

- **Verdicts that survive** rest on demonstrable per-sentence failure, and hold
  whoever wrote the prose. `Google.HeadingPunctuation`'s regex `[a-z0-9][.]`
  matches the `1.` in numbered step headings (`## 1. Make a scratch directory`),
  producing 20 false positives and no true ones. `Google.Colons` wants
  "Python" lowercased after a colon. `Google.Headings` misfires on the acronym
  in "Export to PDF (optional)". `Google.We` fires on a first-person pronoun
  inside a quotation from a source.
- **Verdicts that are void** rest on "that is the author's deliberate choice".
  Every one needs redoing. The clearest case is `write-good.TooWordy`, where 34
  of 44 findings were dismissed *because* they were "it is" rather than "it's" —
  which, given the stated preference for contractions, makes them true
  positives.

A related failure mode: two verdicts recommended adopting a rule as a
"free regression guard" on the strength of it having zero real findings, when
in fact the findings existed and were false positives. One agent re-ran Vale
against the repository's own `.vale.ini` — which does not enable Google at all
— got zero alerts, and concluded the harness had manufactured them. Verifying
agent claims against the raw data caught it. It would not have been caught by
reading the verdicts.

## Prior art

Two independent surveys reached the same conclusion: **nobody has built an
interface whose purpose is deciding whether a prose rule is worth having.**
Interfaces that display violations are everywhere. Interfaces that help choose
a rule set number about two, and neither is in the Vale ecosystem.

### The Vale ecosystem

| Surface | What it does | Decision support? |
|---|---|---|
| [Vale Studio](https://studio.vale.sh) | Single-rule authoring playground — paste YAML, paste text, compile, see alerts | No. Authoring one rule, not choosing among many |
| [Config Generator](https://vale.sh/generator) | Five-step wizard: base style, supplementary styles, formats, strictness → `.vale.ini` | Package-level only; rule *counts* are the sole evidence |
| [Style Explorer](https://vale.sh/explorer) | Read-only catalogue: 16 packages, per-rule name/severity/type/description | No toggles, no examples, no export |
| Vale CMS | Closed-source, paid. Browser config editor with resolved-rule preview | Shows what your config *does*, not what a rule would *cost* |
| Vale Server | Discontinued. Dashboard was style-library package management | Package-level |
| `vale-ls`, VS Code extension | Violations, plus quick fixes driven by a rule's own `action` | No "disable this rule" code action |
| CLI | `sync`, `ls-config`, `ls-metrics`, `ls-dirs`, `ls-vars` | No interactive mode, no `fix`, **no HTML output** |

The CLI's output formats are `line`, `CLI`, `JSON`, or a custom Go
`text/template` — which is how SARIF and RDJSONL are produced, and the
available route to an HTML report.

No third-party Vale dashboard, HTML report generator, or report viewer exists.
Published accounts of rule triage from Stoplight, Datadog, Spectro Cloud, ING
and Grafana all describe it as a manual spreadsheet exercise: count by rule,
eyeball samples, assign a bucket.

**The near-miss is an LLM prompt.** [`vale-cli/agent-tools`](https://github.com/vale-cli/agent-tools)
(MIT, August 2026) ships a `/vale:triage` skill described as *"Turn a large
first Vale run into a decision per rule — fix, downgrade, or switch off."* Its
workflow is `--output=JSON`, group by `.Check`, take the top 20, sort each into
Fix / Downgrade / Disable. Its framing of why counts matter is worth quoting:

> A team that hears "18,000 alerts" reaches for `MinAlertLevel`; a team that
> hears "four rules are 80% of this" makes a decision.

Its prohibitions are the more useful half — do not raise `MinAlertLevel`
(*"it hides alerts without deciding anything, and the decisions are the
point"*), do not disable a rule that is only noisy on legacy content when
downgrading would do, and do not recommend a bucket without saying what it
costs. It shows counts but no samples, and persists nothing beyond the chat
transcript and a `.vale.ini` comment.

**The engine exists, paywalled and agent-only.** Vale CMS's MCP server exposes
`diff_rule` (*"compares which alerts an edit adds or removes across a
corpus"*), `diff_style`, `audit_style` (*"what the style costs to run"*), and
`stress_rule`, which generates near-miss inputs to hunt false positives. This
is corpus-scale rule-impact analysis with no human-facing view.

### Other prose linters

`textlint`'s [playground](https://textlint.org/playground/) has genuine
enabled/disabled rule lists with live re-linting — but five hardcoded rules and
zero persistence: no `localStorage`, no `.textlintrc` export, no URL state.
`proselint`'s `.proselintrc` is literally `{"checks": {id: bool}}`, the exact
artifact a triage UI would emit, with only `--dump-config` to touch it.
`retext`, `alex` and `write-good` have nothing.

**LanguageTool has both halves, joined by nothing.** Its
[Rule Editor](https://community.languagetool.org/ruleEditor2/) step 4 is
"Evaluate error pattern" → "Check Evaluation Results", which runs a candidate
pattern over a Lucene-indexed corpus and surfaces real sentences it would fire
on. Candidate rule → real-corpus evidence → human judgement. But it is
oriented at authoring a *new* rule, and its output is XML on the clipboard:
no per-match accept/reject, no persisted decision. Separately, its nightly
regression diffs are static HTML tables classifying matches ADD / REM / MOD
against a large corpus, filterable by rule id — corpus-scale, rule-keyed
violation review, read-only, after which the reviewer argues on a forum. Its
product clients offer "Turn off rule everywhere" from a violation popup: a
durable decision recorded from a concrete example, but reactive and
one-at-a-time.

### The one true match, and it is legacy

**Acrolinx's Guidance Wizard** is the only interface found whose stated purpose
is deciding whether a guideline should be on. It is premised on evidence —
documented as working best once writers have been using the product a while,
because it needs collected issue data.

Design features worth taking:

- Guidelines are ranked **by issue frequency**, and for style and grammar the
  wizard **explicitly declines to recommend**. It ranks and leaves judgement to
  the human.
- **Progressive disclosure of evidence.** For a flagged word: hover to see one
  example sentence, click to see every sentence where it was found.
- **Thresholds set against your own distribution.** For length guidelines,
  configuring one shows a histogram of sentence/paragraph lengths in your own
  content — occurrences against detected length — so the threshold is chosen
  against reality rather than guessed.
- **The action vocabulary contains narrowing primitives**, not just on and off:
  off / leave on / restrict to certain document-structure contexts / change the
  word-or-sentence limit. The last two map onto Vale's `scopes`, `filters` and
  `views`.
- Decisions are routed by kind: a flagged word becomes a terminology entry, a
  spelling exception, or stays reported.

Its documented weakness is the thing to beat: **past changes cannot be
retrieved.** No decision history and no captured rationale. Disabling a
guideline inside a goal also silently flips that goal's preset to "Custom".

Status caveat: Acrolinx rebranded as Markup AI in September 2025 — the CEO
called it *"more of a restart than a rebrand"* — and the relevant analytics
dashboards were retired in October 2025. Treat the Guidance Wizard as design
precedent, not as a live product to evaluate.

### Adjacent prior art in code linters

- [`eslint-nibble`](https://github.com/IanVS/eslint-nibble) — *"Ease into
  ESLint, by fixing one rule at a time."* Counts first, arrow-keys to select a
  rule, then autofix or show the violation list. The closest existing
  interaction model; its buckets are "fix now / not now" rather than
  on / off / narrow.
- [ESLint Config Inspector](https://github.com/eslint/config-inspector) —
  official local web UI showing which rules are on, at what severity, and which
  config layer enabled each. Read-only: it explains a config, it does not help
  choose one.
- **ESLint's `--init` autoconfig** inspected your source files and derived a
  rule configuration from what the code already did — "Enabled 275 out of 275
  rules based on 2 files." **Removed in v8.0.0.** It died of unparseable
  files, a plugin-ordering bug, and generating configs containing deprecated
  rules. That it shipped for years and was then deleted is evidence about the
  approach, not trivia.
- **RuboCop `--auto-gen-config`** is the closest live thing to rule *fitting*:
  for cops with an `EnforcedStyle`, "if one style is used for all files, these
  cops will add the settings for the style being used". Metrics cops get a `Max`
  fitted just high enough that nothing reports. The documented next step is
  human — copy from `.rubocop_todo.yml` "for everything you consider in line
  with your style".
- **Bulk suppression is a different thing.** ESLint `--suppress-all`, oxlint,
  `detekt`, `ktlint`, PHPStan, Psalm `--set-baseline`, `mypy-baseline` all adopt
  a rule unconditionally and grandfather existing violations. None uses the
  corpus to decide whether the rule is right.

## Two mechanisms

The failure above is a failure of *elicitation*: the maintainer's preferences
were inferred by an agent from prose the agent was told to treat as
authoritative. There are two ways to get real preferences, and they are
complementary rather than competing.

|  | Elicit — ask | Observe — mine edits |
|---|---|---|
| Signal | Preferences stated against samples | Preferences demonstrated in real edits |
| Cold start | Works today on existing prose | Needs the author to be editing |
| Cost to the author | Real effort per rule | Free; rides on work already happening |
| Abstraction | Judging a rule from samples | No abstraction — the edit *is* the datum |
| Coverage | Every candidate rule, systematically | Only what the author happened to touch |

### Observing: mining the author's own edits

[`BruceEckel/ThinkingInPython`](https://github.com/BruceEckel/ThinkingInPython)
implements this, and it is the most developed instance found in either survey.
Eckel's stated strategy: *"edit a chapter, ask AI to derive guidelines from my
edits, then AI-apply those guidelines throughout the book before editing the
next chapter"*, expecting asymptotically-decreasing edits. Two skills share one
store; `bruce_edit_db.md` currently holds zero rules and zero candidates, so
the machinery is carefully designed and **unproven in practice**.

The mechanisms worth taking:

- **Provenance from git trailers.** A commit without `Co-Authored-By: Claude`
  is the author's own work. Where the human rewrote AI prose, the edit says
  directly how the AI should write, and that is the strong signal; where the
  human rewrote their own older draft, it may be the chapter maturing rather
  than a standing preference, and is weighted lower. This matters here because
  it answers R15's open question 5 — whether a human-written control group
  exists anywhere. **Git records the provenance the corpus itself does not.**
- **Word-diff, not line-diff.** The chapters use Semantic Line Breaks and a
  reflow target re-wraps paragraphs, so a line diff shows changed lines whose
  words are identical, and reading one as an edit manufactures rules out of
  whitespace. `git diff --word-diff=porcelain --ignore-all-space`. This applies
  directly to any repository doing deterministic reflow, including this one
  ([S46](../spec/S46-prose-checks.md)).
- **Classify before inducing.** Each prose change is sorted local /
  generalizable / contradicts-a-standing-decision, with a transplant test:
  would this fix still apply in a different chapter on a different subject? If
  it depends on the surrounding paragraph, it was local.
- **Two independent sightings to promote.** One sighting is a candidate; a
  second, in a *different* chapter, makes it a rule. Only rules are applied.
- **Permanent retirement with a reason**, explicitly so that "every capture
  round re-proposes the same rejected rule" stops happening. This session
  re-proposed conclusions R15 had already settled; the ledger is the structural
  fix.
- **Every rule carries a test answerable from one sentence.** "Cut 'itself'" is
  not a test; "cut 'itself' where the sentence means the same with it deleted"
  is. No usable test means it stays a candidate however convincing it sounds.
- **Narrow beats broad**, because a second sighting widens a rule later and
  nothing narrows one that was too broad from the start.
- **Firing counts are the primary wrongness signal**, available before a human
  reads any prose. Hard stops: one rule firing more than 40 times in a chapter,
  or more than 15× its own per-chapter average, or accounting for more than half
  of all edits in a sweep, means hand it back for narrowing.
- **Additive edits are rarer and worth more.** *"A rule set made only of cuts
  produces compliant, characterless prose."* Directly relevant to the em-dash
  question: removing dashes is subtractive, and what replaces them is the
  actual decision.
- **Round capped at 8 proposals**, with the local:generalizable ratio reported
  as a self-check — *"a round where nearly everything became a rule is a broken
  round."*

A simpler public analogue is
[`jzOcb/writing-style-skill`](https://github.com/jzOcb/writing-style-skill):
record the AI draft and the human-edited final, diff them, have an LLM extract
rules, grade them P0/P1/P2 by confidence, auto-apply P0 to the skill file. No
provenance weighting, no two-sighting promotion, no firing-count safety net, no
retirement ledger.

### The academic framing

The field does not call this "style guide inference"; searching that term
returns almost nothing. The terms are **latent preference inference** and
**edit intention classification**.

- **PRELUDE / CIPHER** ([arXiv 2404.15269](https://arxiv.org/abs/2404.15269),
  NeurIPS 2024) is the closest match. It assumes user edits are driven by a
  latent preference *expressible as text*, has an LLM infer a natural-language
  preference from each edit, and retrieves preferences for similar contexts at
  generation time. No fine-tuning, so the learned artifact stays human-readable
  and user-editable — the right property for something that has to become a
  Vale config. Its metric is the one that matters: **edit-distance cost over
  time**, i.e. did the human have to edit less.
- **IteraTeR** ([ACL 2022](https://aclanthology.org/2022.acl-long.250/),
  Grammarly Research) supplies a usable taxonomy: MEANING-CHANGED versus
  NON-MEANING-CHANGED, the latter splitting into FLUENCY / COHERENCE / CLARITY
  / STYLE. Only the last is a style rule, and separating them is the step Eckel
  performs by hand as local-versus-generalizable.
- **arXivEdits** ([EMNLP 2022](https://arxiv.org/abs/2210.15067)) treats edit
  extraction as span alignment rather than `diff`, with a neural CRF sentence
  aligner at 93.8 F1. The rigorous fallback if word-diff proves insufficient.
- **WikiAtomicEdits** ([EMNLP 2018](https://arxiv.org/abs/1808.09422)) is raw
  material only, but carries a relevant finding: language produced *during
  editing* is distributionally different from language in ordinary corpora.

Commercial claims in this space are mostly manual configuration with AI
labelling. Grammarly Business style rules are hand-authored one at a time or
CSV-imported, with no pre-enable preview — only post-hoc Viewed / Accepted /
Dismissed counts per rule, which is a usable retire signal but shows no
examples. Writer.com's style guide is a hand-built artifact and its custom
rules are built for you by their support team. Acrolinx/Markup AI's "digitize
your style guide" is preset-and-goal selection. The one genuine exception is
**Jasper's Automatic Style Guide Setup** — upload one to three documents, and it
extracts preset toggles plus replacement rules — which is open beta, tier-gated,
constrained to a fixed schema, and self-described as *"an AI-recommended
starting point, not a guaranteed auto-import"*.

### Fitting a rule set to a known-good corpus

The remaining mechanism is neither eliciting nor observing: take prose the
author already considers good, run the candidate rules over it, and read the
violation rate as evidence about the rule. A rule that fires constantly against
writing the author is happy with contradicts their style. A rule that fires
rarely is safe to adopt. The middle band is what needs human judgement.

This is Eckel's firing-count guardrail turned around — he uses rate as a safety
valve on induced rules; here it is the primary signal. It is also the direct
inverse of this session's error, which read high counts against an AI-written
corpus as evidence about the *rule* when the corpus had no standing to judge.

Prior art for this framing is thin to nonexistent for prose. ESLint built it
and deleted it; RuboCop's `EnforcedStyle` majority-inference is the only live
implementation, and for code. **No published method or tool takes a corpus plus
a candidate rule set and outputs which rules to adopt.**

The stylometry literature supplies the statistics without assembling them:
Burrows's Delta with a calibration step that converts raw deltas to
probabilities by fitting on known-author pairs, and per-feature minimal
reliable sample size screening, which is the direct analogue of per-rule
calibration confidence. Its warnings apply — comparing texts of very different
lengths introduces bias, so downsample to the shortest; genre is a confound;
and stylistic drift argues for time-slicing a baseline corpus rather than
pooling decades of writing.

The obvious control corpus for this repository is the maintainer's own writing
from outside it. That is untried.

## The interface design space

No design is proposed here. What the failure and the survey establish is a set
of constraints any design has to satisfy.

**The artifact the human looks at must not contain the agent's verdict.** The
whole failure mode above is an agent's judgement standing in for a preference.
Counts, densities and verbatim samples; no recommendation to anchor on.
Acrolinx's wizard declines to recommend for exactly this class of decision.

**The flagged span has to be visible without hunting for it.** A plain-text
ballot listing `E-Prime` findings as whole lines requires the reader to
re-derive which word fired, for 522 findings. `Match` and `Span` are in the
JSON; the medium has to be able to render them. This is the argument for HTML
over a text file.

**One line of context is not enough, and all of it is too much.** Acrolinx's
hover-for-one-example, click-for-all is the precedent. R15's open question on
the unit of comparison stands: sentence isolates a rule best but strips the
context that makes a construction good or bad, and some complaints — rhythm,
monotony — are invisible below section level.

**The decision vocabulary needs narrowing primitives, not just on and off.**
The semicolon case is the proof: neither enable nor disable is correct, and the
honest answer is a rate. Candidate vocabulary, merging Vale's triage buckets
with Acrolinx's actions: gate / warn / restrict-to-scope / set-a-threshold /
drop. The threshold case needs the author's own density distribution displayed,
as Acrolinx does with its length histogram.

**A rule that is right in aggregate and wrong per instance needs a different
shape of check.** This is R15's open question 4, and the em-dash and semicolon
findings are both instances. A per-occurrence gate is the wrong instrument for
a complaint about rate. Vale has no density-rule primitive, so this is either a
custom check or a separate script.

**Decisions must persist with their reasons, and rejections must be
permanent.** Acrolinx's inability to retrieve past changes is the gap; Eckel's
retirement ledger is the fix; Vale's own advice — *"name it in the config with
a comment saying why"* — is the minimum. The store should be boring and
diffable. proselint's `{"checks": {id: bool}}` is the right level of ambition
for the machine-readable half.

**Throughput matters at this scale.** 1,332 findings across 25 rules is a small
corpus. Judging per-finding does not scale; judging per-rule from a sample
requires the sample to be honest about what it omits.

**Generalisation is an open question.** `mdd`'s domain is Markdown in git, so
edit-mining is unusually natural here — the provenance and the diffs are
already present, and other users of `mdd` who want good-looking documentation
have the same problem. But S06 already places "Vale as an `mdd` feature" out of
scope, and nothing here settles whether a preference harness is a repository
tool or a product feature.

## What this note does not settle

S06's open question 2 stays open. The Google package measurement stands, but
every verdict resting on house voice is void, so "three of twenty-five rules"
is not a result — it is an artifact of a broken premise. The five rules
rejected on sight are in the same position: four of the five were rejected for
reasons the maintainer contradicted.

The rules whose rejection survives, because their failures are demonstrable
per sentence: `Google.HeadingPunctuation`, `Google.Colons`, `Google.Headings`,
`Google.We`, `Google.Quotes`, `Google.Parens`, `Google.Timeless`, and
`write-good.E-Prime`.

## Open questions

1. Which corpus calibrates the rules? The maintainer's writing from outside
   this repository is the obvious control group and has never been assembled.
   How much is needed, and does mixing registers (this project's prose versus
   whatever else exists) reintroduce the genre confound stylometry warns about?
2. Elicit or observe first? Edit-mining is free but has no cold start;
   a review interface works today but costs the maintainer real time. Building
   both is more than this problem is worth.
3. Is the unit of judgement the rule or the finding? Per-rule from a sample is
   affordable and is what every existing tool does. Per-finding is honest and
   does not scale. R15's minimal-rewrite pairing is a third unit again.
4. Does a density check belong in Vale at all? A rate complaint may be better
   served by a separate script reporting per-file densities against a budget,
   leaving Vale to the per-occurrence rules it is built for.
5. Should the em-dash reduction happen before or after the harness exists? It
   is the clearest known preference and the largest single edit; doing it by
   hand first removes the most valuable calibration signal from the corpus, and
   doing it after means the guide stays wrong for longer.
6. Repository harness or `mdd` feature? Edit-mining over Markdown in git is a
   natural fit for this tool's domain, and S06 explicitly scoped prose linting
   out. Whether preference elicitation is a different question from prose
   linting is unresolved.
7. Is the `mdx2vast` dependency worth taking to lint `index.mdx`, or should the
   page move to `.md`?

## Artifacts

Nothing from this session was kept. The throwaway scripts — a report bucketer,
a per-rule splitter, a text-ballot generator, a sample printer, and a density
counter over `docs/guide/` and `docs/articles/` — were deleted, as was the
synced Google package (v0.7.1, MIT), which `vale sync` refetches on demand. The
experiment configuration was a separate `.vale.ini` enabling `Vale`,
`write-good` and `Google` at `MinAlertLevel = suggestion`, and the 20 per-rule
verdicts exist as structured JSON in the session transcript only.

The measurements reproduced above are the durable part. If a harness is built,
it should be built inside the repository rather than recovered from scratch
space — the same conclusion R15 reached about its own artifacts, and the reason
the numbers here are in the note rather than in a file.
