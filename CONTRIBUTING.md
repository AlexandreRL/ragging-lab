# Contributing to ragging-lab

**Welcome — genuinely.** 👋

This repo exists to make retrieval evaluation and vector search internals *click* for people. If you are
here, you are already the audience, which means your confusion, your question, and your "actually, that's
not quite right" are all useful contributions. You do not need to be an IR researcher. You do not need to
write code. You do not need permission.

**Two promises from the maintainer:**

1. Every issue and PR gets a real human reply, usually within a few days. "This isn't a good fit" comes with
   a reason, and an alternative where there is one.
2. There are no stupid questions here. If something in the notebook was unclear, that is a **defect in the
   notebook**, not in you — and reporting it is one of the most valuable things you can do.

---

## The 60-second version

```bash
# 1. fork and clone, then set up (see docs/SETUP.md if anything misbehaves)
py -3.12 -m venv .venv                     # python3.12 -m venv .venv on macOS/Linux
.venv\Scripts\python -m pip install --upgrade pip
.venv\Scripts\python -m pip install torch --index-url https://download.pytorch.org/whl/cpu
.venv\Scripts\python -m pip install -r requirements.txt

# 2. branch off dev (not main - see docs/BRANCHING.md)
git checkout dev && git pull
git checkout -b my-change

# 3. make your change, then prove the notebook still runs top to bottom
.venv\Scripts\python -m jupyter nbconvert --to notebook --execute --inplace \
    retrieval_eval_and_vector_search_internals.ipynb

# 4. push and open a PR against `dev`
```

That last step matters more than usual here: **the notebook is the product**, so "it runs clean from a fresh
kernel" is the test suite.

---

## Ways to contribute, from smallest to largest

### 🟢 Needs no setup at all

- **"I got lost here."** Open an issue with the cell number and what you expected. Seriously — the whole
  point of a teaching notebook is that someone learns from it, and the places where that fails are invisible
  to the author.
- **Fix a typo, a broken link, or my inconsistent British/American spelling.** Edit on GitHub, open a PR.
- **Improve an explanation.** If a paragraph took you three reads, rewrite it and show us.
- **Report a number that looks wrong.** The notebook prints a lot of measurements. If one contradicts your
  experience, say so — that is a great issue.

### 🟡 Small and high value

- **Add labelled queries to §3.2.** This is the highest-leverage change anyone can make. The notebook itself
  computes that ~30 queries are needed to resolve a 0.05 difference in recall@5 — more labels directly
  widens what the lab can prove. Each query needs: a question, `gold` document grades, verbatim
  `answer_spans`, and a `category`. §3.3's verifier will tell you immediately if a span does not occur in
  the corpus.
- **Add a metric** to §4.1 — ERR, Rank-Biased Precision, recall with graded gains. **Must come with a
  hand-worked assert** in the style of §4.2; that is the house rule and it exists because untested metric
  code has burned all of us.
- **Add an exercise** to §10.2, phrased as a decision a client would actually ask about.
- **Tune something and report it.** Run a sweep, post the table and the paired-bootstrap verdict in an
  issue. Negative results are welcome and genuinely useful.

### 🔵 Bigger, very welcome

- **Port the lab to a real vector store.** A new section that runs the same measurements against Qdrant,
  pgvector, Chroma, Weaviate or LanceDB — ideally including how each one handles filtering and deletes,
  because that is where the differences actually bite.
- **Add an index family.** OPQ, DiskANN-style on-disk search, ScaNN-style anisotropic quantization, IVF-HNSW
  hybrids. Keep the from-scratch-then-library structure.
- **Add a reranker.** ColBERT-style late interaction, a monoT5 variant, or a learned fusion.
- **Add a held-out split and a leakage audit** (exercise 7, promoted into the notebook proper).
- **CI that executes the notebook** on push. There is deliberately none yet; a lightweight GitHub Action
  that runs it offline and fails on any cell error would be a great first infrastructure PR.

Not sure which to pick? Open an issue describing what you *want to learn*, and we will find the overlap.

---

## House rules for notebook code

The notebook is teaching material, so the bar is different from library code. Five rules:

1. **Readable beats clever.** Someone is reading this to learn the algorithm. Name the paper's variables
   (`M`, `efConstruction`, `nprobe`) so the mapping is obvious, and prefer an explicit loop over a dense
   one-liner when the loop is the lesson.
2. **Comment the *why*, not the *what*.** `# normalise the centroids so coarse scoring matches fine scoring`
   earns its place; `# loop over clusters` does not.
3. **No unmeasured claims.** If you write "faster" or "better", the cell must print the number. This is the
   repo's whole personality.
4. **State the trade-off.** Every technique costs something. Say what, in the markdown next to it.
5. **Keep it runnable offline and deterministic.** Seed your RNGs from `CFG.seed`. If your change needs a
   network call or an API key, it must degrade gracefully — look at how `LLM` and `Embedder` in §1 do it, and
   match that pattern.

Style, loosely: 100-column lines, type hints on public functions, docstrings that say what the thing is
*for*. There is no formatter gate — a maintainer will not reject a PR over whitespace.

### If you add a code cell

- Put it in the section it belongs to, with the `# --- N.M title ---` header comment convention.
- Print or `display()` something. A cell that computes silently teaches nothing.
- Pair a chart with its DataFrame. Every chart in this notebook ships with its table, so the numbers are
  copyable and the figure is accessible to screen readers and colour-blind readers.
- Follow the palette: `SERIES[0..4]` in fixed order, never cycled. It is validated for colour-vision
  deficiency, and re-assigning hues by rank breaks that.

---

## Submitting a pull request

**Target `dev`**, not `main`. See [docs/BRANCHING.md](docs/BRANCHING.md).

Checklist (the PR template repeats it):

- [ ] The notebook runs top to bottom from a **fresh kernel** with **no API key**, zero errors.
- [ ] If you added a metric or an algorithm, there is an assert that would fail if it were wrong.
- [ ] New claims are backed by printed output.
- [ ] Committed notebook outputs are from your final run (`Restart and Run All`, then save).
- [ ] `docs/` updated if you changed setup, structure, or added a section.

**On notebook diffs:** they are ugly, and that is fine. We commit outputs on purpose so the notebook is
readable on GitHub without running it. If the diff is unreviewable, say what changed in the PR description
and a maintainer will work with it. Do not strip outputs to make the diff pretty.

**Small PRs get merged faster** than big ones, and a PR that does one thing is easier to say yes to.

---

## Reporting a bug

Open an issue with:

- The **cell number** and the error, including the traceback tail.
- Your platform, Python version, and whether you are running with or without an API key.
- `pip list` output for `numpy`, `torch`, `faiss-cpu`, `transformers`, `sentence-transformers` if the failure
  looks like an import or a DLL problem.

Already-known platform traps (not bugs, but check here first): the Windows **Smart App Control** `regex`
DLL block, and the `sentence-transformers` first-run model download. Both are in
[docs/SETUP.md § Troubleshooting](docs/SETUP.md#troubleshooting).

---

## Questions, ideas, half-formed thoughts

Open an issue. "Is reciprocal rank fusion really better than tuning alpha?" is a perfectly good issue title
and exactly the kind of discussion this repo is for. If you would rather ask quietly, say so in the issue and
keep it short — nobody is judging the question.

---

## Recognition

Contributors are credited in the release notes and, for anything beyond a typo, in the notebook section they
touched. If you would rather not be named, say so and you will not be.

Thank you for being here. Go break something and tell us what you learned. 🔬
