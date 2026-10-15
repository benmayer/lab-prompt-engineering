# Failure Analysis & v1 → v3 Comparison

Source of truth: `prompt_engineering_lab.ipynb` (tables are generated there; raw responses in `results/raw_responses.json`, summary in `results/summary.csv`).
Model `gpt-4o-mini`, temperature 1.0 unless noted. **Consistency** is defined per task in the notebook header: modal share of identical normalised answers (sentiment), modal share of identical parsed JSON (extraction), and share of runs passing one fixed 5-check spec (product description, since free text is never byte-identical).

## 1. v1 failure analysis (5 / 10 / 15 runs)

| Task | Consistency 5 / 10 / 15 | Format valid (15) | Correct (15) | Unique outputs (15) |
|---|---|---|---|---|
| Sentiment | 20% / 60% / 27% | 0% | 100% | 10 |
| Product description | 0% / 0% / 0% | 0% | 0% | 15 |
| Data extraction | 20% / 10% / 20% | 0% | 0% | 13 |

### Documented failure patterns

**Sentiment v1**
- 0/15 answers are a bare label; every answer is a sentence ("The customer message can be classified as **Positive Feedback**").
- Hedges with a second category in most runs ("... or **Customer Satisfaction**"); capitalisation and markdown differ run to run.
- The easy base message hides a deeper failure: on the Step 12 messages v1 often does not give a sentiment at all ("complaint", "Order Update", "mixed feedback") → 0% correct on 4 of 5 variations.
- 5 → 10 runs *looked* better (20% → 60%) then fell back (27% at 15): small batches are noise.

**Product description v1**
- 220-347 words (spec: 40-100). 0/15 pass any single length/bullet/markdown check except price presence.
- Bold markdown and headings in every run; 5-6 feature bullets; exclamation marks and "Order yours today!" calls to action.
- Invents brand names (SwiftConnect, SwiftClick, SmoothScroll) and specifications (33 ft range, DPI presets); 11 distinct opening lines in 15 runs.

**Data extraction v1**
- 0/15 valid JSON; 100% of runs contain the right facts but as markdown bullets (12 with bold) or prose, sometimes with a preamble.
- Key labels change between runs ("Delivery Speed" vs "Issue"; "Packaging Condition"), and values are not normalised (`#12345`, `March 15th`) - unusable for a program without a cleaning step.

## 2. All versions on the base inputs (fresh 15-run batches)

| Task | v1 | v2 | v3 (final) | v1 → v3 |
|---|---|---|---|---|
| Sentiment | 33% | 100% | **100%** | +67 pts |
| Product description | 0% | 100% | **93%** (v3.1: 100%) | +93 pts |
| Data extraction | 33% | 73% | **100%** | +67 pts |

Success criterion (>80% for v3 on all tasks): **met**. See `results/consistency_comparison.png`.

Residual v2 failure (extraction): 3/15 (and 4/15 in the fresh batch) responses wrapped the JSON in a ```` ```json ```` code fence, so `json.loads` failed even though the content was correct. Chain-of-Thought with a `FINAL JSON:` delimiter fixed it.

## 3. Variations (10 runs each; v1 vs v3)

| Task / case | v1 consistency | v1 correct | v3 consistency | v3 correct |
|---|---|---|---|---|
| Sentiment: negative | 50% | 0% | 100% | 100% |
| Sentiment: neutral statement | 10% | 0% | 100% | 100% |
| Sentiment: mixed review | 10% | 0% | 100% | 100% |
| Sentiment: sarcasm | 10% | 70% | 100% | 100% |
| Sentiment: question | 30% | 0% | 100% | 100% |
| Product: water bottle | 0% | 0% | 100% | 100% |
| Product: headphones | 0% | 0% | 100% | 100% |
| Extraction: late + wet box | 10% | 0% | 100% | 100% |
| Extraction: missing fields | 10% | 0% | **50%** | **50%** |

**Where v3 fell short.** Extraction `missing_fields`: the customer never mentions delivery, but 5/10 runs invented a `delivery_sentiment` (negative ×3, neutral ×2), projecting "nobody answered my emails" onto delivery.
**v3.1 fix** (explicit scope rule: service complaints → `null`): 15/15 correct on that case, and no regression on the other two extraction cases. For product, longer examples plus a 55-75 word target removed the near-floor length misses: 45/45 spec-pass across three products.
Caveat: v3.1 was tuned on the case it was diagnosed on, so that result is in-sample.

## 4. Temperature (Bonus 1, 10 runs per cell)

| Task | Version | T=0 | T=0.3 | T=0.7 | T=1.0 |
|---|---|---|---|---|---|
| Sentiment | v1 | 50% | 50% | 60% | 40% |
| Sentiment | v3 | 100% | 100% | 100% | 100% |
| Product | v1 | 0% | 0% | 0% | 0% |
| Product | v3 | 100% | 100% | 100% | 100% |
| Extraction | v1 | 90% | 100% | 50% | 30% |
| Extraction | v3 | 100% | 100% | 100% | 100% |

v1 extraction at T=0.3 is 100% *consistent* yet 0% valid JSON - repeatable but wrong. Lowering temperature cannot repair a prompt that never specified the output.

## 5. Limitations

- Small samples (10-15 runs per cell) and one model; a single 15-run batch can swing by ±10-20 points (see the two v1 and v2 extraction batches), so treat differences of that size as noise.
- Product "consistency" is conformance to a spec I wrote, applied to all versions. v1 scores 0% partly because it was never asked for that spec; the metric measures reliability against requirements, not copy quality.
- Ground-truth labels (e.g. mixed review → `neutral`) are policy choices I made; a different client policy would change the prompt and the labels.
