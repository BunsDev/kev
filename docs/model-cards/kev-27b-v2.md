---
language: en
license: apache-2.0
library_name: transformers
base_model: Qwen/Qwen3.8-27B
base_model_relation: finetune
pipeline_tag: text-classification
tags:
  - decision-model
  - calibration
  - full-weight-sft
  - weight-averaging
  - multiple-choice
  - typesafe
  - qwen3.8
  - release-candidate
datasets:
  - deepmind/aqua_rat
  - allenai/ai2_arc
  - coastalcph/lex_glue
  - allenai/cosmos_qa
  - tau/commonsense_qa
  - tasksource/esci
  - openai/gsm8k
  - nvidia/HelpSteer2
  - nvidia/HelpSteer3
  - hotpotqa/hotpot_qa
  - AmazonScience/massive
  - allenai/math_qa
  - openlifescienceai/medmcqa
  - pfb30/multi_woz_v22
  - sentence-transformers/natural-questions
  - allenai/openbookqa
  - google-research-datasets/poem_sentiment
  - allenai/qasc
  - allenai/quartz
  - allenai/social_i_qa
  - stanfordnlp/snli
  - ChilleD/StrategyQA
  - allenai/winogrande
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
  - name: Kev-27B v2 (release candidate)
    results:
      - task: { type: text-classification, name: typed decision, out-of-domain, locked test }
        dataset: { type: mixed, name: "transfer-v4 test (locked)" }
        metrics:
          - { type: accuracy, value: 0.8887 }
          - { type: brier_score, value: 0.154 }
      - task: { type: text-classification, name: held-out datasets, test }
        dataset: { type: mixed, name: "breadth-v1 test (14 held-out datasets, 4 excluded as registered)" }
        metrics:
          - { type: accuracy, value: 0.832 }
      - task: { type: text-classification, name: held-out task families, test }
        dataset: { type: mixed, name: "tasksource-heldout-v1 test (24 held-out families, 7 excluded as registered)" }
        metrics:
          - { type: accuracy, value: 0.795 }
---

# Kev-27B v2 (release candidate)

> **Private release candidate, awaiting approval.** This checkpoint is not released. It lives in the private repository
> `jaredpalmer/kev-27b-v2-candidate` for review. The released 27B model is still [`jaredpalmer/kev-27b`](https://huggingface.co/jaredpalmer/kev-27b).
> Whether this candidate replaces it is Jared's decision.

Kev-27B v2 is a **decision model**: one document (the *state*) and a set of typed questions in, a probability distribution per question out, in one forward pass. No text generation. It serves TypeSafe's public `/v1/systemone` contract, like every Kev. It is a full-weight checkpoint of `Qwen/Qwen3.8-27B` (revision `1d4bf0f2`) plus a pointer head, and it is made in two steps:

1. **Full-weight SFT.** Every backbone weight of Qwen3.8-27B was fine-tuned for one epoch on `sft-v2-r22`, a private corpus of 145,840 decision records (round 22).
2. **Blend toward Kev-27B.** The SFT backbone was averaged with Kev-27B's backbone: 0.85 × SFT + 0.15 × Kev-27B, where Kev-27B's LoRA adapter was merged in fp32. The SFT's pointer head was kept (round 23).

It is served at temperature **1.32**. That value was fitted on held-out datasets that neither parent trained on.

**What it is better at than Kev-27B.** The comparisons below were registered before the reads, and every paired interval is a 95 % record-clustered bootstrap.
- Held-out datasets (breadth-v1 test): **+1.2 pp [+0.3, +2.2]**.
- Held-out task families (tasksource-heldout-v1 test): **+5.3 pp [+3.7, +6.8]**.
- Kev's skill, developer-tooling and real-document suites together (hard-v1, devtools-v1 and documents-v1 test): **+8.9 pp [+7.5, +10.3]**.
- On the locked out-of-domain test it reaches **0.8887** accuracy with served Brier **0.154**. The bar was 0.886. Kev-27B scores 0.8963 / 0.160.

**Read this first.**
- **The confirmation is weaker evidence than a fresh one.** Round 23 was designed after round 24's confirmation had been read. Round 24's candidate (the unblended SFT) missed the locked bar and was worse on long contracts, and the blend was chosen as a response. Round 23 was then confirmed on the same test partitions and the same locked transfer-v4 that round 24 had read. Its bars were fixed before any round-23 read, one candidate was confirmed, and each read was made once. Even so, these partitions were no longer untouched for this question (`PLAN.md`, "Round 23 (registered)", "Reuse of the test partitions").
- **It is not better than Kev-27B on short states.** Locked transfer-v4: −0.8 pp [−2.0, +0.5] against Kev-27B. On the registered short-state panel (without `emotion`) it is −0.9 pp [−2.0, +0.1]. On the whole transfer-r3 test panel, `emotion` included, it is −2.1 pp [−3.5, −0.8].
- **It is worse than Kev-27B on long contracts.** On the CUAD contracts of longdoc-v1 test it scores 0.874 against 0.890 (−1.6 pp [−3.0, −0.3]), and its ECE is 0.053 against 0.007. In every length bucket up to 64k tokens its ECE is 2 to 4 times Kev-27B's. On development its accuracy was level (+0.8 pp [−0.2, +1.8]), but its ECE was already 0.097 against 0.063. The blend barely moved this: round 24's unblended SFT was −1.8 pp with ECE 0.055 on the same test.
- **Several headline suites are in distribution.** hard-v1, devtools-v1 and documents-v1 train partitions are in its training data, and the ood / agents / guardrails suites share generators with its training components. Gains there are held-out items of trained families, not transfer.
- **The base is post-trained.** `Qwen/Qwen3.8-27B` is Qwen's instruction-tuned release, and what Qwen trained it on is not known to us.
- **It needs a data-centre GPU.** The bf16 weights are 51 GB (65.5 GB resident when served); one B200, H200 or H100 80 GB. There is no Mac path.

## Confirmation (registered, round 23)

Round 23 had six candidate blends. Five passed the development rule, and this checkpoint (`27b-k-w85`) ranked first. It was then confirmed with round 24's stages, identical bars, one read each ([`experiments/rounds/r23.json`](https://github.com/jaredpalmer/kev/blob/main/experiments/rounds/r23.json); verdicts in `runs/r23-verdict/`).
- Every delta is paired against the released Kev-27B, in percentage points.
- The candidate is served at its pool temperature 1.32 and Kev-27B at its shipped 1.38.
- The exclusions were registered before any read and remove the same rows from both sides:
  - breadth-v1: without `routerbench`, `cfcolor`, `humicroedit` and `chessbench`;
  - tasksource-heldout-v1: without seven families, listed privately;
  - devtools-v1: without `flakeflagger` and `commitpackft_type`.

| stage / criterion (panel, questions) | Kev-27B v2 | Kev-27B | Δ [95 %] | verdict |
|---|---|---|---|---|
| tests: breadth-v1 test accuracy, lower bound > 0 (2,489) | 0.832 | 0.820 | +1.2 [+0.3, +2.2] | pass |
| tests: tasksource-heldout-v1 test accuracy, lower bound > 0 (2,024) | 0.795 | 0.743 | +5.3 [+3.7, +6.8] | pass |
| tests: pooled hard-v1 + devtools-v1 + documents-v1 test accuracy, lower bound ≥ −1 (2,795) | 0.889 | 0.800 | +8.9 [+7.5, +10.3] | pass |
| **locked: transfer-v4 locked accuracy ≥ 0.886 (656)** | **0.8887** (583) | 0.8963 (588) | −0.8 [−2.0, +0.5] | pass |
| locked: served Brier ≤ 0.165 (656) | 0.154 | 0.160 | – | pass |
| bf16 serving check: max \|Δp\| ≤ 0.03 and ≤ 1 flip in 280, plus 8k / 32k / 64k states | see Serving | – | – | pass |

Report-only test reads (same exclusions):

| panel (questions) | Kev-27B v2 | Kev-27B | Δ [95 %] |
|---|---|---|---|
| hard-v1 test (1,088) | 0.918 | 0.749 | +16.9 [+14.2, +19.8] |
| devtools-v1 test, gated sources (771) | 0.825 | 0.789 | +3.6 [+1.3, +5.9] |
| documents-v1 test (936) | 0.908 | 0.869 | +4.0 [+2.1, +5.9] |
| documents-v2, private held-out test (953) | 0.921 | 0.881 | +4.0 [+2.0, +6.1] |
| breadth-v1 test, all 14 datasets (3,089) | 0.757 | 0.748 | +0.8 [−0.1, +1.8] |
| longdoc-v1 test, CUAD contracts (2,194) | 0.874 | 0.890 | −1.6 [−3.0, −0.3] |
| longdoc-v1 test, generated agreement bundles (2,400) | 1.000 | 1.000 | – |

On CUAD test the candidate loses accuracy at every length, and its ECE is 2 to 4 times Kev-27B's in every bucket (accuracy / ECE):

| CUAD test, state length (questions) | Kev-27B v2 | Kev-27B |
|---|---|---|
| under 8k tokens (867) | 0.874 / 0.059 | 0.900 / 0.025 |
| 8k-16k (443) | 0.880 / 0.052 | 0.892 / 0.028 |
| 16k-32k (442) | 0.873 / 0.061 | 0.882 / 0.019 |
| 32k-64k (442) | 0.867 / 0.060 | 0.876 / 0.014 |
| all lengths (2,194) | 0.874 / 0.053 | 0.890 / 0.007 |

The development rule's CUAD accuracy guard (lower bound ≥ −2 pp) would not hold on test (−3.0), as it did not for round 24's unblended SFT (−1.8 pp [−3.2, −0.5], ECE 0.055). Both are report-only at this stage and change no verdict. The generated bundles are at ceiling for every system.

On the ten breadth-v1 datasets that were gated, the gain holds on test. Over all fourteen it is smaller (+0.8 pp), and its interval includes zero. The four excluded datasets were moved to report-only by the 2026-09-27 audit, before round 23.

## Breadth index on test (against Jev and AutoJev)

This is the chance-corrected index of the community Decision Index, computed over breadth-v1's 14 held-out datasets in five areas (`scripts/breadth_report.py`, `runs/r23-breadth-report/`). Jev (through the AI Gateway) and AutoJev-27B (by its own server) were read once, on the same test items, in round 24. Their rows are reused here and were not read again.

| | Kev-27B v2 | Kev-27B | Jev | AutoJev-27B |
|---|---|---|---|---|
| index (95 % CI) | **52.3** [49.2, 55.4] | 50.2 [47.0, 53.2] | 54.0 [51.2, 57.0] | 50.0 [47.0, 53.3] |

Against Kev-27B the index is +2.1 [−0.4, +4.6]. Against Jev it is 1.8 points lower on the point estimates; no paired interval against Jev was registered. Round 24's unblended SFT scored 53.7 on the same items, so the blend gave back about 1.4 index points.

## Results as served (whole suites)

The table below uses whole suites (minus two duplicated CodeReviewer ids), with no exclusions. Each model is at its own served temperature. Numbers come from `runs/release/kev-27b-r23.json` (`scripts/release_numbers.py --release kev-27b-r23`).

| | **Kev-27B v2 (T = 1.32)** | Kev-27B (T = 1.38) |
|---|---|---|
| **locked test**, out-of-domain accuracy / Brier (transfer-v4) | **0.889 / 0.154** | 0.896 / 0.160 |
| locked test, coverage at ≤ 5 % error | **0.875** | 0.835 |
| out-of-domain accuracy / Brier (transfer-v4 development) | 0.851 / 0.218 | 0.848 / 0.229 |
| short states, transfer-r3 test (spent panel, `emotion` included) | 0.858 | 0.879 |
| MMLU-Pro (transfer-v9 development, 10-way) | 0.675 | 0.665 |
| unknowable items answered at ≥ 0.9 (lower is better) | 0.00 | 0.00 |
| breadth-v1 development, all 14 datasets | 0.757 | 0.745 |
| hard-v1 development / test | 0.912 / 0.918 | 0.733 / 0.749 |
| devtools-v1 development / test, all sources | 0.756 / 0.790 | 0.702 / 0.711 |
| documents-v1 development / test (CFPB complaints) | 0.916 / 0.908 | 0.862 / 0.869 |
| SemIf (144 authored decisions) | 0.965 | 0.972 |
| WANLI-v2 (1,002 NLI pairs) | 0.756 | 0.745 |
| TypeSafe (89 answered rows) | 0.854 | 0.865 |

SemIf, WANLI-v2 and TypeSafe are report-only; the audit found them unsound as a gate. scienthoon, the support-ticket suite on which Kev-27B was selected, was removed as an evaluation on 2026-09-27, before round 23, so this candidate has no scienthoon read. Its SFT parent (round 22's final checkpoint) was 5.5 pp below Kev-27B there [−7.8, −3.2] (`PLAN.md`, "Round 22 result", "scienthoon removed").

**Out-of-domain component suites** (development, report-only). These suites are held-out domains of generators that also produced training components, so they are not independent transfer tests. Accuracy / ECE against Kev-27B:
- ood-v2 (4,988 questions): 0.956 / 0.020 against 0.944 / 0.044.
- agents-ood-v1 (2,084): 0.988 / 0.032 against 0.967 / 0.137.
- guardrails-ood-v1 (4,949): 0.984 / 0.010 against 0.944 / 0.079.

## Calibration

`head.pt` carries temperature **1.32** (1.3195). `scripts/calibrate_checkpoint.py` fitted it on round 23's registered pool, and the fit reproduces the round's own pool fit exactly:
- transfer-r3's calibration partition, eight held-out public sources, 448 questions;
- transfer-v9 development MMLU-Pro, 200 questions;
- minus any transfer-v4 development duplicates (none);
- 648 questions in all.

The script refuses rows that share data with the checkpoint's training, and it found none. The pooled T has a 90 % bootstrap interval of [1.20, 1.45]. Out of fold, the pool's ECE goes from 0.048 raw to 0.038 (5 folds, group-disjoint); the two intervals overlap. `KEV_TEMPERATURE=1.0` gives the raw logits.

As served, test-partition ECE against Kev-27B:
- breadth-v1, gated datasets: 0.015 against 0.013;
- tasksource-heldout-v1: 0.049 against 0.051;
- pooled hard / devtools / documents-v1: 0.014 against 0.027.

Served at the ends of the T interval, breadth ECE rises to 0.026 at T 1.20, and tasksource-heldout ECE to 0.067 at T 1.45 (`runs/r23-verdict/27b-tests-t-interval.json`). The two panels pull in opposite directions, so no single T in the interval is best for both.

## Serving (bf16)

The serving check follows round 24's protocol: `scripts/serving_bench.py` on an H200, 200 decision-v7 development records (280 questions), with fused kernels and CUDA graphs. Reports: `runs/serving-27b-r23/report.json`, `runs/serving-27b-r23-long/report.json`.

| | Kev-27B v2 |
|---|---|
| served (CUDA graphs, bf16) vs the fp32 evaluation path, max \|Δp\|, flips | 0.0223, 0 flips |
| isolation: question alone vs the full request, max \|Δp\| | 0.0039, 0 flips |
| long states, served vs benchmark, max \|Δp\| at 8k / 32k / 64k tokens | 0.0064 / 0.0095 / 0.0017, 0 flips |
| model time, new state / cached state at 64k tokens | 9.4 s / 733 ms |
| peak GPU memory at 64k tokens | 87.1 GB |
| GPU memory resident / load time | 65.5 GB / 17.6 s (cached weights) |
| requests/s, decision-v7 development records at 1 / 64 concurrent clients | 21.1 / 36.4 |

The serving context is 65,536 tokens of state, plus at least 8,192 per question branch. Training states were capped at 32,768 tokens, so 32k-64k inputs have been evaluated (longdoc-v1) but not trained on.

## How it was built

- **Base model**: `Qwen/Qwen3.8-27B` (revision `1d4bf0f2`, Apache-2.0). It is a hybrid of Gated DeltaNet and full-attention layers, so questions run as separate causal rows continuing from the shared state (`kev/model.py`).
- **SFT data**: `sft-v2-r22`, private; only its manifest is public (`evals/sft-v2-r22/manifest.json`). The train partition has 145,840 records and 337,130 questions, with states of at most 32,768 tokens. Components:
  - `sft-v1`, 78,786 records. It holds Kev-27B's own training set (decision-v7 with soft targets, the dates / unknowable delta and buried long states), the hard-v1, devtools-v1 and documents-v1 train partitions, and 24 public datasets capped at 500 train records each. It also holds the open-weight-generated synthetic families: long documents, tool routing, retrieval, intent, rubric judging, abstention twins and numeric reasoning.
  - `tasksource-v1`, 24,000 records: 119 commercially licensed task families; list private.
  - `longify`, 3,200 records: sft-v1 train states embedded in 8k-32k documents, exact labels.
  - `longdoc`, 7,617 records: code-assembled long documents with code labels.
  - `ood`, 7,312; `tone`, 7,544 (calm / frustrated / angry minimal pairs); `injection`, 2,699; `agents`, 4,875 (agent-session logs); `guardrails-pii`, 4,152; `guardrails-grounding`, 5,655.
- **How the data was made**:
  - Generated text and labels come from open-weight teachers (GLM-5.3, DeepSeek-V4-Pro, Inkling, Mistral Large 3, gpt-oss-120b, MiMo-V2.6-Pro) or from code.
  - documents-v1's labels come from open-weight teachers, filtered by closed-model judges and adjudication.
  - No Jev outputs were used.
  - Every record was screened against every evaluation partition of every frozen Kev suite, the private evaluation mirror, JevBench's public items and the eval-only suites.
- **SFT settings** (round 22, `experiments/round22/lr2e6.json`):
  - `kev.train --full_ft 1`, one epoch on 8 H200 (FSDP2, fp32 master weights and moments, bf16 backbone);
  - learning rate 2e-6 (head 1e-4), weight decay 0.01, `--batch 8 --accum 2` with length-balanced micro-batches;
  - each state run once with its questions branching from it (`--shared_prefix`);
  - none-of-the-above pairs (p 0.25) only on states of at most 8,192 tokens, and at most 40,960 padded tokens per pass;
  - seed 0.
- **Blend** (round 23, `scripts/interpolate_checkpoint.py --toward`):
  - Each backbone tensor is 0.85 × SFT + 0.15 × Kev-27B, computed in fp32 and rounded once to bf16.
  - Kev-27B (`jaredpalmer/kev-27b@01b81998`, a rank-16 LoRA) enters as base + peft's `get_delta_weight` on its 496 adapted tensors, unrounded.
  - The pointer head is the SFT's.
  - Blended weights sha256 `d27af6ab…` (`interpolation.json` in this repository).
- **Hub relation**: `finetune`. Every weight derives from fine-tunes of the one base; the Hub's `merge` relation is for merges of several listed base models, and Kev-27B is itself an adapter of the same base.

## Known limits

- It is not better than Kev-27B on short states, and it is worse and overconfident on long contracts (see "Read this first"). For contract review, refit the temperature on your own documents, or keep Kev-27B.
- Knowledge is still set by the base. MMLU-Pro is 0.675, against Jev's 0.840 on the same items.
- Its training data covers hard-v1, devtools-v1 and documents-v1, so the large gains there are in distribution.
- The development rule's closest call was short-state accuracy, whose lower bound was −1.95 pp against a −2 pp bar.
- Its temperature has a 90 % interval of ±0.12, which moves test ECE by up to 0.02 (Calibration).

## Intended use

Kev-27B v2 is meant for typed decisions over documents up to 64k tokens: classification, routing, extraction choices and judging, served behind TypeSafe's System One contract, where calibrated probabilities feed thresholds and review queues.
- Freeze thresholds on your own labelled workload. For contract review in particular, refit the temperature on your own documents (`kev.calibrate`).
- It is not a generative model or a chat model. It does not replace a human decision where errors are costly.

## Use

```bash
uv run --extra serve python -m kev.serve --run jaredpalmer/kev-27b-v2-candidate --port 8008   # CUDA, bf16 + fused kernels + CUDA graphs; 51 GB of weights, 65.5 GB resident, 64k context
```

The repository is private, so the server needs a Hugging Face token with access (`hf auth login` or `HF_TOKEN`). Any TypeSafe-compatible client works: `TypeSafeClient(api_key="local", base_url="http://127.0.0.1:8008", model="kev-latest")`.

## License

Apache-2.0 for the weights and head. The Qwen3.8-27B base is Apache-2.0. The training data carries its own licences, recorded per source in the manifests:
- Of sft-v1's 24 public datasets, four are share-alike: ARC (CC-BY-SA-4.0), HotpotQA (CC-BY-SA-4.0), Natural Questions (CC-BY-SA-3.0) and SNLI (CC-BY-SA-4.0). The other twenty are CC-BY-4.0, MIT or Apache-2.0.
- Kev-27B's own training set (decision-v7's ten public sources) is carried over unchanged, with the terms listed for Kev-27B.
- The CFPB complaints (documents-v1) are US government works.
- devtools-v1's sources are licence-checked; CodeReviewer diffs are kept only from permissively licensed projects.
- The generated components use open-weight teachers whose licences leave their outputs unrestricted. One of them, GLM-5.3, has an MIT-style licence with a condition on very large model-as-a-service operators.

The corpus itself is private. Its manifests record each source's licence, revision and screens.

## Provenance

- Checkpoint: round 23 arm `27b-k-w85` on the `kev-runs` volume (`/runs/r23-wise/27b-k-w85/checkpoint`). Release copy: `/runs/release/kev-27b-r23/checkpoint`, whose weights sha256 is equal to the source's. Its `head.pt` has sha256 `7968f17b…` and carries T 1.32.
- Hub: `jaredpalmer/kev-27b-v2-candidate` (private), commit `0dd33bcc`. The Hub's file hashes give the same weights and `head.pt` hashes. Loaded from the Hub with a token on an H200, it reproduced round 23's semif-v1 read row for row: 252 of 252 rows had identical logits before the temperature, with T 1.32 applied.
- Internal release id: `kev-27b-r23`. The name `kev-27b-v2` already belongs to the released Kev-27B's records.
- Evidence: `runs/release/kev-27b-r23.json`, `runs/release/kev-27b-r23-staging.json` (copy, hashes, temperature fit and Hub verification), `runs/r23-readout/`, `runs/r23-verdict/` and `PLAN.md` ("Round 22", "Round 23", "Round 24", "Release candidate: Kev-27B v2").
