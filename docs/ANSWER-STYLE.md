# Answering the way the question was asked

Script mirroring, answer length, handling ordinary questions, and the interface that
surfaces all three.

[← back to the README](../README.md)

---

## Answering the way the question was asked

Three behaviours that sound like prompt tweaks and were not. Each needed a
mechanism, and each failed first in a way worth recording.

### Reply in the script the user typed

Uzbek is written in both Latin and Cyrillic, and the corpus is mostly
Cyrillic. Asked in Latin, the model kept answering in the script of the
*sources* it had just read — reasonable-looking behaviour that is wrong for a
reader who cannot comfortably read the other script.

Detection was never the problem; it was correct from the start. The instruction
was simply being ignored, and it started being obeyed only once the prompt
**named the letters** rather than the script — "ў, қ, ғ, ҳ" instead of
"Cyrillic".

<div align="center">
<img src="../assets/script-mirroring.gif" alt="The same question asked in Uzbek Latin and Uzbek Cyrillic, each answered in the script it was asked in, both cited to the same Cyrillic-only article" width="720"/>
</div>

*The same question in both scripts, against the live API. Note the source in
each: one Cyrillic article, 106-modda of the Labour Code, answered back in
whichever script the question used.*

### Answer at the length that was asked for

"juda qisqa" (very short) produced a **2,541-character** answer — longer than
the default. The instruction was in the cached system prefix, and a cached
prefix loses to immediate context and to the model's own previous turns. Moving
the restatement into the user turn, where recency actually carries, fixed it.

<div align="center">
<img src="../assets/verbosity.gif" alt="The same legal question asked plainly and then with a very-short directive; the second answer is a third the length and still carries citations" width="720"/>
</div>

Measured against the live API, *«Ishdan boʻshash tartibi qanday?»* asked twice:

| Same question | Answer | Sources |
|---|---|---|
| asked plainly | 1,750 chars | 10 |
| `+ juda qisqa` | **615 chars** | 9 |

The point of the second column is that shortening changes the *answer*, not the
grounding — the short reply still retrieves and still cites.

The directive is also *remembered*: ask for short answers once and following
turns stay short, and `_resolve_turn` walks back past runs of consecutive
directives so "shorter" then "even shorter" still resolves to the last real
question rather than to another directive.

### Answer ordinary questions like a person

Greetings, small talk and "what can you do" are answered deterministically,
without an LLM call. Beyond those, an everyday question gets a warm, ordinary
reply instead of a bureaucratic non-answer:

```
"Salom! Sen nima qila olasan?"  →  "Salom! Men Oʻzbekiston Respublikasining
                                    qonunlari boʻyicha savollarga javob beraman…"
```

This is the feature that created the grounding hole described in
[Grounding](GROUNDING.md) — worth reading as a pair, because the pleasant
behaviour and the dangerous one came from the same change.

## Interface

The frontend is a full product surface, not a debug console: a marketing
landing page, a streamed chat view with deep-linked citation cards and risk
badges, sign-in and pricing dialogs, and a document-analysis view.

- **Design tokens, not hard-coded colours.** One CSS-variable palette in
  `globals.css` with role names (`surface`, `border`, `accent`, `muted`), and
  Tailwind maps to those rather than to raw hex — so dark mode is a token swap
  under a `class` strategy, not a second set of utilities per element.
- **Contrast and touch targets are checked, not eyeballed.** Body text meets
  WCAG AA against both palettes; interactive controls are ≥44 px tall.
- **16 px inputs on mobile**, below which iOS Safari zooms the page on focus.
