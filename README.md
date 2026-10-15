# Prompt Engineering Lab

**Track:** Module 2 — Prompt engineering · **When:** Week 2, Day 5 · **Status:** Required

This repository contains everything you need for this lab.

## Files

| File / folder | What it is |
|---|---|
| [`instructions.md`](./instructions.md) | the lab instructions |
| [`rubric.md`](./rubric.md) | how the submission is graded |
| [`prompt_engineering_lab.ipynb`](./prompt_engineering_lab.ipynb) | **the lab**: setup, v1/v2/v3 prompts, 5/10/15-run tests, failure analysis, variations, final evaluation, temperature bonus (executed, outputs included) |
| [`failure_analysis.md`](./failure_analysis.md) | failure patterns, v1→v3 comparison tables, variations, limitations |
| [`improved_prompts.md`](./improved_prompts.md) | final prompts for all three tasks, why each change was made, which technique worked best |
| [`lab_summary.md`](./lab_summary.md) | the single-paragraph summary |
| `results/summary.csv`, `results/raw_responses.json` | every experiment's metrics and raw model outputs |
| `results/consistency_comparison.png` | v1/v2/v3 consistency chart |
| `results/cache/` | cached API responses so the notebook reproduces the exact same numbers |
| `requirements.txt` | Python dependencies |

## How to run

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
echo 'OPENAI_API_KEY=sk-...' > .env      # .env is gitignored
jupyter nbconvert --to notebook --execute --inplace prompt_engineering_lab.ipynb
```

By default the notebook replays cached responses from `results/cache/` (no API calls, identical numbers).
To sample live from the API, set `USE_CACHE = False` in the setup cell (about 825 calls to `gpt-4o-mini`; a few minutes and well under $1).

## How to complete this lab

1. **Fork** this repository.
2. Do the work described in `instructions.md`, committing to your fork.
3. Open a **pull request** back into this repository.
4. You'll receive **AI feedback** on your PR based on `rubric.md`. Address any blocking feedback and push updates to the same PR.

## Submission hygiene

- Keep this repository scoped to this lab only — no unrelated projects or personal files.
- Use clear, descriptive filenames.
- Remove secrets, API keys, and tokens before committing.
