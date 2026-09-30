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
metrics:
  - accuracy
  - brier_score
  - expected_calibration_error
model-index:
  - name: Kev-9B v2
    results:
      - task: { type: text-classification, name: typed decision, out-of-domain, locked test }
        dataset: { type: mixed, name: "transfer-v4 test (read once)" }
        metrics:
          - { type: accuracy, value: 0.852 }
          - { type: brier_score, value: 0.199 }
      - task: { type: text-classification, name: typed decision, long policy documents }
        dataset: { type: mixed, name: "hard-v1 test (1,088 questions)" }
        metrics:
          - { type: accuracy, value: 0.834 }
      - task: { type: text-classification, name: typed decision, developer tools }
        dataset: { type: mixed, name: "devtools-v1 test (1,071 questions)" }
        metrics:
          - { type: accuracy, value: 0.791 }
      - task: { type: text-classification, name: typed decision, real complaints }
        dataset: { type: mixed, name: "documents-v1 test (936 questions)" }
        metrics:
          - { type: accuracy, value: 0.900 }
---

# Kev-9B v2 (release candidate, not published)

Kev-9B v2 is a **decision model**: one document (the *state*) and a set of typed questions in, a probability distribution per question out, in one forward pass. No text generation. It is a LoRA adapter (r=16) plus a pointer head on `Qwen/Qwen3.5-9B-Base` (revision `68c46c4b`), serving TypeSafe's public `/v1/systemone` contract.

It is the released Kev-9B plus one more epoch of training on long policy documents, developer-tool decisions and real consumer complaints: the same data that took Kev-4B and Kev-0.8B from their first releases to their current ones. It keeps the released Kev-9B's accuracy on everything else, including the locked test, and is better calibrated.

- Candidate of round 27 (`PLAN.md`, "Round 27 (registered)" and "Round 27 result"): trial `runs/r18-9b/00-trial-0`, staged at `/runs/release/kev-9b-r27/checkpoint` on the `kev-runs` volume (adapter sha256 `2b2a70cf…`, `head.pt` `8e1dab2c…`, T 2.19).
- Every number below: `runs/release/kev-9b-r27.json` (`scripts/release_numbers.py --release kev-9b-r27`), each side served at the temperature in its `head.pt`.

## Results (same frozen items for both columns)

| | Kev-9B (released, T 2.30) | **Kev-9B v2 (T 2.19)** | v2 − released, accuracy [95 %] |
|---|---|---|---|
| long policy documents, hard-v1 test (1,088) | 0.584 | **0.834** | +25.0 pp [+22.0, +28.1] |
| developer tools, devtools-v1 test (1,071) | 0.637 | **0.791** | +15.4 pp [+11.0, +19.4] |
| real complaints, documents-v1 test (936) | 0.829 | **0.900** | +7.1 pp [+4.7, +9.2] |
| real complaints, documents-v2 (private, 953) | 0.821 | **0.900** | +8.0 pp [+5.9, +10.2] |
| 14 held-out datasets, breadth-v1 test (3,089) | 0.692 | 0.698 | +0.6 pp [−0.4, +1.6] |
| out-of-domain, transfer-v4 dev (656) | 0.822 | 0.820 | −0.2 pp [−2.4, +1.8] |
| short-state, transfer-r3 test (1,150) | 0.847 | 0.847 | +0.0 pp [−1.2, +1.2] |
| SemIf authored (144) | 0.910 | 0.917 | +0.7 pp [−3.5, +4.9] |
| **locked test**, transfer-v4 test (656), accuracy / Brier | 0.852 / 0.224 | **0.852 / 0.199** | +0.0 pp [−1.7, +1.8] |
| MMLU-Pro (transfer-v9 dev) | 0.515 | 0.590 | |
| unknowable items answered at ≥ 0.9 (transfer-v9) | 0.00 | 0.00 | |

Chance-corrected index on the 14 held-out datasets (breadth-v1 test, `scripts/breadth_report.py`): 41.0 [38.8, 43.9] against the released Kev-9B's 40.0 [38.1, 43.0].

**Calibration.** `head.pt` carries T = 2.19, fitted on held-out datasets the checkpoint never trained on: transfer-r3's calibration partition (eight sources) and transfer-v9's MMLU-Pro, 648 questions, 90 % interval [2.05, 2.41]. The released Kev-9B's 2.30 was fitted on its own in-distribution development rows, which the standing rules no longer allow. On the fit rows ECE goes 0.125 raw → 0.041; on transfer-v4 development 0.103 → 0.041. Locked-test ECE 0.034 (released 0.042), confident errors 1.4 % (3.2 %), coverage at ≤ 5 % error 0.74 (0.65). `KEV_TEMPERATURE=1.0` restores raw logits.

## How it was decided

Round 18 (2026-09-24) trained this delta and selected no candidate: it passed both primaries and failed only WANLI-v2, scienthoon and the pooled externals, all since removed as unsound gates. Round 27 re-selected round 18's two finished arms on round 18's rule with the 2026-09-27 audit's changes (no external gates, `emotion` out of the short-state panel, short-state Brier ≤ +0.02), both arms passed, and the larger primary won. Confirmation, read once: hard-v1 + devtools-v1 test pooled +18.7 pp [+16.7, +20.8], documents-v1 test +7.1 [+4.7, +9.2], locked transfer-v4 accuracy +0.0 [−1.7, +1.8] against a −1 pp bar, served Brier −0.025 [−0.047, −0.007]. The selection was not blind (round 18's development reads were known); the test partitions and the locked read are the guard.

## How it was built

- **Parent**: `jaredpalmer/kev-9b@2629c06a` (the `decision-v7` recipe plus the dates/unknowable delta; see its card).
- **Delta**: `kev.train --init_from jaredpalmer/kev-9b --data evals/round15/joint/train.jsonl --replay 10000 --lr 2e-5 --epochs 1 --max_state 7552 --p_none_pair 0.25`, bf16, gradient checkpointing, one H200, 2.7 h. `evals/round15/joint` is documents-v1 train (CFPB complaint narratives) concatenated with round 10's skills data (hard-v1 and devtools-v1 train), 16,539 records, the joint recipe that confirmed Kev-0.8B in round 15; the replay draws 10,000 records from `decision-v7`. No Jev outputs were used.

## Known limits

- Trained on states of up to 7,552 tokens. Longer documents are served, but not what it was trained for.
- Knowledge (MMLU-Pro 0.59) is set mostly by the base.
- devtools-v1 calibration is the weakest (ECE 0.098 on test).
- DeltaNet kernels have no MPS implementation; on Apple Silicon `kev.serve` runs it through MLX.

## Use

```bash
uv run --extra serve python -m kev.serve --run jaredpalmer/kev-9b --port 8008   # after release
```

Any TypeSafe-compatible client works: `TypeSafeClient(api_key="local", base_url="http://127.0.0.1:8008", model="kev-latest")`.

## License

Apache-2.0 for the adapter and head; the Qwen3.5 base is Apache-2.0; datasets carry their own licenses.
