# Retrieval: deployment status, and the full benchmark record

The headline benchmark numbers are in the
[README](../README.md#retrieval-benchmark). This is the working record behind them —
what dense retrieval bought, why reranking is off, and every measurement that shaped the
current configuration, including the ones that were unflattering.

[← back to the README](../README.md)

---

## Deployment status

The live app runs dense retrieval, sparse full-text search, article-title
search and exact-article lookup, fused by RRF. **Cross-encoder reranking is
implemented but switched off** — see below.

For most of this project's life dense retrieval was silently dead.
`sentence-transformers` was never listed in `requirements.txt`, so every chunk
carried a real `bge-m3` vector (1024-dim, unit-norm, 11,042 distinct across
11,538 chunks) while the *query* side could not embed at all. The dense branch
raised on every request, the retriever fused three branches instead of four,
and the reranker never loaded. Nothing reported it: `/health` showed
`embedded_chunks: 11538` and a green status throughout, because that field
counts documents and cannot detect a broken query path.

Three further defects were hiding behind that one, each only visible once the
one in front of it was fixed:

1. **torch was pinned below 2.6.** `transformers` refuses `torch.load` on older
   versions (CVE-2025-32434) and bge-m3 ships `.bin` weights, so the model
   would have failed to load regardless of available memory.
2. **Baking the weights into the image made it undeployable.** A 4.8 GB image
   exceeds Fly's machine-update timeout; the API returns HTTP 408, `flyctl
   deploy` swallows it and reports success, and the machines keep running the
   previous image. Weights are now downloaded at runtime and the startup warmup
   runs in the background so the port binds immediately.
3. **The relevance threshold silently capped recall at three results.**
   `MIN_RELEVANCE_SCORE` is calibrated for the cross-encoder's sigmoid output,
   but the score falls back to the RRF fused value (~0.016–0.065), which can
   never clear a 0.25 threshold. With the reranker off, every candidate was
   discarded on every query and a three-result fallback took over.

### What dense retrieval bought

Cross-language retrieval, which the corpus makes essential:

| Script / language | Chunks | Share |
|---|---|---|
| Uzbek Cyrillic | 5,904 | 51% |
| Russian | 4,927 | 43% |
| Uzbek Latin | 707 | 6% |

Latin↔Cyrillic is bridged lexically by the transliteration layer. **Uzbek↔Russian
is bridged only by the shared embedding space** — nothing lexical connects
`odam oʻgʻirlash` to `похищение человека`.

Concretely, *"Odam oʻgʻirlash uchun qanday jazo belgilangan?"* previously
returned four results and never reached Criminal Code art. 137. It now returns
eleven and ranks art. 137 by embedding similarity (0.60). Benchmark-wide, MRR
went from 0.694 to 0.807 and Recall@1 from 0.600 to 0.733.

### Why reranking is off: the arithmetic

Reranking was pursued through three models and two architectures. It is off,
and the reason is not tuning.

**Model selection.** `bge-reranker-base` (278M) is small enough but trained on
Chinese and English only — against Russian its scores clustered at 0.5000,
0.5027, 0.5042, and sigmoid(0) is exactly 0.5, so it was emitting near-zero
logits and reordering by noise. Recall@1 fell 0.793 → 0.569.
`jina-reranker-v2-base-multilingual` (278M) is the right model: genuinely
multilingual, and it scores «Понятие сделок» 0.32 against 0.13 for an unrelated
article.

**Architecture.** Run in-process the reranker contends with the query embedder
for the same cores behind one uvicorn worker — 40s per query, then nothing
completing. So it now runs as [its own service](../reranker-service/), reachable
only at `uzlex-reranker.internal`, and that part works: the service loads, the
scores discriminate, and the client degrades to fused order when it cannot
answer.

**It still does not help, and the reason is arithmetic.** A transformer forward
pass costs roughly `2 × params × tokens`. For 12 candidates at 96 tokens
against a 278M model that is about 640 GFLOP. Two dedicated CPU cores deliver
on the order of 50 GFLOPS, so ~13 seconds — which is exactly what was measured:

| Passages × tokens | shared-cpu-4x | performance-2x |
|---|---|---|
| 12 × 96 | — | 13.0 s |
| 12 × 320 | 272 s | 26.0 s |
| 8 × 320 | 75 s | — |
| 4 × 320 | 8.8 s | 19.7 s (cold) |

The superlinear blow-up on shared CPU is throttling; the dedicated-CPU numbers
are the real cost. Reaching a ~1s rerank needs roughly 15× more compute. That
is a GPU, where the same work is milliseconds — not a smaller model, a shorter
sequence, or a different host size.

Retrieval currently answers in ~2.1s with Recall@5 = 0.931 and Recall@10 =
0.983 without reranking at all. Paying 13s for a possible reordering of results
that are already correct 93% of the time is not a trade worth making.

**The service is kept, scaled to zero.** `flyctl scale count 1 -a
uzlex-reranker` brings it back; on a GPU machine it becomes immediately
worthwhile, and `RERANKER_BACKEND=remote` on the main app is all that is needed
to use it. Two details in it are worth not rediscovering: bind `::` rather than
`0.0.0.0`, because Fly's private network is IPv6-only and a service on the IPv4
wildcard resolves and then refuses every connection; and Fly allocates public
IPs on app creation, which must be released for a service that carries no auth.

### Current production configuration

| Setting | Value | Why |
|---|---|---|
| VM | `shared-cpu-4x`, 8 GB | dense retrieval embeds one short query per request; bge-m3 is ~2.3 GB resident |
| `DENSE_RETRIEVAL_ENABLED` | `true` | |
| `RERANKER_ENABLED` | `false` | too slow even on dedicated CPU (above) |
| `PREFETCH_MODELS` | `false` | baking weights in makes the image undeployable |
| `UVICORN_WORKERS` | `1` | each worker would load its own copy of the model |

`RERANKER_ENABLED` and the rate limits are **Fly secrets**, and secrets silently
shadow `fly.toml [env]`. Check `flyctl secrets list` before trusting any value
in the committed config.

### Optional providers, and what each needs

Sign-in and billing are built and tested, but every one of them needs an
account created under the operator's own name. Until then each stays dormant
by design: the API returns `503`, the UI omits the control, and no third-party
script is loaded.

| Feature | Secrets (backend) | Public (frontend) | Also required |
|---|---|---|---|
| Telegram login | `TELEGRAM_BOT_TOKEN` | `NEXT_PUBLIC_TELEGRAM_BOT` | @BotFather bot + `/setdomain` |
| Google login | `GOOGLE_CLIENT_ID` | `NEXT_PUBLIC_GOOGLE_CLIENT_ID` *(same value)* | OAuth client + authorised origin |
| Phone login | `ESKIZ_EMAIL`, `ESKIZ_PASSWORD`, `ESKIZ_SENDER` | `NEXT_PUBLIC_PHONE_LOGIN=true` | Eskiz.uz account |
| Payme (Pro) | `PAYME_KEY`, `PAYME_MERCHANT_ID`, `PAYME_ENABLED=true` | `NEXT_PUBLIC_PAYME_MERCHANT_ID` | merchant cabinet + sandbox checklist |

Set them with `flyctl secrets set` — never in `fly.toml`, which is committed.

### Vocabulary: asking in ordinary words

The statute says *xodim*; a person describing their own situation says
*ishchi*. Both mean "employee", nothing lexical connects them, and the
multilingual embedding did not bridge them either — the dense branch scored the
gold article 0.0 on both phrasings. Labour Code art. 160 ranked 1st when asked
with the statute's word and did not appear in the top 20 when asked with the
ordinary one.

A small synonym map ([`synonyms.py`](../backend/app/services/rag/synonyms.py))
now expands query terms for the **lexical branches only** — the dense branch
embeds the question as asked, since padding that text with synonyms moves the
query vector away from what the user wrote. It is deliberately conservative:
*shartnoma* (contract) and *bitim* (transaction) are not grouped, because in a
tool that claims to cite the governing provision, conflating terms a lawyer
distinguishes is worse than a miss. Tests pin those non-equivalences down.

Three things surfaced while building it, each of which would have been invisible
without checking against the database:

1. **A space inside a tsquery term is a syntax error.** Multi-word synonyms were
   emitted as `иш берувчи:*`, `to_tsquery` raised, and the keyword search turned
   that into an empty result through its except-and-return-`[]` handler — so
   adding synonyms silently disabled the sparse and title branches for exactly
   the queries they were meant to help. They are adjacency phrases now.
2. **The reflexive pronoun was signal, not scaffolding.** Stripping *o'zi* as a
   framing word looked obviously right and destroyed the distinction between
   art. 160 (employee's own initiative) and art. 166 (employer's).
3. **Postgres full-text ranking has no corpus statistics.** *ходим* appears in
   hundreds of Labour Code titles and *ходимнинг ташаббуси* in exactly the one
   article about resigning, but `ts_rank_cd` weights them identically, and
   length normalisation then favours the shorter, vaguer title — the governing
   article scored 0.011 against a competitor's 0.033. Article titles containing
   the whole phrase are now ranked ahead of titles sharing a single word.

Art. 160 moved from absent to rank 5 on the ordinary-vocabulary phrasing.
Aggregate movement was small — Recall@5 0.860 → 0.877, MRR 0.791 → 0.795 — which
is within the run-to-run variance noted above, so the targeted fix is verified
directly rather than inferred from the totals.

### Crossing between Uzbek and Russian

43% of this corpus is Russian-only, and nothing lexical connects Russian to
Uzbek — so a question asked in Uzbek could not reach those acts through the
keyword branches *at all*. Dense retrieval was the only bridge, and bge-m3's
Uzbek is the weakest part of its multilingual coverage. Measured: *"Битим деб
нима тушунилади?"* never reached Civil Code art. 101 «Понятие сделок», and
*"So'roq qayerda o'tkaziladi?"* never reached Criminal Procedure art. 96 «Место
допроса».

Legal terminology is a closed vocabulary, which makes a glossary a workable
bridge where a general bilingual dictionary would not be. Roughly thirty pairs
now connect the two languages, each a term of art with one settled counterpart.

The glossary must not become a back channel for merging terms the codes
distinguish, so `bitim`/`сделка` and `shartnoma`/`договор` remain separate
groups and a test asserts that in both directions.

| Item | Question | Gold act | Before | After |
|---|---|---|---|---|
| uz-171 | Uzbek Cyrillic | Civil Code (ru) | miss | **rank 2** |
| uz-172 | Uzbek Latin | Crim. Procedure (ru) | miss | **rank 4** |

Recall@5 went 0.877 → 0.912 and Recall@10 0.930 → 0.965, both clearing target
on the 57-question set. Median retrieval rose from 1327 ms to 1441 ms — the
expanded term list costs something, and it stays well inside budget.

### Transliteration: four defects that hid 4% of the corpus

94% of this corpus is Cyrillic, so the Latin↔Cyrillic layer decides whether a
Latin query reaches the text at all. Four defects lived in it, each silent in
the same way: the query succeeds, retrieval returns *something*, and only the
governing article is missing.

Measured over the full Uzbek heading vocabulary — 3,683 distinct words, 20,719
occurrences — the share unreachable from any Latin spelling was **4.05%**. It
is now **0.14%**.

| Defect | What it hid |
|---|---|
| **The glottal stop vanished.** `normalize_apostrophes` folds every apostrophe glyph into U+2018 before the mapping table runs; the table only had rules for U+02BC and U+0027, written for raw input before normalisation was put in front of them. | `ta'til` → `та‘тил` → `татил`, which cannot prefix-match the corpus form **`таътил`**. Also hit the glossary's own `da'vo`/`даъво` and `mas'uliyat`/`масъулият`. |
| **Word-initial `e` is `э`, not `е`.** Uzbek Cyrillic reserves `е` for /ye/, which Latin spells `ye` and the table already handled. | 98 distinct words, **532 occurrences**, including `этиш` at 124 — about as common as an Uzbek verb gets. |
| **`o‘` and `g‘` are single letters** and must be matched before the y-digraphs. Matching `yo` first split `yo‘l` down the middle. | `yo‘l` → **`ёъл`**, a string that appears nowhere. Road, lost, passenger: none reachable. |
| **Latin `ts` is `ц`** in the Russian loanwords that fill legal text. | 230 occurrences — `лицензия`, `декларация`, `процессуал`. |

The last one cannot be a table rule, because `ts` also arises where a native
`t` meets a native `s`: `ko‘rsatsa` is `кўрсатса`, not `кўрсаца`. Both readings
are offered as separate candidates in the OR-query instead, and the wrong one
matches nothing.

The safety property worth stating: comparing the reachable set before and
after, **173 words were gained and none were lost** — the change is strictly
additive at this layer. The index is built from the raw columns, so this is
query-side only and needed no re-indexing.

### Auditing the benchmark itself

A benchmark can be wrong in ways that look exactly like the system being wrong.
Auditing all 57 scored items against the corpus found four whose gold article
shared its title with another article in the same act — retrieval was being
marked incorrect for returning an equally correct provision:

| Item | Problem | Fix |
|---|---|---|
| uz-004 | Criminal Code 73 and 89 are **both** «Условно-досрочное освобождение от отбывания наказания» | accept both |
| uz-131 | Civil Procedure 128 and 174 are **both** «Давлат божи» | accept both |
| uz-104 | targeted «Солиқ тўловчилар» — a title the Tax Code carries **14 times**, once per tax type | replaced |
| uz-161 | targeted «Солиқни тўлаш тартиби», repeated the same way | replaced |

The distinction matters. Where the corpus genuinely carries one provision under
two numbers, accepting both is correct and `gold_article` now takes a list.
Where the question was simply too generic to have a single answer, the
*question* was the defect and no scoring rule could rescue it.

[`audit_gold.py`](../backend/benchmarks/audit_gold.py) runs this check against the
live corpus and exits non-zero on a missing or ambiguous label, so it can gate
a change to the benchmark:

```bash
python backend/benchmarks/audit_gold.py --base https://uzlex-ai.fly.dev
```

It reads `/api/v1/laws/articles`, which also makes the benchmark's provenance
claims auditable by someone without database access.

Correcting the labels was close to score-neutral, which is the point: uz-004 and
uz-131 now pass because they were always right, while the two replacement
questions are genuinely harder than the ambiguous ones they displaced, and
Recall@10 slipped from 0.965 to 0.947. The numbers now measure retrieval rather
than the benchmark's own defects.

### Choosing between codes

`uz-007` asked *"Какая ответственность за нарушение правил пожарной
безопасности?"* and expected Criminal Code art. 259. The Code of Administrative
Responsibility carries an article with the **identical title**, art. 211, so the
question had no single correct answer and the gold label picked one arbitrarily.
The question now names the kind of liability, and `uz-007a` mirrors it for the
administrative side — testing one direction alone would not show whether the
system distinguishes the codes or merely prefers one of them.

Rewording exposed a real gap rather than closing the item. Asked specifically
about *уголовная* liability, the system still returned the administrative
article first: nothing in either article's text or title says which liability it
imposes, and the only thing separating them is the name of the code they sit in
— which retrieval never looked at.

`act_affinity` scores how strongly a question names the act a candidate comes
from. One-directionally, because act names are mostly dates and boilerplate no
question would repeat; and excluding the words common to every act name, without
which "ответственность" matches the administrative code's own title and drags
every liability question toward it. Both directions now rank first.

### Terms of art versus how people actually ask

The last cluster of misses had one shape: people describe the situation, the
statute names the doctrine. The Criminal Code defines *невменяемость* as being
unable to understand the significance of one's actions — which is exactly how a
non-lawyer phrases it — and the Civil Procedure Code says *мақбуллик* where
people say *қабул қилинади*. Adding those to the glossary took `uz-110` from a
miss to rank 3 and moved `uz-130` up.

Recall@10 reached 0.983 and Recall@5 0.931, so **both recall targets are met**.

A third candidate was deliberately dropped. It would have mapped the statute's
phrase *истисно этувчи* to the particular verb form one benchmark question
happened to use — tuning to a query rather than encoding a term of art, and the
line between the two is the whole difference between improving the system and
inflating its score.

### What was tried and rejected

Widening the retrieval pools (`RETRIEVAL_TOP_K_DENSE`/`SPARSE` from 40 to 150)
was the obvious general fix, since the missing articles all scored
`dense_score = 0.0` — they fell outside the pool entirely. Measured, it traded
badly:

| | Recall@1 | Recall@5 | Recall@10 | Median latency |
|---|---|---|---|---|
| 40 | 0.776 | **0.914** | **0.948** | **1459 ms** |
| 150 | **0.810** | 0.897 | 0.931 | 2009 ms |

It buys Recall@1 and costs Recall@5 and @10, because `RERANK_CANDIDATE_CAP`
truncates the fused list and a wider pool simply adds competitors that push the
gold out. It also crosses the 2000 ms budget. Kept at 40.

### Where it still fails

`uz-104` and `uz-160` remain outside the top five. Both are reachable only by
mapping a specific inflected verb form to a specific statutory phrase, which is
where useful generalisation stops and benchmark-fitting begins. They are left
failing on purpose; a working cross-encoder reranker is the honest fix, and that
remains blocked on latency (above).
