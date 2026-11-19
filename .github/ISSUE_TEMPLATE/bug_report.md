---
name: "🐛 Bug / a cell fails"
about: The notebook errors, or a number looks wrong
title: "[bug] "
labels: ["bug"]
---

### Which cell

Section and cell number:

### What happened

<!-- Paste the error name, message and the last ~10 traceback lines. -->

```
```

### Environment

- OS and version:
- Python version (`python -V`):
- Running with an API key? yes / no
- Relevant package versions:

```
# .venv\Scripts\python -m pip list | findstr /i "numpy pandas torch faiss transformers sentence-transformers anthropic"
```

### Already checked

- [ ] I looked at [docs/SETUP.md § Troubleshooting](../../docs/SETUP.md#troubleshooting)
- [ ] It reproduces from a fresh kernel (`Restart and Run All`)
