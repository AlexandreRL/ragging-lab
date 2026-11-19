# Branches and workflow

Three long-lived branches. The model is deliberately simple — this is a teaching repo, not a service with
a release train.

```
                    you fork / branch from dev
                              │
  feature/my-change ──────────┘
          │  PR
          ▼
        dev  ──────────► testing ──────────► main
     integration        verification        published
   (default target)   (full clean runs)   (always works)
```

| Branch | What it holds | Who merges into it | Stability promise |
|---|---|---|---|
| **`main`** | The published, known-good state. Anyone cloning this repo gets something that runs. | Maintainer only, from `testing` | **Always runs clean, top to bottom, with no API key.** |
| **`testing`** | A candidate for `main`, being verified | Maintainer, from `dev` | Expected to work; may be mid-verification |
| **`dev`** | Day-to-day integration — **open your PR here** | Anyone, via PR | Should run, but may be briefly rough |

### What is actually enforced

A GitHub ruleset (`require PR`, active) backs this up on **`main`** only:

| Rule on `main` | Effect |
|---|---|
| Pull request required | No direct pushes to `main`, including by the maintainer. **0 approvals required**, so a solo maintainer can merge their own PR. |
| No force-push | `main`'s history cannot be rewritten |
| No deletion | `main` cannot be deleted |

**`dev` and `testing` accept direct pushes.** Day-to-day work does not need the PR ceremony; the gate sits
where the promise is, on the branch people clone. Outside contributors still go through a PR into `dev`,
because they do not have write access in the first place — the ruleset is not what stops them, repository
permissions are.

### Why `testing` exists at all

Because "the notebook runs" is the only test this repo has, and verifying it properly means a **fresh kernel,
a clean cache, no API key, and then again *with* a key**. That takes a few minutes of wall clock and is
worth doing away from `main`. A notebook that fails at cell 40 for a newcomer is the worst possible first
impression for a learning repository.

---

## Working on a change

```bash
git checkout dev
git pull origin dev
git checkout -b feature/add-erl-metric      # or fix/..., docs/...
# ... work ...
git add -A
git commit -m "Add Expected Reciprocal Rank to section 4"
git push -u origin feature/add-erl-metric
# open a PR against dev
```

Branch naming, loosely: `feature/…`, `fix/…`, `docs/…`, `exp/…` for exploratory work you may throw away.
Nobody will bounce your PR over a branch name.

---

## Before you push

The repo's test suite is the notebook itself:

```powershell
.venv\Scripts\python -m jupyter nbconvert --to notebook --execute --inplace `
    --ExecutePreprocessor.timeout=1800 retrieval_eval_and_vector_search_internals.ipynb
```

Zero cell errors, from a fresh kernel, **with no API key set** — that is the gate. Then
`Restart and Run All` in the editor and save, so the committed outputs match your final code.

Notebook outputs **are** committed, on purpose: the notebook is readable on GitHub without running it, which
is most of its value to a newcomer. Diffs are consequently large and ugly. Describe what changed in the PR
body; do not strip outputs to make the diff pretty.

---

## Promotion

A maintainer merges `dev → testing`, verifies:

1. Clean clone, fresh venv, `pip install -r requirements.txt`.
2. Notebook executes end to end **offline**, zero errors.
3. Notebook executes end to end **with** an `ANTHROPIC_API_KEY`, zero errors, and the LLM-dependent cells
   produce sensible output.
4. `docs/` still matches what the notebook does.
5. Runtime has not quietly ballooned (target: under ~3 minutes offline).

Then `testing → main`, tagged if the change is substantial.

Hotfixes to something broken on `main` may branch from `main` and merge back to both `main` and `dev`.
