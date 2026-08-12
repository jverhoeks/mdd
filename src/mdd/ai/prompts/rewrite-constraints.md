## Constraints

- Do NOT change the substance or meaning of any content.
- Do NOT add information that was not present in the original text.
- Do NOT remove important information.
- Do NOT alter headings unless their grammar or capitalisation is clearly wrong.
- Do NOT reword link text, code references, or proper nouns.

## Placeholder tokens

The input may contain tokens of the form `__MDD_PROTECTED_0__`, `__MDD_PROTECTED_1__`, and so on, each alone on its own line.

These are not noise, filler, or leftover markup. Each one stands in for a block — a table, a code fence, an export header, a footer — that was removed from the input before it reached you and will be pasted back in its place afterwards.

- Reproduce every token in your output, byte for byte, on its own line, in the same order and the same position relative to the surrounding text.
- Copy the digits exactly. `__MDD_PROTECTED_2__` is not interchangeable with `__MDD_PROTECTED_1__`.
- Do NOT merge, reorder, renumber, drop, or comment on any token, and do NOT wrap one in backticks, quotes, or emphasis.
- This holds even when a token is the first thing in the input, the only thing on a line, or looks out of place. If the document is almost entirely tokens, return almost entirely tokens.

Dropping a single token makes the whole rewrite unusable and it will be discarded.

## Output

Return only the rewritten markdown body. Do not include any preamble, explanation, or closing remarks. Do not wrap the output in code fences.

Rewrite the whole input. Do not stop partway, and do not summarise, truncate, or replace a section with a note that it continues.
