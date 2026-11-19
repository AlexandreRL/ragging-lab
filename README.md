# ragging-lab

**A runnable lab for the two things people get asked about RAG and usually cannot answer with numbers:
_how good is your retrieval_, and _which index did you choose and why_.**

[![Python 3.12](https://img.shields.io/badge/python-3.12-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Runs offline](https://img.shields.io/badge/runs-fully%20offline-success.svg)](#quickstart)
[![Notebook](https://img.shields.io/badge/notebook-64%20cells%2C%20~90s%20run-orange.svg)](retrieval_eval_and_vector_search_internals.ipynb)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

Most RAG tutorials stop at "it works on my example". This repo starts there and keeps going:
**every claim in it is a measurement you can re-run, change, and break.** No quoted benchmarks, no
"HNSW is faster" without a timing, and no "our retrieval improved" without a confidence interval.

👋 **New here?** Jump to the [Quickstart](#quickstart) — two commands and you are running.
Want to help? [CONTRIBUTING.md](CONTRIBUTING.md) has genuinely small first tasks and a promise about
review turnaround. **Every kind of contribution counts**, including "this paragraph confused me".

---

## Why this exists

This started as interview feedback on a RAG take-home. Two gaps, quoted:

> **Retrieval evaluation** — "the interviewer was hoping to see more depth in measuring retrieval quality
> (for example recall at k, MRR or NDCG over a labelled query set). On client projects, this is how you show
> that a RAG system actually works and that a change made it better."
>
> **Vector search internals** — "you had limited exposure to index types, their trade-offs and reranking.
> Clients often expect the engineer to choose and tune these on their own."

That is an extremely common pair of gaps — the industry ships a lot of RAG and measures very little of it.
So rather than read about it, this repo **builds it**: a labelled query set, the metrics from scratch
(unit-tested), a significance test, four index families implemented in NumPy and then mapped onto FAISS,
three rerankers, and a RAG system whose every design choice points at a number in the notebook.

It is a learning artifact, so the code optimises for **readability over speed** — and then measures exactly
how much that costs you (spoiler: the pure-Python HNSW build is 118× slower than FAISS, and the notebook
prints the ratio).

---

## What you will be able to do afterwards

<table>
<tr><th>Track 1 — Retrieval evaluation</th><th>Track 2 — Vector search internals</th></tr>
<tr valign="top"><td>

- Build a **labelled query set** with graded relevance, answer spans, multi-hop and unanswerable queries
- Implement **recall@k, precision@k, success@k, MRR, MAP, NDCG@k, R-precision** — and know what each one
  is blind to
- Project document-level labels onto chunks with a **rule you can defend**
- Prove a change helped with a **paired bootstrap CI**, win/loss counts, and an effect size
- Say how many labelled queries you would need to resolve a 0.05 difference (on this corpus: **~30**)
- Bootstrap labels with an LLM, and name the bias that introduces

</td><td>

- Build **Flat, IVF, HNSW, PQ, int8 and binary** indexes from scratch in NumPy
- Tune `nprobe` / `efSearch` / code size against a **recall target and a latency budget**
- Explain **why HNSW's neighbour-selection heuristic exists** — and measure the recall it buys
- Know why a quantized index needs a **rescoring pass**, and what it costs
- Map all of it onto **FAISS** (`IndexIVFFlat`, `IndexHNSWFlat`, `IndexIVFPQ`, `IndexRefineFlat`, …)
- Choose between **pre-filter and post-filter** for metadata, and explain tombstone deletes
- Rerank with a **cross-encoder, an LLM, or MMR**, and price the lift per millisecond

</td></tr></table>

---

## Quickstart

**Requires** Python 3.12 (3.10+ works), ~3 GB disk for the CPU PyTorch wheel, and no API key.

```bash
git clone https://github.com/AlexandreRL/ragging-lab.git
cd ragging-lab

# Windows (PowerShell)
py -3.12 -m venv .venv
.venv\Scripts\python -m pip install --upgrade pip
.venv\Scripts\python -m pip install torch --index-url https://download.pytorch.org/whl/cpu
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\python -m ipykernel install --sys-prefix --name ragging-lab --display-name "Python (ragging-lab)"

# macOS / Linux
python3.12 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m ipykernel install --sys-prefix --name ragging-lab --display-name "Python (ragging-lab)"
```

Then open [`retrieval_eval_and_vector_search_internals.ipynb`](retrieval_eval_and_vector_search_internals.ipynb)
in VS Code, make sure the kernel is the project `.venv`, and `Run All`. About **90 seconds** end to end.

The notebook is committed **with its outputs**, so you can also just read it on GitHub before running
anything.

**Optional — switch on the Claude cells.** `cp .env.example .env` and set `ANTHROPIC_API_KEY`. The notebook
probes the key with one ~10-token call at start-up and, if anything fails, prints why and silently uses
deterministic offline implementations of the same interfaces. Nothing breaks either way.

Full walkthrough, platform notes and troubleshooting: **[docs/SETUP.md](docs/SETUP.md)**
(including the Windows **Smart App Control** gotcha that breaks `import transformers`).

---

## Repo tour

| Path | What it is |
|---|---|
| **[`retrieval_eval_and_vector_search_internals.ipynb`](retrieval_eval_and_vector_search_internals.ipynb)** | **The lab.** 64 cells, 10 sections, runs offline in ~90 s. Start here. |
| [`alexandre_ai_engineer_rag_with_evaluation.ipynb`](alexandre_ai_engineer_rag_with_evaluation.ipynb) | The original interview task, unmodified — the scaffold the lab answers |
| [`docs/SETUP.md`](docs/SETUP.md) | Install, kernels, platform gotchas, troubleshooting table |
| [`docs/NOTEBOOK_TOUR.md`](docs/NOTEBOOK_TOUR.md) | Section-by-section guide: what each one proves, and the cell to steal |
| [`docs/GLOSSARY.md`](docs/GLOSSARY.md) | Every metric and index term, defined in one line with its failure mode |
| [`docs/BRANCHING.md`](docs/BRANCHING.md) | `main` / `dev` / `testing` — where to open your PR |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | How to help, with concrete first tasks |

### The lab's ten sections

| § | Section | The point |
|---|---|---|
| 1 | Setup | Config in one dataclass; Claude client that probes and degrades; three embedding backends |
| 2 | Corpus & chunking | Chunking is an *evaluation* decision, not just an ingestion one |
| 3 | **Labelled query set** | 30 queries, graded relevance, answer spans, 4 multi-hop, 3 unanswerable — plus label verification that fails loudly |
| 4 | **Metrics from scratch** | 7 metrics, a hand-worked example, asserts, and a cross-check against scikit-learn |
| 5 | Retrievers | BM25, dense, RRF and weighted hybrids behind one 2-line interface |
| 6 | **Did it get better?** | Metric table, recall@k curve, **paired bootstrap CI**, per-category breakdown, error analysis |
| 7 | **Vector search internals** | Flat / IVF / HNSW / PQ / int8 / binary from scratch → measured → mapped onto FAISS; filtering; deletes |
| 8 | **Reranking** | Cross-encoder, Claude listwise, MMR; NDCG lift priced per millisecond; candidate-depth knee |
| 9 | End-to-end `RAGSystem` | Citations, calibrated abstention, rules + LLM-judge groundedness, honest metrics |
| 10 | Your lab | A config sweep harness, a leaderboard, 10 exercises, and a walk-through cheat sheet |

---

## A few things the notebook actually measures

Real output, reproducible on a laptop CPU. The point is not the numbers — it is that they are *measured*.

**Compression without rescoring is a silent quality regression** (12 000 vectors × 128-d, recall@10 vs exact
search):

| Index | Bytes/vector | Compression | recall@10 | RAM for 100M vectors |
|---|---|---|---|---|
| Flat float32 | 512 | 1× | 1.000 | 51.2 GB |
| int8 scalar | 128 | 4× | 0.986 | 12.8 GB |
| Binary, no rescore | 16 | 32× | **0.273** | 1.6 GB |
| Binary + rescore 50× | 16 | 32× | **0.997** | 1.6 GB |
| PQ (m=16), no rescore | 16 | 32× | **0.436** | 1.6 GB |
| PQ + rescore 50× | 16 | 32× | **1.000** | 1.6 GB |

**HNSW's neighbour-selection heuristic is not a detail.** Same corpus, same `M`, only the selection rule
changes — keeping the `M` closest neighbours instead of the paper's diversity rule costs recall at every
`efSearch`, because the graph loses its long-range bridge edges.

**A reranker's value depends entirely on the first stage you can afford.** Paired bootstrap over 27
answerable queries:

| Change | NDCG@5 | 95% CI | Cost |
|---|---|---|---|
| BM25 → BM25 + cross-encoder | 0.825 → **0.908** | [+0.004, +0.184] | +101 ms |
| Strong hybrid → hybrid + cross-encoder | 0.932 → 0.932 | [−0.033, +0.038] | +104 ms |

Same reranker, same corpus. The second row is near the metric's ceiling, so there is no headroom to buy —
and reporting only the first row would be the kind of result that falls apart in front of a client.

**And the one most teams skip:** on this 27-query set, resolving a ±0.05 difference in recall@5 at 95%
confidence needs **~30 labelled queries**. The notebook computes that from the observed spread and prints it,
because "we improved recall by 3%" on 27 queries is not a finding.

---

## Design principles

1. **Every number is measured in the notebook.** If you cannot re-run it, it is not in here.
2. **Every metric implementation is tested** — a hand-worked example with asserts, plus an independent
   cross-check against scikit-learn. Metrics you have not tested are rumours.
3. **Every "it got better" comes with a confidence interval**, win/loss counts, and the sample size needed to
   resolve the effect.
4. **Honest findings beat flattering ones.** The notebook says out loud that at 12 000 vectors an ANN index
   buys you nothing, that whole-document chunking wins its row for a *labelling* reason, and that an
   LLM judging an LLM is correlated error.
5. **Readable over fast**, then measure what readability cost.

---

## FAQ

**Do I need an API key?** No. Everything runs offline. A Claude key switches on four cells; the notebook
tells you exactly what it fell back to and why.

**Does it use OpenAI?** The derived lab uses Claude (or nothing). The original interview notebook imports
`openai`; it is included unmodified for reference.

**Is this specific to the policy-document corpus?** No — swap `DOCUMENTS` for your own, write ~30 labelled
queries, and the whole notebook becomes a measurement of *your* system. That is why the interfaces are
deliberately two lines wide.

**Why implement indexes by hand instead of just using FAISS?** Because `nprobe`, `efSearch` and `m` stop
being magic strings once you have built the thing they control. §7.8 then shows the FAISS equivalents side
by side, and recommends you ship the library.

**Can I use this to prepare for an interview?** That is what it was built for. §10.3 is a cheat sheet of
"why did you choose X" questions with the one-line answer each — grounded in the measurements above it.

**Is there a GPU requirement?** No. CPU only, by design.

---

## Contributing

**Outside contributions are the point of making this public.** Whether you are a beginner who got stuck on
cell 12 or an IR researcher who thinks my NDCG normalisation is debatable — there is a place for your input
here, and you will get a friendly reply.

Good places to start, no deep expertise needed:

- 📝 **Tell us where the notebook confused you.** Genuinely the most valuable issue type.
- 🔤 **Add labelled queries** to §3.2 — more labels is the single highest-leverage improvement in the repo.
- 🧪 **Port the lab to another vector store** (Qdrant, pgvector, Chroma, LanceDB) as a new section.
- 📊 **Add a metric** (Recall@k with graded gains, ERR, Rank-Biased Precision) with a tested implementation.
- 🌍 **Translate the markdown** — or just fix my British/American spelling drift.

Read **[CONTRIBUTING.md](CONTRIBUTING.md)** for the 5-minute version, and
**[CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md)** for how we treat each other (short version: be kind, assume
good faith).

---

## Credits and licence

Built by [Alexandre Rebouças Leite](https://github.com/AlexandreRL). MIT licensed — see [LICENSE](LICENSE).
Use it in your own work, teach from it, fork it for a client project.

Standing on the shoulders of: [HNSW](https://arxiv.org/abs/1603.09320) (Malkov & Yashunin),
[Product Quantization](https://ieeexplore.ieee.org/document/5432202) (Jégou et al.),
[Reciprocal Rank Fusion](https://dl.acm.org/doi/10.1145/1571941.1572114) (Cormack et al.),
[FAISS](https://github.com/facebookresearch/faiss), [sentence-transformers](https://www.sbert.net/),
and [rank_bm25](https://github.com/dorianbrown/rank_bm25).

⭐ If this made retrieval evaluation or ANN indexes click for you, a star helps other people find it.
