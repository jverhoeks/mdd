# S46: prose checks and reflow

**Purpose:** `mdd prose` is a group of deterministic, model-free checks and fixers over a Markdown corpus — semantic-line-break reflow, mechanical prose lint, heading-anchor validation, a project-owned vocabulary gate, and a freshness gate — so a corpus can be gated in CI reproducibly and prose diffs stay reviewable.

**Status:** Draft

## Introduction

`mdd` already owns "Markdown corpora in git at scale": it mirrors Confluence
and SharePoint into Markdown ([S14](S14-confluence-sync.md),
[S18](S18-sharepoint-sync.md)), searches across mirrors
([S19](S19-search-command.md)), and applies judgement to their contents with
[`mdd ai rewrite` / `mdd ai index`](S21-ai-rewrite-and-index.md) and
[`mdd ai review`](S22-ai-review-command.md).

What it has no answer for is the other half of corpus hygiene: the
**mechanical** half. Nothing in `mdd` today can be put in front of a merge and
asked "is this Markdown well-formed as prose?" without calling a model. Every
existing quality answer is either a model's opinion (`mdd ai review`, which is
triage material for a human, not a gate) or is scoped to `mdd`'s own repository
rather than shipped to users (the `docs-vale` and `spec-check` `mise` tasks).

### The motivating use case

The case this spec is written for is **agent-assisted authoring of long-form
Markdown** — a book, a course, a documentation site — where the human is the
author and the agent is the editor. That loop has one hard prerequisite:

> **A prose change must produce a reviewable diff.**

If prose is hard-wrapped to a fixed column, editing four words in the middle of
a paragraph reflows every subsequent line, and `git diff` reports the whole
paragraph as changed. Neither the human nor the agent can see what actually
happened. Review degrades into re-reading, the human stops reading carefully,
and the agent — which has only the diff — cannot reason about the change at
all. The failure is quiet and it compounds: it is worst exactly when the corpus
is large and the edits are frequent, which is when a review loop matters most.

The fix is well known and is the load-bearing feature of this spec: **semantic
line breaks**. One sentence per line, with a still-overlong sentence broken at a
top-level clause boundary. A one-sentence edit then produces a one-line diff.
Everything else here is smaller, but it is the same shape of problem: a
deterministic property of the text that a machine can check and a machine can
fix, so a human never has to.

### Prior art

The design was inspired by [Bruce Eckel](https://github.com/BruceEckel)'s
Claude-assisted authoring setup for his book repository,
[ThinkingInPython](https://github.com/BruceEckel/ThinkingInPython), which pairs
an agent with a set of deterministic prose gates. The idea that reviewable
prose diffs are the foundation of an agent-assisted authoring loop, and that
the surrounding checks should be mechanical rather than model-driven, is his.

This is an independent implementation of similar ideas. No material — code,
rule text, file layout, wording, or configuration schema — was copied from that
repository, and it was not read while writing this spec. Where a name was
needed, one was invented here. The generic-to-any-Markdown-corpus subset is
what lands in `mdd`; see [Out of scope](#out-of-scope) for the deliberately
excluded book-specific half.

### Deterministic, not AI

Every check in this spec is a pure function of the bytes on disk plus the
project's configuration. No network call, no credential, no model, no cache
that changes the answer. That is not an aesthetic preference — it is the
requirement that makes a CI gate worth having:

| Property | `mdd prose` | [`mdd ai review`](S22-ai-review-command.md) / [`rewrite`](S21-ai-rewrite-and-index.md) |
|---|---|---|
| Answer for identical input | always identical | may vary by model, version, temperature |
| Needs a gateway, token, network | no | yes |
| Suitable as a merge gate | yes | no |
| Output | findings with a rule id | prose judgement for a human |
| Failure mode | a false positive the author suppresses | a plausible wrong answer |

A gate that can disagree with itself on a rerun is not a gate; it is a flaky
test that blames the author. So the split is a rule, not a default:
**nothing under `mdd prose` may call a model, and no `mdd ai` subcommand may be
relied on to gate a build.** The two compose in one direction — run
`mdd prose check` first so the model never spends tokens on a corpus with a
broken anchor in it — and the composition is a user's choice, not wiring inside
either command.

The same split explains why the reflow lives here rather than inside
[`mdd ai rewrite`](S21-ai-rewrite-and-index.md). `rewrite` already protects
frontmatter, fenced blocks and tables from the model; reflow needs the same
masking but must produce a byte-identical result on a rerun, which a model
cannot promise.

## Requirements

### The group

- `mdd prose` is a top-level command group with the subcommands below. It
  follows the two-level, `verb-noun` shape [S35](S35-argparse-cli-parsing.md)
  mandates; no subcommand nests further.

  ```
  mdd prose reflow      [PATH ...] [--write] [--width N] [--json] …
  mdd prose lint        [PATH ...] [--json] …
  mdd prose anchors     [PATH ...] [--json] …
  mdd prose vocabulary  [PATH ...] [--json] …
  mdd prose freshness   [PATH ...] [--json] …
  mdd prose check       [PATH ...] [--json] …
  ```

- `mdd prose check` runs every *enabled* check in one pass over one parse of
  each file, and is the intended CI entry point. Enablement is per project (see
  [Configuration](#configuration)); an unconfigured project gets the
  conservative default set, and `reflow` is **not** in it.
- Every subcommand accepts zero or more `PATH` positionals — files or
  directories, default `.` — walks directories for `*.md` and `*.qmd`, and
  honours [`.mddignore`](S39-mddignore.md) rooted at the walk root, including
  its `prune_dir` fast path. `--ignore <path>` is accepted with the same union
  semantics as the sync commands.
- Scoping is **path-based, not mirror-based.** The registered-root-source
  registry [`mdd search`](S19-search-command.md) uses answers "where are all my
  mirrors" — the wrong question for a gate, which runs inside one repository's
  checkout and must not silently reach into another. A user who wants to lint a
  configured mirror passes its path.

### Reporting and exit status (all subcommands)

- A finding is emitted as one line on **stdout**:

  ```
  <path>:<line>:<column>: <severity>: <rule-id>: <message>
  ```

  matching the shape `scripts/spec-check.py` and `scripts/check-mdd-commands.py`
  already print, which is also the shape every editor's error parser
  understands. `column` is 1-based and may be `0` when a finding is
  whole-line or whole-file.
- `--json` switches to **line-delimited JSON**, one object per finding, the
  same convention as [`mdd search --json`](S19-search-command.md) — not a
  wrapping array, so a consumer can stream it. Fields:

  ```json
  {"path": "docs/ch01.md", "line": 42, "column": 17,
   "severity": "error", "check": "lint", "rule": "space-before-punctuation",
   "message": "space before ';'", "excerpt": "… the plan ; then …",
   "fixable": true}
  ```

  `--json` implies no colour and no progress output on stdout, for the reason
  [S19](S19-search-command.md) already records: ANSI codes corrupt machine
  output.
- Progress, warnings and the run summary go to **stderr** via the standard
  logger. Findings go to stdout. A CI job can therefore keep the findings and
  discard the noise.
- **Severity is a property of the rule, set by the project.** Each rule
  resolves to `error`, `warning`, or `off`. `--min-severity {error,warning}`
  (default `error`) decides what makes the exit code non-zero; everything at or
  above `warning` is always *printed*.

  This exists because `mdd`'s own `docs-vale` task has to run Vale twice — once
  with `--no-exit` to show warnings, once at `--minAlertLevel=error` to decide
  the outcome — since Vale conflates "print it" with "fail on it". Separating
  the two is a one-line decision at design time and unfixable afterwards.
- Exit status: `0` clean (nothing at or above `--min-severity`), `1` findings,
  `2` argparse usage error. `1` is also returned for an operational failure
  (unreadable file, malformed config), with the reason logged at `error`. There
  is no distinct "found problems" versus "could not run" code; CI treats both
  as red, and the log says which it was.
- A run summary on stderr: `mdd prose check: 3 errors, 11 warnings in 214
  files (2 fixable with --write)`, or `mdd prose check: clean (214 files)`.

### Suppression

- A finding is suppressed for the following line by an HTML comment on its own
  line:

  ```markdown
  <!-- mdd-prose-ignore: punctuation-outside-quote -->
  ```

  A comma-separated list of rule ids suppresses several; the bare form
  `<!-- mdd-prose-ignore -->` suppresses every rule on the next line. A
  file-wide form, `<!-- mdd-prose-ignore-file: reflow -->`, may appear anywhere
  in the file and disables the named rules (or all of them) for the whole file.
- HTML comments are used because they are invisible in every Markdown renderer
  and survive round-tripping through the [IR](S30-markdown-ir-conversion.md).
- A suppression whose rule id does not exist is itself a finding
  (`unknown-suppression`, `warning`). A suppression that suppresses nothing is
  reported under an opt-in `unused-suppression` rule, default `off` — useful
  when cleaning up, noisy in normal work.

### The shared line classifier

- Every check consumes **one** classification of the document, produced once
  per file, that assigns each line — and each span within a prose line — to
  exactly one class. No check may re-derive "is this prose?" for itself.
- The classes are at minimum: `frontmatter`, `fenced-code`, `indented-code`,
  `html-block`, `html-comment` (including multi-line), `table`, `block-quote`,
  `heading`, `list-marker`, `link-target`, `inline-code`, `footnote-ref`,
  `math`, `blank`, `prose`.
- This is a requirement rather than an implementation detail because the
  alternative has a specific, ugly failure: five checks each with their own
  idea of where a fenced block ends will disagree on an edge case, and the
  reflow will rewrite a line the lint told the author was code. A single
  classifier makes that class of bug impossible rather than unlikely.
- The classifier **fails closed**. An unbalanced fence, an unterminated HTML
  comment, or frontmatter that does not close is a `parse-error` finding and
  the file is skipped by every check — and, critically, never written to.
- Mask fidelity is the correctness bar: a masked span is passed through
  byte-for-byte. A check may look at a masked span (`vocabulary` may want to
  ignore it; `anchors` must read link targets) but no check may rewrite one.

### `mdd prose reflow` — semantic line breaks

The load-bearing subcommand.

**Behaviour**

- Default mode is **check-only**: report every prose line that differs from its
  reflowed form and exit non-zero. `--write` applies the reflow in place. There
  is deliberately no `--dry-run` flag: check-only *is* the dry run, and offering
  both invites the combination `--write --dry-run`, whose meaning nobody can
  guess.
- Reflow rewrites **only** lines classified `prose`. Headings stay on one line
  however long. Table rows, code, HTML and frontmatter are untouched.
- Within a paragraph, output is **one sentence per line**. A sentence longer
  than the target width is broken further, at top-level clause boundaries.
- A list item's continuation lines are indented to the item's content column so
  the list still parses. A block quote's continuation lines keep the `> `
  prefix. Nested structure is preserved.
- Reflow never changes the number of blank lines, the order of blocks, or any
  character other than the whitespace it replaces with a newline (and the
  newline it replaces with a space when joining a paragraph that was
  hard-wrapped).

**Clause breaking — the part that is easy to get wrong**

- Breaking at **every** comma is wrong and must not be the behaviour. An
  ordinary in-sentence list (`apples, pears, plums and a small quantity of
  regret`) becomes one line per item, which is a worse diff than the hard wrap
  it replaced, and reads as damage in review. This is stated as a requirement
  because it is the obvious first implementation and it is a regression.
- The algorithm is **greedy clause packing**. Split the sentence into clause
  units at top-level boundaries; then emit lines by appending units while the
  line stays within the target width, starting a new line when the next unit
  would exceed it. A sentence that fits stays on one line and is never split.
- A clause boundary is a `,`, `;`, `:`, or a spaced em/en dash that sits at
  bracket and quote depth zero and outside every masked span. Boundaries inside
  parentheses, brackets, quotes, inline code, or a link's text or target are
  not boundaries.
- `--width N` (default 100, from config) is a **soft target**, not a hard
  limit: a clause unit longer than the width is emitted on its own line rather
  than broken mid-clause. Breaking mid-clause to satisfy a column count would
  reintroduce exactly the hard-wrap diff behaviour this feature exists to
  remove.
- `--min-line N` (default 32) suppresses a break that would leave a fragment
  shorter than `N` characters, so a trailing `, too.` does not become its own
  line.

**Sentence boundaries**

- Detection is rule-based and must not split on a period that ends an
  abbreviation. An extensible, project-editable abbreviation list is required —
  `e.g.`, `i.e.`, `cf.`, `etc.`, `vs.`, `Dr.`, `Mr.`, `Fig.`, `No.`, `Ch.`,
  `approx.` and so on ship as a default set, and the project's list is merged
  into it, not substituted for it.
- Initials are handled: a single uppercase letter followed by `.` is not a
  sentence end (`W. Somerset Maugham`). Because that rule misfires on a real
  one-letter word at the end of a sentence, the project may declare
  `single-letter-words` — letters that *are* words in this corpus and do end a
  sentence. Grade names, variable names and pronouns in translated text all hit
  this.
- Also not sentence ends: a period inside a number or version (`3.14`,
  `v1.2.3`), inside a masked span, or in an ellipsis. A sentence end is
  recognised for `.`, `!`, `?` and their combinations, followed by whitespace
  or end of block, with a following character that is not lowercase.

**Safety properties** — these are requirements on the implementation, and each
is a test:

1. **Idempotence.** `reflow(reflow(x)) == reflow(x)` for every input.
2. **Rendered-output equivalence.** Reflowing must not change what the document
   means. Verified by parsing before and after with
   [`parse_markdown`](S30-markdown-ir-conversion.md), running the
   [normalising pipeline](S31-ir-normalization-and-whitespace.md) over both,
   and requiring the two documents to compare equal modulo node ids. A file
   whose reflow fails this check is **not written**, and the failure is
   reported as an `error` naming the file. This is cheap, it is the property
   that actually matters, and it turns "the reflow ate my nested list" from a
   bug report into a build failure.
3. **Byte preservation outside prose.** Every non-`prose` byte in the output is
   identical to the input, including the final newline, the file's line-ending
   convention, and trailing frontmatter whitespace.

**Adoption**

- Turning `reflow` on in an existing repository produces **one enormous diff**:
  effectively every prose file changes. This is unavoidable and must be stated
  in `--help`, in the guide, and here.
- Therefore `reflow` is **opt-in per project** and is not in the default check
  set. A project enables it in `configs/prose.yaml`.
- The recommended adoption path is one commit that does nothing but reflow,
  recorded in the repository's `.git-blame-ignore-revs` so `git blame` skips
  it. The guide documents this; `mdd` does not write that file.

### `mdd prose lint` — mechanical slips

Small mistakes a spell checker cannot see, each with a stable rule id, each
`error` by default unless noted:

| Rule id | Finds |
|---|---|
| `multiple-spaces` | two or more spaces between words on a prose line |
| `space-before-punctuation` | whitespace before `,` `;` `:` `.` `!` `?` |
| `blank-run` | more than one consecutive blank line |
| `trailing-whitespace` | trailing spaces or tabs at end of line |
| `punctuation-outside-quote` | a `,` or `.` placed after a closing quote where house style puts it inside — **`off` by default** |
| `invisible-space` | a non-breaking or other invisible space character where a plain space was meant — **`warning`** |

- `trailing-whitespace` allows exactly **two** trailing spaces, which is
  Markdown's hard line break, unless `allow-hard-break: false` is configured.
  Three or more, or a tab, is always a finding. A project that has adopted
  `reflow` will find hard breaks near-useless and may want the strict setting.
- `punctuation-outside-quote` is off by default because it encodes a house
  style — the American convention — that not every corpus follows.
  `quote-punctuation: inside | outside | off` selects the convention;
  `outside` inverts the check.
- **The literal escape.** Moving a mark inside a quoted literal corrupts the
  literal: `the flag is "--dry-run", not "--check"` must not become
  `"--dry-run,"`. Three layers, applied in order:
  1. Anything already masked as `inline-code` is never considered. Backticks
     are the right way to write a literal and the check should push authors
     toward them.
  2. A heuristic skip when the quoted span looks like a literal rather than
     like prose: it contains no whitespace, or it begins with `-`, `--`, `/`,
     `$`, or `<`, or it matches a configured `literal-quote-patterns` regex
     list. Deliberately conservative — a false skip loses a finding, a false
     flag corrupts text.
  3. The per-line suppression comment, for whatever the first two miss.
- No rule in this subcommand has an autofix in v1. They are one-character
  changes, and `reflow` is already the risky writer; see
  [Open questions](#open-questions).

### `mdd prose anchors` — internal cross-references resolve

- Every internal link of the form `path/file.md#anchor`, `#anchor`
  (same-file), or `path/file.md` must resolve: the target file must exist, and
  where an anchor is given, a heading in that file must slug to it.
- External links (`http:`, `https:`, `mailto:` and any other scheme) are not
  checked. Link-rot checking needs the network and would break the
  determinism rule.
- **Slugging rule**, stated so it can be relied on and so a project can tell
  whether its renderer agrees: lowercase the heading text; strip Markdown
  inline markup, leaving its text; remove every character that is not a letter,
  digit, space, hyphen or underscore; replace runs of whitespace with a single
  hyphen. A heading whose slug collides with an earlier one in the same file
  gets `-1`, `-2`, … appended in document order. This is GitHub's rule, which
  is where the corpora `mdd` mirrors are read.
- `{#explicit-id}` at the end of a heading line sets the anchor directly,
  overrides the slug, and takes part in collision numbering.
- `--slug-style` selects the rule if a corpus is published by a renderer that
  slugs differently; `github` is the only style in v1, and the flag exists so
  adding one later is not a breaking change.
- An unresolved anchor reports the nearest existing anchor in the target file
  by edit distance, as a `did you mean` hint — the affordance
  `scripts/check-mdd-commands.py` already has, and the thing that makes the
  check pleasant rather than tedious.
- A duplicate heading text is reported under `ambiguous-anchor` (default
  `warning`): the link still resolves, but a later edit reordering the two
  headings silently retargets it.
- This check is what makes it *safe* to split, merge or renumber files in a
  large corpus — the operation that is most obviously correct to attempt and
  most reliably breaks something invisible.

### `mdd prose vocabulary` — project-owned house vocabulary

- A project supplies a list of banned phrases; a match is a finding. `mdd`
  ships **no** phrase list. House vocabulary is a house matter, and a list
  shipped in a tool becomes an argument in every consuming project.
- Each entry carries the phrase, an optional `message` explaining what to
  write instead, and an optional `severity`. Matching is case-insensitive and
  word-boundary-anchored by default. A `regex: true` entry matches as a regular
  expression; the flag is explicit so a stray `.` in a phrase never quietly
  becomes a wildcard.
- An **accept-list** of words the project has decided are correct lives
  alongside it, in a plain one-word-per-line file at a stable configured path.
  `mdd` does not spell-check — that is a solved problem with better tools — but
  it owns the accept-list's location so the same file can be handed to
  `codespell`, Vale, or an editor. `mdd prose vocabulary` consults it only to
  avoid flagging an accepted word that a `regex` entry would otherwise catch.
- `--list` prints the resolved, merged rule set and exits `0`, so a user can
  see what is actually in force.

### `mdd prose freshness` — content that goes stale

- For corpora that document something fast-moving, verify that each file
  carries a review date in its frontmatter and that the date is recent enough.
- The field name is configured (default `last-verified`) and the value must
  parse as an ISO-8601 date. Frontmatter is read through the typed layer of
  [S40](S40-typed-frontmatter.md); a malformed value is a finding, not a crash.
- `--max-age DAYS` (default from config) sets the threshold. `--as-of DATE`
  overrides "today" so the check is reproducible in a test and in a
  rebuilt-from-an-old-commit CI job. Without it the check is a pure function of
  the corpus *and the clock*, which is as deterministic as a freshness check
  can be, and the flag is how a test pins the clock.
- Three rules: `stale` (older than the threshold), `missing-review-date` (field
  absent), `malformed-review-date`. Defaults are `warning`, `off`, `error`
  respectively — reporting by default, gating when the project asks. A corpus
  mirrored from Confluence has no `last-verified` field at all, so defaulting
  `missing-review-date` to `error` would make the check useless out of the box.
- `--only-missing` and `--only-stale` narrow the run, for the "add the field
  everywhere first" adoption step.

### Configuration

- One YAML file, `prose.yaml`, with a single top-level `prose:` section,
  discovered exactly as every other per-domain config
  ([S07](S07-data-protection.md), and the configuration guide): explicit
  `--config PATH` (error if missing), then `./configs/prose.yaml`, then
  `~/.config/mdd/prose.yaml`. First hit wins; there is no merging across
  locations. Merging is the blacklist's behaviour and it exists there because
  the blacklist is a *safety* list that must not be shrinkable by shadowing. A
  style config is the opposite: a repository must be able to state its whole
  house style in one file a reviewer can read.
- No config file at all is not an error. The default set runs `lint` and
  `anchors` at their default severities. `reflow`, `vocabulary` and `freshness`
  do nothing until configured, because each of them either produces a
  corpus-wide diff or needs project-supplied data to mean anything.
- The file is committed to the corpus repository, next to the content it
  governs, and contains no secrets.

```yaml
# configs/prose.yaml
prose:
  checks:
    reflow: error          # off by default; opting in is a corpus-wide diff
    lint: error
    anchors: error
    vocabulary: error
    freshness: warning

  reflow:
    width: 100
    min-line: 32
    abbreviations: [approx., Prof., Sect.]   # merged into the built-in set
    single-letter-words: [I, A]              # letters that really do end a sentence

  lint:
    quote-punctuation: inside                # inside | outside | off
    allow-hard-break: false
    literal-quote-patterns:
      - '^-{1,2}[a-z]'                       # CLI flags
    rules:
      invisible-space: error                 # override a default severity

  anchors:
    slug-style: github

  vocabulary:
    accept-list: .prose-accept.txt
    banned:
      - phrase: "leverage"
        message: "use 'use'"
      - phrase: "click here"
        message: "link the destination's name instead"
      - phrase: '\bsimply\b'
        regex: true
        severity: warning

  freshness:
    field: last-verified
    max-age-days: 180
    rules:
      missing-review-date: warning
```

### Data protection and destructive behaviour

- **No network, no credentials, no model.** Nothing under `mdd prose` reads a
  token, resolves an `op://` reference, or opens a socket. There is nothing for
  [S07](S07-data-protection.md)'s credential rules to apply to.
- **The confidentiality blacklist is not consulted**, for the same reason
  [`mdd search`](S19-search-command.md) does not consult it by default: content
  the user already has on disk is not leaving its source system by being read
  locally. The blacklist gates content *going out*, and no `mdd prose`
  subcommand sends anything anywhere. If a future subcommand ever did, that
  changes.
- **`--write` mutates the user's files, so it gets the care every other
  mutating command in `mdd` gets:**
  - `--write` is opt-in per invocation, never sticky, and never inferred from
    the config file. A config may enable the *check*; only the command line can
    authorise the *write*.
  - Writes are atomic — temp file, `fsync`, rename — reusing the existing
    frontmatter-writer shape rather than a second implementation.
  - A file is written only if it parsed cleanly, its masking was balanced, and
    it passed the rendered-output-equivalence check above. Fail closed: any
    doubt means the file is left exactly as it was and an `error` is reported.
  - Only files under a `PATH` the user named are ever opened for writing, and
    `.mddignore` is honoured on the write path exactly as on the read path.
  - Each modified file is logged at `info`, one line per file, so a destructive
    run leaves an audit trail in scrollback and not only a count.
  - `--write` combined with `--json` is an argparse error. A caller streaming
    machine output while the tree changes underneath it is parsing a moving
    target, and the two modes serve different jobs.
- **Excerpts leak content into logs.** A finding's `excerpt` field, and the
  human-format message, quote a fragment of the source line. On a corpus
  mirrored from a private Confluence space that fragment ends up in a CI log,
  which is a weaker boundary than the repository. Excerpts are therefore
  trimmed to the flagged span plus a small context window, and `--no-excerpt`
  omits them entirely for jobs whose logs are broadly readable.
- No subcommand deletes a file, ever. The only mutation in this spec is
  in-place rewriting of prose lines by `reflow --write`.

## Design Approach

**Parse once, classify once, check many.** Every subcommand is a consumer of
one shared per-file artefact: the text, plus the line and span classification.
`mdd prose check` therefore costs barely more than the most expensive single
check, and — the real point — the checks cannot disagree about what prose is.

**A line-oriented classifier, not the IR, for the checks themselves.** The
[Markdown IR](S30-markdown-ir-conversion.md) is the right model for
*meaning* and the wrong one for *findings*: it deliberately abstracts away the
byte offsets a `file:line:column` report needs, and its normalising pipeline
discards exactly the whitespace `lint` exists to complain about. So the
classifier is its own small, byte-faithful scanner over the source text.

The IR is used for one thing, and it is the thing it is good at: **verifying
the reflow did not change the document.** Parse before, parse after, normalise
both, compare. That reuses [S30](S30-markdown-ir-conversion.md) and
[S31](S31-ir-normalization-and-whitespace.md) for a correctness proof rather
than for plumbing, and it means the reflow inherits every flavour edge case the
IR corpus ([S32](S32-ir-test-corpus-expansion.md)) already covers instead of
rediscovering them one bug report at a time.

**Greedy packing, not comma splitting.** The naive rule — break at every
sentence-internal punctuation mark — is easy, and it turns prose into
fragments. Greedy packing to a soft width has the property that matters: a
short sentence is one line, a long sentence becomes a few coherent lines, and
the line a sentence occupies changes only when that sentence changes. The width
is a target rather than a limit because enforcing a hard column would break
mid-clause and bring back per-line diff churn, which is the disease.

**Fail closed everywhere.** An unbalanced fence, a mis-parsed comment, a
reflow that fails equivalence: every one of them stops the file rather than
guessing. A prose tool that occasionally corrupts a file is worse than no prose
tool, because the corruption is silent and lands in a commit.

**Severity as data; printing and gating as separate decisions.** Learned from
`mdd`'s own two-pass Vale invocation. A finding has a severity; the command
line decides what gates. One pass, no contortion.

**Nothing shipped that is a house-style opinion.** The banned-phrase list is
empty by default. `punctuation-outside-quote` is off. `reflow` is off. `mdd`
ships mechanism; each corpus supplies its policy. The exceptions are the
handful of rules where there is no plausible second opinion — nobody wants two
spaces before a semicolon.

**Path-based scoping.** A gate that walks a configured mirror it was not
pointed at is a gate that fails for reasons unrelated to the change under
review. `PATH` positionals and `.mddignore`; nothing clever.

## Implementation Notes

### Module layout

Following the established shape — a thin CLI adapter under `commands/`, the
engine in a sibling package, per [S36](S36-module-structure.md)'s file-size
targets:

```
src/mdd/commands/prose.py     # register(), typed Namespace subclasses, handlers
src/mdd/prose/
    config.py                 # prose.yaml discovery + resolved frozen config
    classify.py               # the shared line/span classifier
    report.py                 # Finding, severity resolution, human + JSON emit
    reflow/
        sentences.py          # sentence segmentation, abbreviations, initials
        clauses.py            # clause-boundary detection, greedy packing
        apply.py              # per-block rewriting, IR equivalence check, write
    lint.py
    anchors.py                # slugging, link extraction, resolution, suggestions
    vocabulary.py
    freshness.py
```

`register(subparsers, parents)` adds the `prose` group and its subcommands, and
`prose.py` joins the registered-module tuple in `src/mdd/cli.py`. Each handler
gets its own `argparse.Namespace` subclass and a single `cast` at the top, per
[S35](S35-argparse-cli-parsing.md). None of the subcommands takes the shared
`--dry-run` parent — see the `--write` note above — and the shared `--config`
parent is used as-is.

### The finding record

One frozen dataclass, produced by every check and consumed by one emitter, so
the human and JSON formats cannot drift apart:

```python
@dataclass(frozen=True)
class Finding:
    path: Path
    line: int              # 1-based; 0 for whole-file
    column: int            # 1-based; 0 for whole-line
    check: str             # "reflow" | "lint" | "anchors" | …
    rule: str              # stable id, e.g. "space-before-punctuation"
    message: str
    severity: Severity     # error | warning
    excerpt: str | None
    fixable: bool
```

Rule ids are part of the public interface: they appear in output, in
suppression comments, and in config. Renaming one is a breaking change and
needs an alias.

### Classifier sketch

A single left-to-right scan producing, per line, a class and a list of masked
spans:

```python
@dataclass(frozen=True)
class Classified:
    lines: tuple[LineClass, ...]
    masks: tuple[tuple[Span, ...], ...]   # per line, sorted, non-overlapping

def classify(text: str) -> Classified | ClassifyError: ...
```

`ClassifyError` carries the line and reason, and is turned into a `parse-error`
finding; it is a value rather than an exception because "this one file is
unparseable" must not end a corpus-wide run.

Block-level state (fence depth, HTML comment, frontmatter, table, block quote)
is a small explicit state machine. Inline masking (code spans, link targets,
footnote references, autolinks, raw inline HTML, math) runs per prose line with
backtick-run counting, so a code span that itself contains a backtick masks
correctly. This is the fiddliest code in the feature and should be the most
heavily tested; the flavour configuration in `src/mdd/markdown/ir/flavour.py`
is the reference for which constructs are in scope.

### Reflow pipeline

Per prose block: unwrap to a single logical string, recording the prefix that
must be re-applied to each output line (list indent, `> `, or nothing); segment
into sentences; split each sentence into clause units; greedily pack units into
lines; re-apply the prefix. Masked spans travel as opaque tokens through the
whole pipeline and are substituted back at the end, which is what makes "never
break inside a link" structural rather than a rule that can be forgotten.

Then the equivalence gate: parse the before and after text, normalise both,
compare with node ids stripped. Comparing whole documents rather than per-block
keeps the check simple and catches the interesting failures, which are the ones
where a rewrite changes block *structure*.

### Composition with `mdd ai`

Nothing in `src/mdd/prose/` imports `src/mdd/ai/`, and nothing in
`src/mdd/ai/` imports `src/mdd/prose/`. The composition is at the shell:

```bash
mdd prose check && mdd ai review --all
```

`mdd ai rewrite` produces prose that will not be reflowed; the recommended
order after an AI edit is `mdd ai rewrite --apply`, then `mdd prose reflow
--write`, documented in the guide. Wiring reflow into `rewrite` automatically is
rejected: it would make a deterministic transformation a side effect of a
non-deterministic one, and a user who wanted only the first could not get it.

### Relationship to this repository's own prose tooling

`mise run docs-vale` and `scripts/spec-check.py` stay as they are. Vale is a Go
binary with a downloaded style package and a network sync step — fine as a
development dependency of `mdd` itself, wrong as something `mdd` imposes on
every corpus it is pointed at. The overlap is small in practice: Vale does
spelling, term casing and readability advice; `mdd prose` does mechanics,
structure and layout. Once `mdd prose` exists, `mdd`'s own `docs-check` task
should run it over `docs/` as well — dogfooding, and the cheapest possible test
that the checks are tolerable to live with.

### Documentation and the CLI-string gate

`mise run check-mdd-commands` validates every `mdd …` string in the top-level
docs and in `docs/guide`, `docs/articles` and `docs/reference` against the real
argparse tree, and it is part of `mise run ci`. So the guide section for
`mdd prose` lands **with the implementation**, not before it — a commands-guide
entry for a command that does not exist yet turns CI red. `docs/spec/` is
exempt from the gating run, which is why this spec may name the commands
freely.

A bundled agent skill ([S23](S23-skills-bundle.md)) for `mdd prose` is worth
having — the adoption sequence for `reflow` is exactly the kind of multi-step
procedure a skill is for — but it follows the commands rather than shipping
with them.

### Testing

- Unit tests per check under `tests/prose/`, CLI-level tests driving the real
  dispatcher under `tests/commands/test_prose.py`, matching the existing
  layout. Coverage floor is the repository's 85%; aim above 90% for this code.
- A golden corpus under `tests/corpus/prose/` of input and expected-output
  pairs, one directory per scenario, covering at minimum: every masked
  construct, a sentence with a list of commas that must **not** be split, an
  abbreviation at a line end, an initial, a version number, a nested list, a
  block quote, a table adjacent to prose, a multi-line HTML comment, CRLF line
  endings, and a file with no trailing newline.
- Property tests: idempotence and IR equivalence over the golden corpus and
  over the existing IR corpus ([S32](S32-ir-test-corpus-expansion.md)), which
  is already the most hostile pile of Markdown in the repository and is exactly
  the right adversary for a reflow.
- A test that every rule id appearing in the default config resolves to a real
  rule, and vice versa. Declared-but-nonexistent rule ids are the thing that
  rots first in a configurable linter.
- No test in this feature may touch the network or a model; `mdd prose` needs
  no `integration` marker because there is nothing to integrate with.

## Related upstream specs

- [000-specs](000-specs.md) — shared conventions.
- [S07](S07-data-protection.md) — credential and blacklist rules; this spec
  records why neither applies and what `--write` inherits instead.
- [S19](S19-search-command.md) — the `--json` line-delimited output convention,
  and the root-source registry this spec deliberately does not use.
- [S21](S21-ai-rewrite-and-index.md) — `mdd ai rewrite`, whose protected-region
  masking is the closest prior art in the tree, and which sits on the other
  side of the deterministic/judgement line.
- [S22](S22-ai-review-command.md) — `mdd ai review`; a report for a human, not
  a gate. `mdd prose` is the gate.
- [S30](S30-markdown-ir-conversion.md) — markdown ↔ IR conversion, used for the
  reflow equivalence proof.
- [S31](S31-ir-normalization-and-whitespace.md) — the normalising pipeline the
  equivalence comparison runs through.
- [S32](S32-ir-test-corpus-expansion.md) — the adversarial Markdown corpus the
  reflow property tests run against.
- [S35](S35-argparse-cli-parsing.md) — command-group registration, the
  two-level limit, and the typed-Namespace-plus-`cast` convention.
- [S36](S36-module-structure.md) — file-size shape for the new sub-package.
- [S39](S39-mddignore.md) — `.mddignore`, honoured on both the read and the
  write path.
- [S40](S40-typed-frontmatter.md) — typed frontmatter, used by the freshness
  check.

## Open questions

1. Should `mdd prose lint` grow a `--write` autofix for the whitespace rules?
   They are unambiguous and trivially reversible, so the argument for is
   strong; the argument against is that a second writer doubles the surface
   that can corrupt a file. Deferred until `reflow --write` has been lived with.
2. Is a single `--width` enough, or does a corpus want a different target for
   list items and block quotes, whose prefixes eat into the line? Probably yes
   in the long run; one knob until someone complains.
3. Should `mdd prose check` be able to restrict itself to files changed against
   a git ref (`--since main`)? Very attractive for adoption — it is the only
   way to gate a corpus that cannot be fixed in one pass — but it makes the
   command depend on git state, which is a real complication. Likely a v2.
4. Does the anchor check need to understand a documentation generator's own
   routing (a static-site builder mapping `docs/foo.md` to `/foo/`), or is
   file-relative resolution enough? File-relative is right for a Markdown
   corpus in git; generator-aware resolution is a different tool. Revisit if
   [the documentation site](S06-documentation-site.md) wants it.
5. Should the abbreviation and single-letter-word lists be shareable across
   projects — a small bundled set per language, selected by config — rather
   than each corpus curating its own? Only worth it once there are several
   corpora to compare.

## Out of scope

The exclusions below are deliberate and are the boundary of this spec.
`mdd prose` covers what is **generic to any Markdown corpus**. Everything
specific to a particular kind of book, course or site belongs in that project's
own tooling, where it can change at the project's pace.

- **Anything specific to executable-code books.** In particular:
  - Extracting fenced code blocks into runnable source files.
  - Compiling or running those files and pinning their real output back into
    the listing.
  - Exercise-and-solution coverage: checking that every exercise has a
    solution, that every solution is referenced, and that numbering is
    contiguous.

  These need a language toolchain, a build directory, and a project-specific
  convention for how a listing names its file. `mdd` would be guessing at all
  three. A project builds them on top of `mdd prose`, not inside it.
- **Spell checking.** Solved elsewhere and better. `mdd` owns the accept-list's
  location so another tool can read it; that is all.
- **Style and readability advice** — passive voice, sentence length, reading
  grade, clichés. Judgement, not mechanics. Vale does it for `mdd`'s own docs,
  and [`mdd ai review`](S22-ai-review-command.md) is where a model's opinion
  belongs.
- **Grammar checking.** Same reason, one step further.
- **External link checking.** Needs the network, so it cannot be deterministic
  and cannot live in this group.
- **Rewriting anything but prose lines.** No reformatting of tables, no
  normalising of list markers, no fence-style unification, no frontmatter key
  ordering. A Markdown formatter is a different tool with a different blast
  radius, and combining the two would make the reflow's one-enormous-diff
  adoption cost even larger.
- **A `mdd prose fix` that applies every fixable rule at once.** Deliberately
  absent while there is exactly one writer.
- **Enforcing a house style out of the box.** No bundled banned-phrase list, no
  default quote convention, no shipped vocabulary. See
  [Configuration](#configuration).
- **Cross-corpus checks.** Every subcommand runs over the paths it was given.
  Resolving an anchor into a *different* mirror is
  [`mdd search`](S19-search-command.md)'s territory and a much harder problem.
- **Incremental / changed-files-only operation.** See
  [Open questions](#open-questions), item 3.
