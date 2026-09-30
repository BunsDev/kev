---
language: en
license: apache-2.0
library_name: peft
base_model: Qwen/Qwen3.5-9B-Base
base_model_relation: adapter
pipeline_tag: text-classification
tags:
  - decision-model
  - calibration
  - lora
  - multiple-choice
  - typesafe
  - qwen3.5
datasets:
  - legacy-datasets/banking77
  - google/boolq
  - fancyzhx/ag_news
  - nyu-mll/multi_nli
  - SetFit/sst5
  - Yelp/yelp_review_full
  - CogComp/trec
  - fancyzhx/dbpedia_14
  - SetFit/amazon_reviews_multi_en
  - stanfordnlp/imdb
  - bigcode/commitpackft
  - davidheineman/consumer-finance-complaints-large
metrics:
  - accuracy
  - brier_score
  - expected_calibration_error
model-index:
  - name: Kev-9B
    results:
      - task: { type: text-classification, name: typed decision, out-of-domain, locked test }
        dataset: { type: mixed, name: "transfer-v4 test (locked)" }
        metrics:
          - { type: accuracy, value: 0.852 }
          - { type: brier_score, value: 0.199 }
      - task: { type: text-classification, name: long policy documents, test }
        dataset: { type: mixed, name: "hard-v1 test (1,088 questions)" }
        metrics:
          - { type: accuracy, value: 0.834 }
      - task: { type: text-classification, name: developer-tool decisions, test }
        dataset: { type: mixed, name: "devtools-v1 test (1,071 questions)" }
        metrics:
          - { type: accuracy, value: 0.791 }
      - task: { type: text-classification, name: real complaints, test }
        dataset: { type: mixed, name: "documents-v1 test (936 questions)" }
        metrics:
          - { type: accuracy, value: 0.900 }
---

# Kev-9B

This is **Kev-9B v2**, released 2026-09-30 on the `main` branch of `jaredpalmer/kev-9b`.

> **Previous version.** Kev-9B v1 (trial `night2-9b-du/00-trial-0`, T 2.30) stays on the Hub at
> [`jaredpalmer/kev-9b@v1`](https://huggingface.co/jaredpalmer/kev-9b/tree/v1). Its headline numbers: locked transfer-v4 test **0.852**
> with served Brier **0.224**, hard-v1 test 0.584, devtools-v1 test 0.637, documents-v1 test 0.829. Its card is the README at that tag
> (and `docs/model-cards/kev-9b.md` in the [Kev repository](https://github.com/jaredpalmer/kev) before 2026-09-30). Below, "Kev-9B v1" is that checkpoint; every comparison is against it.

Kev-9B v2 is a **decision model**: one document (the *state*) and a set of typed questions in, a probability distribution per question out, in one forward pass. No text generation. It serves TypeSafe's public `/v1/systemone` contract, like every Kev. It is a rank-16 LoRA adapter (45.4M trainable parameters) plus a pointer head on `Qwen/Qwen3.5-9B-Base` (revision `68c46c4b`), made in one step on top of v1:

1. **Documents and skills delta.** Kev-9B v1 was trained for one more epoch on round 15's joint data: documents-v1 train (real CFPB complaint narratives) and round 10's skills data (hard-v1 and devtools-v1 train), 16,539 records, with 10,000 records replayed from `decision-v7` (round 18, arm (a)). It is the recipe that confirmed Kev-0.8B in round 15, and the data that Kev-4B has had since round 10.

It is served at temperature **2.19**. That value was fitted on held-out datasets it never trained on.

**What it is better at than Kev-9B v1.** The comparisons below were registered before the confirmation reads, and every paired interval is a 95 % record-clustered bootstrap.
- Long policy documents (hard-v1 test): **+25.0 pp [+22.0, +28.1]**.
- Developer-tool decisions (devtools-v1 test, registered sources): **+9.9 pp [+7.2, +12.5]**.
- Real complaints (documents-v1 test): **+7.1 pp [+4.7, +9.2]**; the private held-out documents-v2, **+8.0 pp [+5.9, +10.2]**.
- On the locked out-of-domain test it is level: **0.852** against 0.852 (+0.0 pp [−1.7, +1.8]), with served Brier **0.199** against 0.224 (−0.025 [−0.047, −0.007]).

**Read this first.**
- **The selection was not blind.** This delta was trained and read on development in round 18 (2026-09-24), where it passed both primaries and failed only WANLI-v2, scienthoon and the pooled externals. All three were later removed or ungated as unsound gates. Round 27 re-selected round 18's two finished arms on the audited rule, knowing their development numbers. The test partitions and the locked read, each read once for this checkpoint, are the guard (`PLAN.md`, "Round 27 (registered)").
- **Its large gains are in distribution.** hard-v1, devtools-v1 and documents-v1 train partitions are in its training data. Those gains are held-out items and templates of trained families, not transfer. On datasets it never trained on it is level with v1 (breadth-v1 below).
- **It is not better than v1 on new work.** breadth-v1 test +0.6 pp [−0.4, +1.6], transfer-v4 development −0.2 pp [−2.4, +1.8], transfer-r3 test +0.0 pp [−1.2, +1.2].
- **Its temperature was written, not refitted.** `scripts/calibrate_checkpoint.py` cannot list the sources of `evals/round15/joint`, so it refuses to fit. The value in `head.pt` is round 27's own pool fit, written with `--temperature` and the reason recorded, after `kev.rounds validate` had checked the pool against the arm's training. That check does not cover `evals/night2`, v1's own dates / unknowable delta, whose manifest lists no sources. Its records are four synthetic families, none of them in the pool.

## Confirmation (registered, round 27)

Round 27 had two candidates, round 18's arms at lr 2e-5 and 1e-5. Both passed the development rule, and this checkpoint (`9b-r18a`, lr 2e-5) ranked first. It was then confirmed with round 18's stages, one read each ([`experiments/rounds/r27.json`](https://github.com/jaredpalmer/kev/blob/main/experiments/rounds/r27.json); verdicts in `runs/r27-verdict/`).
- Every delta is paired against Kev-9B v1, in percentage points.
- v2 is served at its pool temperature 2.19 and v1 at its shipped 2.30.
- devtools-v1 is without `flakeflagger` and `commitpackft_type` in every panel, as the 2026-09-27 audit registered; the same rows leave both sides.

| stage / criterion (panel, questions) | Kev-9B v2 | Kev-9B v1 | Δ [95 %] | verdict |
|---|---|---|---|---|
| development: hard-v1 + devtools-v1 accuracy, lower bound > 0 (1,855) | 0.821 | 0.628 | +19.3 [+17.2, +21.6] | pass |
| development: documents-v1 accuracy, lower bound > 0 (920) | 0.902 | 0.833 | +7.0 [+4.8, +9.2] | pass |
| development: short states (transfer-v4 dev + transfer-r3 test, without `emotion`), accuracy lower ≥ −2 (1,586) | 0.871 | 0.871 | +0.1 [−1.1, +1.2] | pass |
| development: short-state Brier upper ≤ +0.02 / confident errors upper ≤ +1 pp | 0.197 / 2.3 % | 0.197 / 3.7 % | −0.000 / −1.3 pp | pass |
| tests: hard-v1 + devtools-v1 test accuracy, lower bound > 0 (1,859) | 0.822 | 0.635 | +18.7 [+16.7, +20.8] | pass |
| tests: documents-v1 test accuracy, lower bound > 0 (936) | 0.900 | 0.829 | +7.1 [+4.7, +9.2] | pass |
| **locked: transfer-v4 locked accuracy ≥ v1 − 1 pp (656)** | **0.852** | 0.852 | +0.0 [−1.7, +1.8] | pass |
| locked: served Brier ≤ v1 + 0.005 (656) | 0.199 | 0.224 | −0.025 [−0.047, −0.007] | pass |

The development rule also held unknowable answers at ≥ 0.9 confidence under 5 % (0 %) and hard-v1 ECE within 0.01 of v1's (0.050 against 0.073).

Report-only test reads:

| panel (questions) | Kev-9B v2 | Kev-9B v1 | Δ [95 %] |
|---|---|---|---|
| hard-v1 test (1,088) | 0.834 | 0.584 | +25.0 [+22.0, +28.1] |
| devtools-v1 test, all sources (1,071) | 0.791 | 0.637 | +15.4 [+11.0, +19.4] |
| documents-v2, private held-out test (953) | 0.900 | 0.821 | +8.0 [+5.9, +10.2] |
| breadth-v1 test, all 14 datasets (3,089) | 0.698 | 0.692 | +0.6 [−0.4, +1.6] |

## Breadth index on test

This is the chance-corrected index of the community Decision Index, computed over breadth-v1's 14 held-out datasets in five areas (`scripts/breadth_report.py`).

| | Kev-9B v2 | Kev-9B v1 | Kev-4B | Kev-27B |
|---|---|---|---|---|
| index (95 % CI) | **41.0** [38.8, 43.9] | 40.0 [38.1, 43.0] | 38.0 [35.5, 41.3] | 52.3 [49.2, 55.4] |

Against Kev-9B v1 the index is +1.0 [−0.9, +2.5]: level.

## Results as served (whole suites)

The table below uses whole suites (minus two duplicated CodeReviewer ids), with no exclusions. Each model is at its own served temperature. Numbers come from `runs/release/kev-9b-r27.json` (`scripts/release_numbers.py --release kev-9b-r27`).

| | **Kev-9B v2 (T = 2.19)** | Kev-9B v1 (T = 2.30) |
|---|---|---|
| **locked test**, out-of-domain accuracy / Brier (transfer-v4) | **0.852 / 0.199** | 0.852 / 0.224 |
| locked test, ECE / confident errors (p ≥ 0.9 and wrong) | **0.034 / 1.4 %** | 0.042 / 3.2 % |
| locked test, coverage at ≤ 5 % error | **0.742** | 0.645 |
| out-of-domain accuracy / Brier (transfer-v4 development) | 0.820 / 0.262 | 0.822 / 0.264 |
| short states, transfer-r3 test (`emotion` included) | 0.847 | 0.847 |
| MMLU-Pro (transfer-v9 development, 10-way) | 0.590 | 0.515 |
| unknowable items answered at ≥ 0.9 (lower is better) | 0.00 | 0.00 |
| breadth-v1 development / test, all 14 datasets | 0.700 / 0.698 | 0.697 / 0.692 |
| hard-v1 development / test | 0.813 / 0.834 | 0.574 / 0.584 |
| devtools-v1 development / test, all sources | 0.772 / 0.791 | 0.631 / 0.637 |
| documents-v1 development / test (CFPB complaints) | 0.902 / 0.900 | 0.833 / 0.829 |
| documents-v2 (private held-out test) | 0.900 | 0.821 |
| SemIf (144 authored decisions) | 0.917 | 0.910 |

## Calibration

`head.pt` carries temperature **2.19** (2.1936), round 27's registered pool fit:
- transfer-r3's calibration partition, eight held-out public sources, 448 questions;
- transfer-v9 development MMLU-Pro, 200 questions;
- minus any transfer-v4 development duplicates (none);
- 648 questions in all.

The pooled T has a 90 % bootstrap interval of [2.05, 2.41]. On the pool, ECE goes from 0.125 raw to 0.041; on transfer-v4 development, from 0.103 to 0.041. `KEV_TEMPERATURE=1.0` gives the raw logits.

Kev-9B v1's 2.30 was fitted on its own decision-v7 development rows, held-out items of what it trained on. The standing rules no longer allow that, and v2 is the first 9B served at a held-out-pool temperature. As served, test-partition ECE against Kev-9B v1:
- breadth-v1, all 14 datasets: 0.034 against 0.044;
- hard-v1: 0.054 against 0.075;
- devtools-v1: 0.098 against 0.147;
- documents-v1: 0.017 against 0.103.

A read of the staged checkpoint on semif-v1 (`runs/rel9-staged-semif`) reproduces round 18's rows at T 2.19: equal logits over T, 0 flips.

## How it was built

- **Base model**: `Qwen/Qwen3.5-9B-Base` (revision `68c46c4b`, Apache-2.0). It is a hybrid of 24 Gated DeltaNet and 8 full-attention layers, so questions run as separate causal rows continuing from the shared state (`kev/model.py`).
- **v1** (`jaredpalmer/kev-9b@v1`): the `decision-v7` recipe (two epochs, lr 5e-5) plus the dates / unknowable delta (`evals/night2`, one epoch, lr 2e-5). Its card describes both.
- **Delta** (round 18, `experiments/round18/joint.json`, arm (a)):
  - `kev.train --init_from jaredpalmer/kev-9b --data evals/round15/joint/train.jsonl --replay 10000`, one epoch;
  - learning rate 2e-5, LoRA r=16 on attention, MLP and DeltaNet projections, batch 2 with accumulation 4;
  - states of at most 7,552 tokens, none-of-the-above pairs on 25 % of Choice records, bf16 autocast, gradient checkpointing;
  - seed 1, one H200, 2.7 hours.
- **Data**:
  - documents-v1 train: CFPB complaint narratives with product and issue labels from open-weight teachers, filtered by closed-model judges and adjudication.
  - hard-v1 train: programmatically labelled skill records in seven families (long policy documents, trade-offs, probability, multi-hop, temporal / numeric, judging a proposed answer, missing-fact abstention), templates 0-3; evaluation uses templates 4 and 5.
  - devtools-v1 train: developer-tooling decisions from licence-checked sources, with human, heuristic or by-construction labels, never an LLM's.
  - No Jev outputs were used.

## Known limits

- Its gains are on the families it trained on; on held-out datasets it is level with v1, and Kev-27B leads it by 11 points on the breadth index.
- Knowledge is set by the base: MMLU-Pro 0.590, against Kev-27B's 0.675.
- Trained on states of at most 7,552 tokens. Longer documents are served, up to 65,536 tokens, but not what it was trained for.
- devtools-v1 is its least calibrated suite (test ECE 0.098).
- Its temperature has a 90 % interval of [2.05, 2.41].
- The DeltaNet kernels have no MPS implementation. On Apple Silicon `kev.serve` runs this checkpoint through MLX (`kev/mlx_model.py`); on CUDA with `flash-linear-attention` it answers in tens of milliseconds.

## Intended use

Kev-9B v2 is meant for typed decisions over documents of a few thousand tokens: classification, routing, extraction choices and policy checks, served behind TypeSafe's System One contract, where calibrated probabilities feed thresholds and review queues. It fits a single L40S or H100.
- Freeze thresholds on your own labelled workload, and refit the temperature on it if it differs from what Kev trained on (`kev.calibrate`).
- It is not a generative model or a chat model. It does not replace a human decision where errors are costly.

## Use

```bash
uv run --extra serve python -m kev.serve --run jaredpalmer/kev-9b --port 8008      # CUDA, bf16 + fused kernels + CUDA graphs; MLX on Apple Silicon
uv run --extra serve python -m kev.serve --run jaredpalmer/kev-9b@v1 --port 8008   # the previous version
```

Any TypeSafe-compatible client works: `TypeSafeClient(api_key="local", base_url="http://127.0.0.1:8008", model="kev-latest")`.

## License

Apache-2.0 for the adapter and head. The Qwen3.5-9B base is Apache-2.0. The training data carries its own licences, recorded per source in the manifests.
