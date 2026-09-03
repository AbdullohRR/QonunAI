# Grounding: how the citation guarantee broke, and what fixed it

The mechanics of the anti-hallucination guarantee are summarised in the
[README](../README.md#how-the-anti-hallucination-guarantee-actually-works). This is the
longer account: the hole that existed for most of the project, how it was found, and
what now closes it.

[← back to the README](../README.md)

---

### The way this guarantee actually got broken

Worth recording, because the mechanism was subtle and the lesson generalises.

Everything above protects answers that go *through* retrieval. It says nothing
about a question that never reaches it — and a routing change quietly created
exactly that path.

Deciding "not a legal question" used to happen *after* retrieval, where the
real work was done by a different rule: **a question that found sources is
legal by demonstration.** The scope check only sorted what was left over, so
it could afford to be a rough heuristic.

Moving that check in front of retrieval — a fix for a real problem, since a
deflection passes validation cleanly and gives nothing to hook — made it the
sole arbiter. Under that load its default was backwards: *absence of legal
vocabulary became proof of a non-legal question.*

Measured on ordinary phrasings, **4 of 11** genuine legal questions fell
through, because no word in them is an act name, an article number, or a
glossary term:

| Question | Why it slipped |
|---|---|
| «Ishdan boʻshash tartibi qanday?» | dismissal — no statutory term in the sentence |
| «Ишдан бўшаш тартиби қандай?» | same, Cyrillic |
| «Как разделить имущество при разводе?» | divorce, property division |
| «How many days of annual leave am I entitled to?» | entitlement phrased as a person would |

Each was answered from the model's own memory: confident specifics, **no
citation, no source panel, `retrieval_ms: 0`.** The one failure mode the
project exists to prevent, reached through the component meant to prevent it —
and invisible from outside, because an ungrounded answer looks exactly like a
grounded one until you check whether any source was consulted.

The prompt-level backstop did not save it either. `GENERAL_SYSTEM` already
says, in as many words, to treat a disguised legal question as legal and
refuse to answer from memory. The model ignored it — the same lesson as point
3 above, arriving a second time by a different door: **grounding holds because
of a mechanism, not because the prompt asks for it.**

The fix is not more legal keywords. Law has no closed vocabulary, so every
gap in such a list is silent and dangerous. The *non-legal* side is enumerable
in the only sense that matters: a miss there is harmless. So the conversational
path now requires **positive evidence of an everyday topic** — cooking,
weather, sport, translation — and the deliberate gap between "looks legal" and
"looks everyday" falls to the legal side. An unrecognised question about
restaurants gets a stiff "the retrieved provisions don't cover this", which is
a worse conversation and not a wrong answer.

Verified on the live instance after the fix — the same question that had
returned `retrieval_ms: 0`:

```
Ishdan bo'shash tartibi qanday?   → retrieval_ms 1692 · 9 sources · 3 citations
Osh qanday pishiriladi?           → retrieval_ms    0 · conversational, as intended
```
