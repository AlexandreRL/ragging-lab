# Glossary

Every term the lab uses, in one or two lines, with **the thing it is blind to** — because that is the part
interviews and client reviews actually probe. Section references point into
[`retrieval_eval_and_vector_search_internals.ipynb`](../retrieval_eval_and_vector_search_internals.ipynb).

---

## Evaluation

**Labelled query set / qrels** — queries paired with relevance judgements, `{query: {doc: grade}}`. The
apparatus everything else depends on. Sourcing and bias: §3.

**Graded relevance** — judgements on a scale (here 3 = contains the answer, 2 = needed for a complete
answer, 1 = useful context, 0 = irrelevant) rather than relevant/not. Required for NDCG to mean anything.

**Relevance threshold** — the grade at which a binary metric counts an item as relevant. This lab uses
**≥ 2**. *Always state it*: recall@5 at threshold 1 can be 20 points higher than at threshold 2, and both
are honestly called "recall@5".

**Answer span** — a verbatim substring of the corpus that answers a query. Makes chunk-level labels
reproducible instead of hand-waved, and lets a verifier catch labels that reference text not in the corpus.

**Silver vs gold labels** — *silver* is LLM- or heuristic-generated; *gold* is human-reviewed. Silver sets
inherit the generator's vocabulary, which inflates lexical retrieval (§3.4 measures this).

**Pooling bias** — if your labelled set was built by labelling what *your current system* retrieved, a new
system's genuinely good results look unlabelled, therefore irrelevant, therefore wrong. Classic way to prove
your new retriever is worse than it is.

**Recall@k** — of all relevant items, the fraction in the top k. **The RAG headline**: it is the ceiling on
answer quality, since nothing unretrieved can be cited. *Blind to:* ordering within k, and precision.

**Precision@k** — of the top k, the fraction relevant. *Blind to:* what you missed. With 1–2 relevant items
it is capped at `n_rel/k` and tells you more about your labels than your system — never report it alone.

**Success@k / hit rate** — did at least one relevant item land in the top k. *Blind to:* how many, and where.

**MRR@k** — mean reciprocal rank of the **first** relevant item. *Blind to:* everything after that first
hit, which makes it the wrong metric for multi-hop questions.

**MAP@k** — mean of precision@i over every rank i holding a relevant item. Rewards getting *all* relevant
items high. *Blind to:* graded relevance (binary only).

**NDCG@k** — discounted cumulative gain over graded relevance, normalised by the best achievable ordering
of the labelled items. The IR default, and the right metric for judging a reranker. Two things to state:
the **gain function** (`2^g − 1` exponential vs `g` linear — the lab prints 0.6236 vs 0.5906 for the same
ranking) and the **k**.

**IDCG (ideal DCG)** — DCG of the perfect ordering. Normalising by it is what makes NDCG comparable across
queries with different numbers of relevant items.

**R-Precision** — precision at k = the number of relevant items. Self-normalising, useful when queries vary
wildly in how many relevant items they have.

**Paired bootstrap** — resample the *per-query differences* between two systems with replacement, take the
mean each time, read the 2.5th/97.5th percentiles. Pairing removes query difficulty, the dominant source of
variance. §6.3.

**Win/loss/tie** — how many queries each system won. "+0.07 on average" can mean "helped 20 queries a
little" or "helped 3 enormously and broke 5"; those call for different decisions.

**Cohen's d** — effect size: mean difference over its standard deviation. Scale-free, so you can compare
"how big" across metrics.

**Required n** — the number of queries needed to resolve a given difference at 95% confidence, computed from
the observed spread. The single most useful sentence to say to a client who asks for a 1% improvement claim.

**Multiple comparisons** — sweep 10 configurations, report the best p-value, and you have found noise.
Correct for it, or keep a held-out set you touch once.

**Groundedness** — does every claim in the answer *follow from the retrieved context*. **Not** correctness:
a grounded answer can be wrong if the corpus is wrong, and a correct answer can be ungrounded if it came
from the model's memory. §9 has the 2×2.

**LLM-as-judge** — using a model to grade outputs. Fast and scalable; correlated with the generator's own
errors, which is why this lab pairs it with deterministic rules.

**Abstention / calibrated refusal** — declining to answer when retrieval is too weak, with the threshold
chosen from labelled answerable vs unanswerable queries rather than guessed. Report **false-refusal** and
**false-answer** rates separately.

---

## Retrieval

**Sparse retrieval / BM25** — ranking by exact term matches weighted by rarity, with term-frequency
saturation (`k1`) and length normalisation (`b`). Wins on IDs, numbers, acronyms. *Fails on* vocabulary
mismatch: zero shared tokens, zero score.

**Dense retrieval / bi-encoder** — query and documents embedded separately; search by vector similarity.
Wins on paraphrase. *Fails on* exact-token precision, because the document was compressed into one vector
before the query existed.

**Cross-encoder** — query and document scored **together** in one forward pass with full cross-attention.
Markedly better, and impossible to precompute — hence a reranker over a short candidate list, never a
retriever.

**Hybrid retrieval** — combining sparse and dense, because their failure modes are nearly disjoint.

**Reciprocal Rank Fusion (RRF)** — fuse rankings with `Σ 1/(k + rank)`, `k ≈ 60`. Uses ranks only, so no
score normalisation and no tuned weight. Ship this first.

**Weighted score fusion** — `α·z(dense) + (1−α)·z(sparse)`. Keeps score magnitude; needs per-query
normalisation and an `α` fitted on a labelled set.

**Score normalisation** — z-score or min-max, per query. Required before mixing scores from different
retrievers: BM25 is unbounded, cosine lives in [−1, 1].

**MMR (Maximal Marginal Relevance)** — iteratively pick `argmax [λ·sim(q,d) − (1−λ)·max sim(d, picked)]`.
Trades relevance for diversity; the RAG use case is multi-hop questions whose top-k would otherwise be five
near-duplicate chunks of one document.

**Candidate depth** — how many documents the first stage hands the reranker. The funnel's real parameter:
a reranker cannot fix what the first stage never returned, and cost is linear in depth. §8.5.

**Chunking** — splitting documents for indexing. Small chunks: sharper vectors, higher precision, better
citations, but multi-sentence answers straddle boundaries. Large chunks: the opposite, plus more prompt
tokens per hit. And it changes what your labels *mean* (§2, §10.1).

**Chunk overlap** — repeating a sentence or two between adjacent chunks. Cheap insurance against boundary
straddling.

---

## Vector search internals

**ANN (approximate nearest neighbour)** — trading exactness for speed or memory. Judged by **recall against
exact search**, which is a different question from retrieval quality against human labels.

**Flat index** — exhaustive scan, one matmul. Exact, no training, instant updates, and viable far longer than
people expect: §7.3 measures it.

**IVF (inverted file)** — k-means partitions the space into `nlist` cells; search scans the `nprobe` nearest.
`nlist ≈ √n` to start. *Characteristic failure:* the cell-boundary miss — a true neighbour just across a
Voronoi boundary in an unprobed cell is invisible.

**nprobe** — IVF's search-time dial, settable per query. The unusual and valuable property: one index can
serve a "fast path" and an "accurate path".

**HNSW** — a multi-layer proximity graph; search enters at a sparse top layer and greedily descends. The
default in most vector databases.

**M** — links per HNSW node (layer 0 gets `2M`). Raises the recall *ceiling*; costs memory (`M·2·4` bytes
per vector) and build time. Changing it means a rebuild.

**efConstruction** — candidate pool size while inserting. Builds a permanently better graph; costs build
time only and is **free at query time**.

**efSearch** — candidate pool size while searching. The production dial. If recall plateaus below target as
you raise it, the *graph* is the limit — raise `M`.

**Neighbour-selection heuristic** — keep a candidate only if it is closer to the new node than to any
already-kept neighbour. Preserves long-range "bridge" edges; "keep the M closest" creates mutually-close
clusters with no exits and greedy search gets stuck. §7.5 measures the difference.

**Tombstone delete** — graph indexes cannot remove a node without disconnecting the graph, so they mark it
deleted and filter it from results. Consequences: it still occupies RAM, is still traversed as a hop, and
recall drifts down until you rebuild. This is why "delete my data" on a vector store is a scheduled
rebuild, not an API call.

**Scalar (int8) quantization** — one byte per dimension from a per-dimension min/max. 4× smaller, ~1% recall
cost. The free win.

**Binary quantization** — one bit per dimension (the sign), compared with Hamming distance via popcount.
32× smaller, very fast, and large recall loss **without rescoring** (0.273 → 0.997 with a 50× rescore here).

**Product quantization (PQ)** — split the vector into `m` sub-vectors, k-means each subspace, store one
code per sub-vector. 8–64× compression. Lossy, and recall **caps out** — more `nprobe` will not fix it.

**OPQ** — PQ preceded by a learned rotation that decorrelates dimensions, so correlated dimensions stop
landing in the same sub-vector and wasting code space. FAISS writes it `OPQ96,IVF4096,PQ96`.

**ADC (asymmetric distance computation)** — PQ's query-time trick: build one lookup table of partial
distances from the uncompressed query, then score each vector as a sum of `m` table reads.

**Rescoring / refine** — use compressed codes to pull a shortlist 5–20× longer than k, then re-score those
few with full-precision vectors. **The non-negotiable companion to any compression.** FAISS:
`IndexRefineFlat` with `k_factor`.

**Pre-filter vs post-filter** — pre-filter selects the allowed subset and scans it exactly (always correct;
gets *faster* as the filter tightens). Post-filter runs ANN then drops non-matches (returns fewer than k as
selectivity falls, because the index never saw the filter). §7.9.

**Selectivity** — the fraction of the corpus a metadata filter admits. Low selectivity breaks HNSW
traversal, because the graph you are walking is mostly excluded nodes.

**Curse of dimensionality** — in high dimensions with no structure, all distances converge and nearest
neighbours stop being meaningful. ANN works on real embeddings because they have **low effective
dimensionality**; benchmark on uniform random vectors and every index looks broken (§7.2, exercise 6).

**DiskANN** — graph on SSD with compressed vectors in RAM, for corpora beyond memory. Latency becomes an
IOPS budget.

**Recall@k vs exact search** — the ANN quality metric. An index at 0.9 recall caps your end-to-end retrieval
quality at roughly 0.9× whatever your labelled evaluation measured.

---

## LLM layer

**Structured output** — constraining the response to a JSON schema (`client.messages.parse` with a Pydantic
model). Replaces "respond only with JSON" prompting, fence-stripping and repair loops.

**`stop_reason`** — always check it before reading `content`. A `refusal` returns HTTP 200 with no usable
text.

**Server-side fallbacks** — route a refused request to another model automatically
(`betas=["server-side-fallback-2026-07-01"]`, `fallbacks="default"`, on the beta namespace). The lab enables
it and detects-and-disables when an organisation does not have it.

**Graceful degradation** — probe the credential at start-up, classify the failure (permanent 401/403 vs
transient 429/network), and keep a deterministic implementation of the same interface behind the same
function signature. §1.2.

**Citation validity** — every cited id must exist *and* have been in the retrieved context. Catches the
failure where a model cites from memory; a citation that looks right and points nowhere is worse than none.
