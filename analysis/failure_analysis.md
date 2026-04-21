# Bao cao Failure Analysis (Role 1 - da dong bo so lieu run moi nhat)

## 1. Benchmark Snapshot (real run)

Run command:
```bash
python data/synthetic_gen.py
python main.py
python check_lab.py
```

Data source:
- `reports/summary.json`
- `reports/benchmark_results.json`

Run metadata (V2):
- Timestamp: **2026-04-21 15:37:28**
- Version: **Agent_V2_Optimized**
- Total cases: **60**
- Elapsed: **10.17s**

V2 metrics (exact from `summary.json`):
- Pass / Fail / Error: **54 / 6 / 0**
- Avg score: **4.0583**
- HitRate@3: **0.8667**
- Agreement rate: **0.9333**
- Cohen's Kappa: **0.5780**
- p95 latency: **0.0642s**
- Total eval cost (judge): **$0.079357**
- Total tokens in / out: **42567 / 184**

Regression (V1 -> V2):
- V1 avg_score: **4.1000**
- V2 avg_score: **4.0583**
- Delta avg_score: **-0.0417**
- Delta hit_rate@3: **+0.1000**
- Gate decision: **BLOCK**

Release-gate reasons:
- `avg_score delta -0.0417 < required +0.2 - FAIL`
- `hit_rate@3 delta +0.1000 within tolerance - OK`
- `agreement_rate 0.9333 >= 0.7 - OK`
- `p95_latency 0.064s within 1.2x V1 - OK`

---

## 2. Bonus Role 1 (retrieval stress test)

### 2.1. Top-k sensitivity (from `benchmark_results.json`)

| Metric | Value |
|---|---:|
| HitRate@1 | **0.7333** |
| HitRate@3 | **0.8667** |
| HitRate@5 | **0.8667** |
| MRR | **0.7917** |

Key readout:
- `Hit@3 - Hit@1 = +0.1334` -> ranking quality improves clearly from top-1 to top-3.
- `Hit@5 == Hit@3` -> expanding context beyond top-3 does not add recall gain in this run.

### 2.2. Hard-cases pack fail rate (bonus condition >20%)

Fail definition for Role 1 bonus:
- A hard-case is counted as fail if retrieval misses at `hit_rate@3 = 0`.

Hard-case scope:
- `adversarial_prompt_injection` (12)
- `edge_ambiguous_ooc` (8)
- `conflicting_information` (6)
- `multi_turn` (4)
- Total: **30**

Result:
- Hard-case retrieval fail: **8 / 30**
- Hard-case retrieval fail-rate: **26.67%** -> **passes bonus threshold (>20%)**

Breakdown by case type:

| Case type | Retrieval fail | Total | Rate |
|---|---:|---:|---:|
| adversarial_prompt_injection | 0 | 12 | 0.00% |
| edge_ambiguous_ooc | 4 | 8 | 50.00% |
| conflicting_information | 2 | 6 | 33.33% |
| multi_turn | 2 | 4 | 50.00% |

---

## 3. Failure clustering (exact from summary)

From `summary.regression.failure_clusters` with **6 fail cases**:

| Cluster | Cases | Score range |
|---|---|---|
| `prompt_attack` | `ADV-004`, `ADV-009` | 1.0 - 2.0 |
| `retrieval_miss` | `CFG-003`, `CFG-006`, `MTR-003`, `MTR-004` | 1.0 - 2.0 |
| `hallucination` | *(none)* | - |
| `incomplete` | *(none)* | - |
| `tone_mismatch` | *(none)* | - |

Interpretation:
- Main failure source is retrieval-level miss on conflicting and multi-turn cases.
- Prompt-attack guardrail is triggered, but answer quality can still fail when context selection is off.

---

## 4. 5 Whys (3 representative fail cases)

### Case #1: `CFG-006` (conflicting_information, retrieval_miss)
1. Symptom: answer drifts to release-gate metric instead of resolving SLA conflict (2h vs 4h).
2. Why 1: retriever selected `doc_release_gate` instead of SLA conflict docs.
3. Why 2: keyword overlap around "release" dominated ranking.
4. Why 3: no intent disambiguation layer for conflicting-policy queries.
5. Why 4: no rerank step weighted on SLA entities (`SLA`, `2h`, `4h`).
6. Root cause: retrieval ranking is still keyword-heavy for conflict resolution cases.

### Case #2: `ADV-009` (adversarial_prompt_injection, prompt_attack)
1. Symptom: injection was detected, but answer still referenced wrong supporting doc.
2. Why 1: top context was not the most relevant doc for reranking question.
3. Why 2: generation path relied on first context too early.
4. Why 3: no post-retrieval semantic reranker before final answer synthesis.
5. Why 4: keyword priority can over-weight generic "retrieval" topic.
6. Root cause: combined ranking + generation-context coupling issue.

### Case #3: `MTR-003` (multi_turn, retrieval_miss)
1. Symptom: follow-up turn failed to retrieve performance doc in top-3.
2. Why 1: retrieval matched current utterance too literally.
3. Why 2: prior-turn intent was not rewritten into explicit query terms.
4. Why 3: history-aware query rewriting is not enabled.
5. Why 4: no dedicated multi-turn retrieval policy.
6. Root cause: missing conversation-aware retrieval strategy.

---

## 5. Action plan (Role 1 scope only)

- [ ] Add intent-aware query normalization for `edge_ambiguous_ooc` and `conflicting_information`.
- [ ] Expand phrase/synonym coverage for ambiguous queries.
- [ ] Add history-aware query rewriting for `multi_turn`.
- [ ] Track fixed Role 1 bonus metrics on every run:
  - Top-k sensitivity (`Hit@1/@3/@5`, `MRR`)
  - Hard-case retrieval fail-rate (`hit@3=0` on hard-cases pack)
