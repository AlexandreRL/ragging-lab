# A tour of the lab notebook

A section-by-section map of
[`retrieval_eval_and_vector_search_internals.ipynb`](../retrieval_eval_and_vector_search_internals.ipynb):
what each part proves, what to take away, and **which cell to steal** for your own project.

Reading time ~15 minutes. Running time ~90 seconds.

---

## How to use this notebook

**If you have 10 minutes:** read §4 (metrics), §6.3 (the paired bootstrap), and the §7 trade-off chart.
That is the core of both feedback topics.

**If you have an hour:** run it all, then do exercises 1–3 in §10.2.

**If you are preparing for an interview:** read §10.3 (the cheat sheet) last, after the measurements make it
mean something. Each line in it is a one-sentence answer anchored to a number above it.

**If you are starting a client project:** steal §4.1 (metrics), §4.3 (harness) and §6.3 (significance),
replace `DOCUMENTS`, and write 30 labelled queries. That is a working evaluation apparatus in an afternoon.

---

## § 1 — Setup

Three pieces of infrastructure, each worth copying:

- **`Config`** — a single dataclass holding every knob. Experiments mean changing one field, not hunting
  through cells.
- **`LLM`** — an Anthropic client that *probes its credentials with one ~10-token call at start-up* and
  classifies the failure (no credentials / rejected / model unavailable / network / rate limited) instead of
  raising halfway through your run. `text()` and `json()` return `None` when unavailable, so callers branch
  once at the boundary. Also tracks tokens and prints a cost estimate.
- **`Embedder`** — sentence-transformers, with automatic fallback to TF-IDF+SVD and then to a hashed random
  projection, plus a cache keyed by text. The cache is not a nicety: re-embedding the corpus on every
  experiment is the most common reason a RAG eval loop is too slow to iterate on.

**Steal:** the probe-and-degrade pattern. It is the difference between a demo and something you can put in
front of a client on a flaky network.

---

## § 2 — Corpus and chunking

Ten synthetic HR/IT policy documents → 34 chunks (2-sentence windows, 1 sentence of overlap, title
prefixed).

**The claim that earns its place:** chunking is an *evaluation* decision, not just an ingestion one. If your
gold labels are document ids but you retrieve chunks, you are measuring something looser than what the
generator sees. §3.3 closes that gap explicitly.

**Steal:** `chunk_documents()` — the chunk id carries its parent document (`POL-003::c1`), which is what
makes citations, metadata filters and label projection possible at all.

---

## § 3 — The labelled query set ⭐

**The section that answers half the feedback.** Metrics are easy; labels are the work.

- **30 queries**: 8 factoid, 6 paraphrase, 4 conditional, 4 multi-hop, 3 multi-fact, 3 acronym,
  3 unanswerable.
- **Graded relevance** (3 = contains the answer, 2 = needed for a complete answer, 1 = useful context).
- **Verbatim `answer_spans`**, which make chunk-level labels *reproducible* rather than hand-waved.
- A **label verifier** that asserts every answer span occurs inside its gold document. A label set
  referencing text that is not in the corpus is the most common silent bug in retrieval evaluation, and it
  always inflates your scores.
- **§3.4 bootstraps labels with Claude** and then measures the bias it introduces: generated queries reuse
  the source chunk's vocabulary (0.34 overlap here vs 0.20 for the hand-written paraphrase queries), which
  **flatters lexical retrieval**. That is why LLM-generated sets are *silver*, not gold.

The query mix is deliberate. Paraphrase queries are where dense wins; acronym and exact-figure queries are
where BM25 wins; unanswerable queries that are *lexically adjacent* to real policies are what make
abstention hard. Without that mix, §6's hybrid comparison would be a tie and §9's threshold calibration
would look better than it is.

**Steal:** the label schema and `verify_labels()`.

---

## § 4 — Metrics from scratch ⭐

Seven metrics over one interface: a ranked list of ids and a `{id: grade}` dict.

`recall@k` · `precision@k` · `success@k` · `MRR@k` · `MAP@k` · `NDCG@k` (graded) · `R-precision`

**What makes this section different from a formula dump:**

- A table of **what each metric is blind to**, and when to use it. Recall@k is the RAG headline because it
  is the *ceiling* on answer quality. Precision@k with 1–2 relevant chunks is bounded by `n_rel/k` and says
  more about your labels than your system.
- **Tested.** A hand-worked 6-document example with asserts, the degenerate cases that break naive
  implementations (unanswerable → `NaN` not `0`; `k` beyond the list length; a perfect ranking must be
  exactly 1.0), and an **independent cross-check of NDCG against scikit-learn**.
- The notebook prints exponential-gain NDCG (0.6236) next to linear-gain NDCG (0.5906) **for the same
  ranking**, because both are called "NDCG" and the difference is bigger than most reported improvements.

**Steal:** the whole of §4.1 and §4.2. If you copy one thing from this repo, copy this.

---

## § 5 — Retrievers

BM25, dense, RRF hybrid, weighted hybrid — all behind `search(query, k) -> [(id, score)]`.

Two things worth internalising:

- **Raw score addition is a bug.** BM25 scores are unbounded; cosine lives in [−1, 1]. Adding them makes
  BM25 the only thing that matters. RRF sidesteps it by using ranks only; weighted fusion needs per-query
  normalisation (z-score here) and a tuned `alpha`.
- A document missing from one retriever's pool gets that pool's **floor**, not 0 — because 0 is an *average*
  z-score, which would reward documents one retriever never returned.

§5.4 shows both failure modes side by side on two queries: the paraphrase query where BM25's top hit is
wrong, and the `2FA` query where dense ranks the acronym chunk below a lexical distractor.

---

## § 6 — Did the change make it better? ⭐

**The section that answers the other half of the feedback.**

- A metric table across retrievers at k = 1, 3, 5, 10, with p50/p95 latency.
- The **recall@k curve** — the most useful retrieval chart there is. It gives you the generator's ceiling
  and tells you where `top_k` stops buying answers and starts buying prompt tokens.
- The **paired bootstrap**: resample the per-query deltas 10 000 times, report the 95% CI, the win/loss/tie
  split, Cohen's d, and **the number of queries needed to resolve a ±0.05 difference** (~30 here, against
  the 27 we have — a fact that reframes every "we improved recall by 3%" claim).
- A **Wilcoxon signed-rank cross-check** of the bootstrap.
- A **per-category breakdown** and a **per-query failure table** showing what was missed and what was
  retrieved instead.

One measurement-hygiene detail worth copying: the cell **warms the embedding cache before timing anything**.
Without it, whichever retriever runs first pays the query-embedding cost and the latency column ranks
retrievers by evaluation order. That bug is in a lot of published RAG benchmarks.

**Steal:** `paired_bootstrap()` and `verdict()`. They turn "it got better" into a sentence you can defend.

---

## § 7 — Vector search internals ⭐

The longest section. Four index families built from scratch in NumPy, measured on 12 000 synthetic
128-dimensional vectors with brute-force ground truth, then mapped onto FAISS.

| Sub-section | What it proves |
|---|---|
| **7.1 Metrics** | cosine = dot = L2 ranking *on unit vectors* (asserted three ways) — and that without normalisation, dot-product search ranks by vector **length** as much as by angle. Silent, no error, just quietly worse results. |
| **7.2 Data** | Why you must benchmark ANN on data with **low effective dimensionality**. On uniformly random vectors every index collapses to exhaustive search and the sweeps teach you nothing. |
| **7.3 Flat** | One matmul, exact, and **measured scaling** from 1 k to 500 k vectors. The honest headline: at 12 000 vectors an ANN index saves you nothing, and "we measured it, flat is fine until ~2M" is a better answer than naming an index. |
| **7.4 IVF** | k-means cells + `nprobe`. Recall 0.66 → 1.00 as `nprobe` goes 1 → 32. Cell sizes are uneven (max 3.7× median), which is *why* p95 is worse than p50 by construction. `nprobe = nlist` is an exhaustive scan that is **slower than flat**. |
| **7.5 HNSW** | Layers, `M`, `efConstruction`, `efSearch` — and **both neighbour-selection rules**, so you can see the paper's diversity heuristic beat "keep the M closest" at every `efSearch`. That is the detail that explains why HNSW works. |
| **7.6 Quantization** | int8 / binary / PQ with the memory maths for 100M vectors, and the non-negotiable pattern: **quantize to shortlist, rescore to rank** (PQ 0.436 → 1.000 with a 50× rescore). Shipping a quantized index without rescoring is the most common way to silently degrade a RAG system. |
| **7.7 Trade-off chart** | Recall vs latency, log scale, every index on one plot with its own table. |
| **7.8 FAISS** | The same structures via `IndexIVFFlat`, `IndexHNSWFlat`, `IndexIVFPQ`, `IndexScalarQuantizer`, `IndexRefineFlat` — with a parameter-mapping table, and a measured from-scratch-vs-library comparison (HNSW build: 14.3 s vs 0.12 s). Also the two gotchas: `train()` before `add()`, and cosine is not a FAISS metric. |
| **7.9 Filtering** | Pre-filter vs post-filter across selectivity 0.5 → 0.002. Post-filtering at 1× over-fetch returns **0.0 of 10** requested results at 0.2% selectivity. Pre-filtering is exact everywhere *and* gets faster as the filter tightens. |
| **7.10 Deletes** | FAISS HNSW `remove_ids` raises. Graph indexes tombstone instead, which makes "delete my data" a scheduled-rebuild product decision, not an API call. |
| **7.11 Decision guide** | A sizing formula, a decision table, and the order to tune things in. |

**Steal:** §7.3's scaling measurement. It is the cell that stops an unnecessary vector-database purchase.

---

## § 8 — Reranking ⭐

The funnel: a cheap recall-oriented first stage over everything, then an expensive precision-oriented
reranker over 20–100 candidates. Cost is independent of corpus size, which is the whole economic argument.

- **Bi-encoder vs cross-encoder**, in one table: the bi-encoder compresses a document into a vector
  *before it sees the query*; the cross-encoder reads both together. That is why it can tell that
  "Alcohol is **not** reimbursable" does not answer "can I expense wine" — demonstrated on exactly that
  query.
- Three rerankers: **cross-encoder** (ms-marco-MiniLM), **Claude listwise** (structured output, with id
  validation because you trust nothing a model returns), and **MMR** (diversity, not relevance).
- The lift is measured **against each pipeline's own first stage**, which produces the honest two-row
  result: +0.083 NDCG@5 on BM25 (significant), ~0.000 on an already-saturated hybrid. Same reranker.
- **§8.5 sweeps candidate depth** and shows NDCG tracking the first-stage ceiling while latency grows —
  the knee is your production setting.

**Steal:** `RerankingRetriever`, and the habit of reporting the lift *per millisecond*.

---

## § 9 — End to end

The interview task's actual deliverable, with each choice traceable to a measurement:

- **Calibrated abstention.** Instead of guessing a score threshold, score the labelled answerable and
  unanswerable queries, look at the overlap (one of three unanswerable queries scores above the weakest
  answerable one, so a perfect threshold **does not exist**), and pick the cut that holds false-refusals
  under a target. Report false-refusal *and* false-answer rates, because one "accuracy" number hides which
  mistake you chose.
- **Groundedness ≠ correctness**, in a 2×2 — and a hybrid judge: deterministic rules (citations must exist
  *and* have been retrieved; every number in the answer must appear in a cited chunk) plus an LLM judge for
  the paraphrase-level part. Rules cannot be talked out of a verdict, which matters because an LLM judging
  an LLM is correlated error.
- The required metrics **plus** citation validity, abstention accuracy and latency — each with a sentence
  on *why it is reported*, and a closing list of what the numbers do **not** say.
- **§9.6** is the design justification written the way a walk-through wants it: a decision, the number
  behind it, and the thing it costs.

---

## § 10 — Your lab

`run_experiment()` / `leaderboard()` / `compare()` for sweeping configurations, then **10 exercises** that
are each a real client decision, and the **walk-through cheat sheet**.

One finding in §10.1 is worth the whole section: whole-document chunking tops the leaderboard on recall@5 —
for a **labelling** reason, not a quality one (a whole document contains every answer span, so there is
exactly one relevant chunk and recall@5 is nearly free), while the generator now reads five entire policies.
Changing the chunker changes the labels, the metric's meaning *and* the prompt cost at once. That is why so
many chunking A/B results are bogus, and the notebook says so next to the table that would otherwise mislead
you.

---

## Where it is deliberately thin

Stated plainly, because a reference that hides its limits is not a reference:

- **27 answerable queries is a small set.** The notebook computes and prints what that n can and cannot
  resolve. More labels is the highest-value contribution anyone can make ([CONTRIBUTING.md](../CONTRIBUTING.md)).
- **No held-out split by default.** The abstention threshold and reranking depth are tuned on the same
  queries they are reported on. Exercise 7 fixes it; promoting that into the notebook proper is an open task.
- **The corpus is synthetic and tiny.** Good for teaching mechanics, useless for predicting your numbers.
- **ANN recall is measured against exact search**, not against human labels. They are different questions
  and the notebook keeps them separate.
- **No multilingual, no tables, no images, no stale-document handling.** All real RAG problems; none of them
  are in scope here.
