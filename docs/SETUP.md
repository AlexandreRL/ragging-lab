# Setup

Everything here gets you to the same place: the lab notebook running top to bottom, offline, in about
90 seconds. If you hit something not covered, [open an issue](https://github.com/AlexandreRL/ragging-lab/issues) —
setup friction is a documentation bug.

---

## What you need

| | Requirement | Notes |
|---|---|---|
| Python | **3.12** recommended, 3.10+ works | `py -0p` on Windows lists what you have |
| Disk | ~3 GB | Most of it is the CPU PyTorch wheel (~800 MB installed) + 180 MB of models |
| RAM | 4 GB free | The §7 synthetic corpus is 12 000 × 128 floats = 6 MB; nothing here is heavy |
| GPU | **not needed** | CPU by design |
| API key | **not needed** | Optional Claude key switches on 4 cells |
| Editor | VS Code (recommended) or JupyterLab | Both covered below |

---

## 1. Create the virtual environment

A project-local `.venv` keeps the lab's pinned versions away from your system Python.

**Windows (PowerShell)**

```powershell
cd ragging-lab
py -3.12 -m venv .venv
.venv\Scripts\python -m pip install --upgrade pip setuptools wheel
```

**macOS / Linux**

```bash
cd ragging-lab
python3.12 -m venv .venv
.venv/bin/python -m pip install --upgrade pip setuptools wheel
```

> You do **not** need to "activate" the venv for any command in this guide — calling
> `.venv\Scripts\python` (or `.venv/bin/python`) directly is unambiguous and avoids a whole class of
> "wrong interpreter" confusion. Activate it if you prefer; both work.

---

## 2. Install PyTorch (CPU wheel) first, on Windows

The default PyPI `torch` wheel can pull CUDA payloads you will never use here. Install from the CPU index
instead:

```powershell
.venv\Scripts\python -m pip install torch --index-url https://download.pytorch.org/whl/cpu
```

On macOS, plain `pip install torch` is already CPU/MPS and this step is unnecessary — `requirements.txt`
covers it.

---

## 3. Install the rest

```powershell
.venv\Scripts\python -m pip install -r requirements.txt
```

This brings in JupyterLab, numpy/pandas/matplotlib/scikit-learn/scipy, `rank_bm25`, `faiss-cpu`,
`sentence-transformers`, and the `anthropic` SDK.

---

## 4. Register the Jupyter kernel

So the notebook can find this exact interpreter by name:

```powershell
.venv\Scripts\python -m ipykernel install --user --name ragging-lab --display-name "Python (ragging-lab)"
```

Verify:

```powershell
.venv\Scripts\python -m jupyter kernelspec list
```

---

## 5. Open the notebook

### VS Code (recommended)

1. Install the **Python** and **Jupyter** extensions. VS Code will offer both when you open the folder —
   they are listed in `.vscode/extensions.json`. From the command line:
   ```powershell
   code --install-extension ms-python.python --force
   code --install-extension ms-toolsai.jupyter --force
   ```
2. `code .` in the repo root.
3. Open `retrieval_eval_and_vector_search_internals.ipynb`.
4. Click the kernel picker, top right → **Python (ragging-lab)**. (`.vscode/settings.json` already points
   the interpreter at `.venv`, so it is usually pre-selected.)
5. **Run All.** Expect ~90 seconds; the slowest single cell is §7.5, the pure-Python HNSW build (~12-15 s).

### JupyterLab

```powershell
.venv\Scripts\python -m jupyter lab
```

Then pick the **Python (ragging-lab)** kernel from the launcher.

---

## 6. Optional: switch on the Claude cells

Four cells use an LLM — synthetic query generation (§3.4), listwise reranking (§8.2), grounded generation
and the LLM-as-judge (§9.3–9.4). Without a key they run deterministic fallbacks and say so.

```powershell
copy .env.example .env
# then edit .env and set ANTHROPIC_API_KEY=sk-ant-...
```

`.env` is git-ignored. Alternatives, in the order the SDK resolves them:

1. `ANTHROPIC_API_KEY` in the environment
2. `ANTHROPIC_AUTH_TOKEN`
3. a profile from `ant auth login` (the notebook's bare client constructor picks this up automatically)
4. the `.env` file next to the notebook (read by the notebook itself, not by the SDK)

Or set `CFG.prompt_for_key = True` in cell 1.1 and re-run cell 1.2 for a one-off `getpass` prompt.

**To force offline mode** even with a key present: `CFG.use_claude = False`.

**What it costs.** The four cells together are a handful of short calls — fractions of a cent at Claude
Opus 5 list prices. Cell 1.2 tracks tokens and prints a running estimate, and `CFG.claude_judge_model`
lets you point the judge and reranker at a cheaper model (`claude-sonnet-5`, `claude-haiku-4-5`).

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `ImportError: DLL load failed while importing _regex` | **Windows Smart App Control** blocks the newest `regex` wheel's unsigned `.pyd`, which breaks `import transformers` | `pip install regex==2025.11.3` (pinned in `requirements.txt` for win32 — it satisfies `transformers>=2025.10.22` *and* loads). See the note below. |
| `Failed to build installable wheels for hnswlib` | No MSVC compiler | Not needed — the lab implements HNSW from scratch and uses FAISS for the library comparison. Ignore it. |
| Kernel not listed in VS Code | Kernel never registered, or VS Code cached the list | Re-run step 4, then **Developer: Reload Window** |
| `ModuleNotFoundError` for a package you installed | Notebook is on a different interpreter | Check the kernel picker says **Python (ragging-lab)**; run `import sys; print(sys.executable)` in a cell |
| First run stalls for a minute at §1.3 | Downloading `all-MiniLM-L6-v2` (~90 MB) and the cross-encoder (~90 MB) from Hugging Face | One-off; they cache under `~/.cache/huggingface`. Subsequent runs are instant. |
| `Warning: You are sending unauthenticated requests to the HF Hub` | No HF token | Harmless. Set `HF_TOKEN` in `.env` to silence it. |
| Warning about symlinks in the HF cache (Windows) | Developer Mode off | Harmless — the cache just uses more disk |
| `[OFFLINE] ... no credentials found` | No API key, as designed | Nothing to fix unless you want the LLM cells; see step 6 |
| `[OFFLINE] ... credentials rejected` | Key is revoked, out of credit, or scoped to another workspace | Check it at [console.anthropic.com](https://console.anthropic.com/settings/keys) |
| `WARNING clustering N points to 256 centroids` from FAISS | Fewer training points than FAISS wants for a PQ codebook | Expected at `CFG.ann_n` defaults, and a genuinely useful production signal. Raise `CFG.ann_n` to silence it. |
| §7.5 takes minutes, not seconds | You raised `CFG.ann_n` | Expected — pure-Python HNSW build is O(n log n) with a big constant. Use the FAISS cell (§7.8) at scale. |
| Charts render tiny or not at all | Matplotlib backend | Add `%matplotlib inline` at the top of cell 1.1 |

### More on the Smart App Control / `regex` trap

Windows 11's Smart App Control (and WDAC in "Verified and Reputable" mode) blocks binaries it does not
consider reputable. Several recent `regex` releases ship a `_regex.cp312-win_amd64.pyd` that trips this,
and because `transformers` imports `regex` at module load, the failure surfaces as a `transformers` import
error rather than anything that mentions App Control.

Check whether it is on:

```powershell
Get-MpComputerStatus | Select-Object SmartAppControlState
```

Options, best first:

1. **Use the pinned `regex==2025.11.3`** (already in `requirements.txt`). It loads and satisfies
   `transformers`.
2. Try other versions until one loads — `2024.11.6` also loads but is *too old* for `transformers 5.x`.
3. Turning Smart App Control off is **irreversible without reinstalling Windows**. Do not do it for this.

---

## Verifying your install

```powershell
.venv\Scripts\python -c "import numpy, pandas, sklearn, scipy, faiss, rank_bm25, anthropic; import torch, transformers, sentence_transformers; print('all imports OK')"
```

Then, the real test — execute the notebook headlessly and fail on any cell error:

```powershell
.venv\Scripts\python -m jupyter nbconvert --to notebook --execute --inplace `
    --ExecutePreprocessor.timeout=1800 retrieval_eval_and_vector_search_internals.ipynb
```

That is exactly the check a PR is expected to pass.

---

## Updating

```powershell
git pull
.venv\Scripts\python -m pip install -r requirements.txt --upgrade
```

If an upgrade breaks something, please [report it](https://github.com/AlexandreRL/ragging-lab/issues) with
your `pip list` — version drift in this stack is real, and the repo's job is to absorb it.
