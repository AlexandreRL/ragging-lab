## What this changes

<!-- One or two sentences. If the notebook diff is unreviewable (it often is - we commit outputs on
     purpose), this description is what gets reviewed. Say what changed and where. -->

## Why

<!-- The decision this helps someone make, or the confusion it removes. -->

## Type

- [ ] Typo / wording / link fix
- [ ] New or improved explanation
- [ ] New labelled queries (§3.2)
- [ ] New metric (§4)
- [ ] New retriever, index or reranker (§5, §7, §8)
- [ ] New exercise (§10.2)
- [ ] Setup / docs / tooling
- [ ] Something else:

## Checklist

- [ ] The notebook runs **top to bottom from a fresh kernel with no API key**, zero cell errors
- [ ] Committed outputs are from my final run (`Restart and Run All`, then save)
- [ ] New metric or algorithm ships with an assert that would fail if it were wrong
- [ ] Any "better / faster / improved" claim is backed by printed output in the same cell
- [ ] Randomness is seeded from `CFG.seed`
- [ ] Charts come with their DataFrame, and use `SERIES[...]` in fixed order
- [ ] `docs/` updated if setup, structure or sections changed
- [ ] Targeted at **`dev`** (see [docs/BRANCHING.md](../docs/BRANCHING.md))

## Measurements, if this changes behaviour

<!-- Paste the before/after table and the paired-bootstrap verdict. Negative and null results are
     welcome - they are findings too, and the repo prefers an honest null to a flattering maybe. -->

```
```

## Anything you would like help with?

<!-- Totally fine to open a PR that is 80% done and ask. Say so here. -->
