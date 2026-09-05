# Vi-PPE-mini: A Vietnamese Pairwise Preference Benchmark for Reliable LLM-as-a-Judge Evaluation

**A Vietnamese pairwise preference benchmark for reliable LLM-as-a-judge evaluation.**

[Dataset card](reports/dataset_card_llm_v4_frozen.md) | [Main results](reports/experiment_results.md) | [Annotation guideline](reports/annotation_guideline.md) | [Phase status](reports/phase_status.md)

Vi-PPE-mini evaluates whether an LLM judge can select the better Vietnamese response while remaining robust to answer order, verbosity, style, and prompt wording. It covers three domains: fact checking, instruction following, and safety.

The benchmark contains 900 reviewed response pairs. Candidate pairs were constructed from Vietnamese source resources with GPT-assisted generation, then checked by two project reviewers using domain-specific rubrics and automatic validation. Every evaluation pair is judged in both its original AB order and its swapped BA order.

## Why This Project

Pairwise LLM evaluation is useful for open-ended generation, but a high accuracy score can hide unstable behavior. Vi-PPE-mini therefore measures both decision quality and robustness:

- pairwise and macro accuracy;
- AB/BA swap consistency;
- parse success for structured judge outputs;
- domain-level performance;
- verbosity and style preference on controlled bias pairs.

## Research Questions

The project is organized around four practical questions:

1. How accurately can open instruction-tuned models judge Vietnamese response pairs?
2. Does a judge preserve the same semantic preference after Response A and Response B are swapped?
3. Do explicit domain criteria improve judging, or can a longer rubric make the decision less reliable?
4. Can a short anti-bias instruction reduce preference for longer or more polished responses?

These questions motivate the benchmark design, the AB/BA protocol, and the separation between the main held-out test set and the controlled bias subset.

## Benchmark Composition

| Split | Fact check | Instruction | Safety | Total |
|---|---:|---:|---:|---:|
| Development | 34 | 33 | 30 | 97 |
| Main test | 166 | 167 | 170 | 503 |
| Controlled bias | 100 | 100 | 100 | 300 |
| **Total** | **300** | **300** | **300** | **900** |

Active frozen files:

- `data/processed/pairs_dev_llm_v4.jsonl`
- `data/processed/pairs_test_llm_v4.jsonl`
- `data/processed/bias_subset_llm_v4.jsonl`
- `data/processed/dataset_manifest_llm_v4.json`

The source resources include ViFactCheck, UIT-ViQuAD2.0, UIT-ViHSD, and manually designed Vietnamese task templates. See [data/README.md](data/README.md) and [real_data_sources.md](reports/real_data_sources.md) for provenance notes.

### Domain Rubrics

| Domain | Main quality criteria | Typical controlled error |
|---|---|---|
| Fact checking | Evidence faithfulness, correctness, calibration, no hallucination | Unsupported detail or overconfident conclusion |
| Instruction following | Constraint following, task completion, specificity, required format | Wrong item count, extra content, or format violation |
| Safety | Harm avoidance, appropriate refusal, helpfulness, no over-refusal | Unsafe compliance, toxic language, or unnecessary refusal |

### Pair Format

Each JSONL record contains the information required for judging and later auditing:

```json
{
  "pair_id": "unique_pair_id",
  "domain": "fact_check | instruction | safety",
  "prompt": "Vietnamese task prompt",
  "evidence": "optional supporting evidence",
  "response_a": "first candidate response",
  "response_b": "second candidate response",
  "gold_winner": "A | B | tie",
  "gold_reason": "short reviewer rationale",
  "criteria": ["domain-specific criteria"],
  "perturbation_type": "controlled error category",
  "source_dataset": "provenance source",
  "source_example_id": "source-level identifier"
}
```

The pair builders create contrastive candidates: one response is intended to be more correct, faithful, safe, or instruction-compliant, while the other contains a controlled defect. Gold labels are then checked against the prompt, evidence, and domain rubric by two project reviewers.

## Evaluation Pipeline

```text
Vietnamese source datasets and task templates
                    |
                    v
        normalization and provenance
                    |
                    v
       GPT-assisted preference pairs
                    |
                    v
     two-reviewer quality verification
                    |
                    v
       validation and frozen manifest
                    |
                    v
  AB/BA judging with Vietnamese prompts
                    |
                    v
 accuracy, swap consistency, and bias diagnostics
```

The gold winner is never shown to the judge model. It is used only after inference to compute metrics.

## Judge Protocol

For every pair, the same judge is called twice:

1. **AB order:** the original Response A and Response B are shown.
2. **BA order:** the responses are swapped, while all other pair content remains unchanged.
3. **Normalization:** the BA prediction is mapped back into the original A/B label space.
4. **Aggregation:** the pair is marked consistent only when both orders express the same semantic preference.

The model returns structured JSON containing a winner, confidence, and a short Vietnamese rationale. The prompt does not expose `gold_winner`, `gold_reason`, or any annotation-only field.

### Reported Metrics

| Metric | What it measures |
|---|---|
| Pairwise accuracy | Fraction of valid aggregated decisions matching the gold winner |
| Macro accuracy | Mean accuracy across fact checking, instruction following, and safety |
| Swap consistency | Fraction of pairs with the same semantic preference in AB and BA orders |
| Parse success | Fraction of judgments parsed into the required output schema |
| Conditional accuracy | Accuracy on bias pairs with a valid non-tie aggregated choice |
| Verbosity bias rate | Rate at which the longer response is selected among valid controlled choices |
| Style bias rate | Rate at which the polished-style response is selected on style-controlled pairs |

Reporting accuracy together with swap consistency is important: a judge may obtain a plausible single-order score while changing its decision when only the answer position changes.

## Main Results (Official V5)

Four open judge families were evaluated with deterministic inference: Qwen2.5-3B-Instruct, Qwen2.5-7B-Instruct, Gemma-3-4B-IT, and Llama-3.1-8B-Instruct.

### Main Held-Out Test

| Best setting | Accuracy | Macro accuracy | Swap consistency | Parse success |
|---|---:|---:|---:|---:|
| Qwen2.5-7B + generic baseline prompt | **94.04%** | **94.00%** | **94.43%** | **100.00%** |

Domain accuracy for the best core setting:

| Fact check | Instruction | Safety |
|---:|---:|---:|
| 93.37% | 89.22% | 99.41% |

Full V5 core matrix:

| Model | Prompt | Accuracy | Macro | Swap | Parse |
|---|---|---:|---:|---:|---:|
| Qwen2.5-3B | generic | 83.70% | 83.63% | 84.89% | 100.00% |
| Qwen2.5-3B | Criteria | 75.15% | 75.11% | 73.56% | 100.00% |
| **Qwen2.5-7B** | **generic** | **94.04%** | **94.00%** | **94.43%** | **100.00%** |
| Qwen2.5-7B | Criteria | 90.06% | 90.00% | 91.45% | 99.90% |
| Gemma-3-4B | generic | 86.48% | 86.42% | 87.67% | 99.11% |
| Gemma-3-4B | Criteria | 76.34% | 76.32% | 77.14% | 100.00% |
| Llama-3.1-8B | generic | 86.88% | 86.79% | 88.27% | 100.00% |
| Llama-3.1-8B | Criteria | 89.86% | 89.80% | 91.45% | 100.00% |

### Controlled Bias Subset

| Best setting | Conditional accuracy | Macro accuracy | Swap consistency | Verbosity choice rate |
|---|---:|---:|---:|---:|
| Qwen2.5-7B + bias-mitigated prompt | **79.26%** | **79.20%** | **83.00%** | **47.83%** |

Full V5 controlled-bias matrix:

| Model | Prompt | Conditional accuracy | Macro | Swap | Verbosity |
|---|---|---:|---:|---:|---:|
| Qwen2.5-3B | generic | 62.88% | 62.79% | 66.67% | 65.00% |
| Qwen2.5-3B | Bias-mitigated | 62.88% | 62.81% | 62.00% | 53.27% |
| Qwen2.5-7B | generice | 76.59% | 76.49% | 79.33% | 55.20% |
| **Qwen2.5-7B** | **Bias-mitigated** | **79.26%** | **79.20%** | **83.00%** | **47.83%** |
| Gemma-3-4B | generic | 73.91% | 73.87% | 77.33% | 55.91% |
| Gemma-3-4B | Bias-mitigated | 67.89% | 67.82% | 69.33% | 56.07% |
| Llama-3.1-8B | generic | 77.26% | 77.19% | 86.67% | 70.34% |
| Llama-3.1-8B | Bias-mitigated | 76.25% | 76.19% | 79.67% | 64.29% |

The official reported experiment version is **V5**. Its main finding is that prompt complexity is model- and domain-dependent: a concise generic prompt produced the strongest overall core result, while explicit anti-bias instructions helped Qwen2.5-7B on the controlled bias subset. Later stricter-prompt runs are treated only as ablations and do not replace the V5 results.

Full tables and error analysis are available in [experiment_results.md](reports/experiment_results.md) and [phase_12_report.md](reports/phase_12_report.md).

## Key Findings

- **Model capacity matters.** Qwen2.5-7B substantially outperforms Qwen2.5-3B on both core accuracy and order robustness.
- **More rubric text is not always better.** Criteria prompting improves some fact-checking configurations but degrades instruction-following accuracy for Qwen2.5-3B and Gemma-3-4B.
- **Bias mitigation is model-dependent.** It improves Qwen2.5-7B accuracy, swap consistency, and verbosity behavior, but does not transfer consistently to Gemma or Llama.
- **Safety is the strongest domain for the best model.** Qwen2.5-7B baseline reaches 99.41% safety accuracy, while instruction following remains more difficult.
- **Schema reliability and semantic reliability are different.** Most runs parse nearly perfectly, yet some still show substantial AB/BA inconsistency.

Representative failures include invented format violations, inconsistent decisions after answer swapping, preference for unsupported verbose answers, and literal schema outputs such as `Response A` instead of `A`. See [phase_12_report.md](reports/phase_12_report.md) for the V5 error analysis.

## Repository Structure

```text
configs/       Model and experiment configuration
data/          Source, processed, and frozen benchmark data
notebooks/     Colab A100 experiment notebooks
prompts/       Vietnamese judge prompt templates
reports/       Paper, dataset documentation, and result summary
results/       Judgments, metrics, figures, and error cases
scripts/       Data, inference, metrics, and audit entry points
src/vi_ppe/    Reusable Python package
tests/         Unit and smoke tests
```

## Local Setup

Local execution is intended for data validation, unit tests, mock inference, and metric checks. Full model inference should be run on a GPU environment.

```bash
python -m pip install -r requirements.txt
python scripts/00_smoke_check.py
pytest -q
```

Mock judge smoke test:

```bash
python scripts/05_run_judge.py \
  --dataset data/samples/smoke_pairs.jsonl \
  --backend mock \
  --template baseline_generic_vi \
  --run-id smoke_mock \
  --resume

python scripts/06_compute_metrics.py \
  --judgments results/judgments/smoke_mock.jsonl \
  --dataset data/samples/smoke_pairs.jsonl \
  --run-id smoke_mock
```

The V5 experiments were executed on Colab A100 in BF16 with deterministic decoding (`do_sample=False`, `temperature=0`). Reproduction notebooks include `notebooks/Vi_PPE_V5.ipynb` and `notebooks/Vi_PPE_V5_Gemma_Llama.ipynb`.

## Reproduction Workflow

The workflow separates CPU-safe development from GPU inference:

1. Run the smoke check and unit tests locally.
2. Validate the frozen dataset and its manifest.
3. Open the V5 notebooks on Colab A100.
4. Run each judge configuration with `--resume` so interrupted runs continue from existing JSONL judgments.
5. Compute core and bias metrics from completed judgments.
6. Compare AB/BA consistency and inspect pair-level failures before reporting conclusions.

Useful entry points:

| File | Purpose |
|---|---|
| `scripts/03_validate_dataset.py` | Validate pair schema and dataset constraints |
| `scripts/05_run_judge.py` | Run AB/BA judge inference with resume support |
| `scripts/06_compute_metrics.py` | Compute core and controlled-bias metrics |
| `src/vi_ppe/judge_backends/hf_local.py` | Batched Hugging Face inference backend |
| `src/vi_ppe/aggregate_swaps.py` | Normalize and aggregate AB/BA decisions |
| `configs/models.yaml` | Model precision, batch size, and hardware profiles |

## Project Status

Phases 0-13 are complete: project scaffold, source adapters, pair construction, annotation and QA, frozen splits, prompt design, inference engine, metrics, V5 Colab experiments, error analysis, and paper packaging. Phase 14, an optional LoRA/QLoRA branch, is not part of the reported benchmark results.

Detailed progress is available in [phase_status.md](reports/phase_status.md) and the individual `reports/phase_XX_report.md` files.

## Limitations

- Candidate responses are GPT-assisted rather than collected from deployed Vietnamese chat systems.
- The benchmark covers fact checking, instruction following, and safety, but not translation, summarization, code, tool use, or long-context tasks.
- Two project reviewers verified the pairs, but the reported version does not include formal inter-annotator agreement or blind adjudication.
- The controlled bias subset is diagnostic; verbosity, style, completeness, and helpfulness may remain partially entangled.

## Authors

- Khanh T. M. Truong
- Lam T. Bui

Faculty of Information Science and Engineering, University of Information Technology, Vietnam National University Ho Chi Minh City.

## Citation

```bibtex
@misc{truong2026vippe,
  title  = {Vi-PPE-mini: A Vietnamese Pairwise Preference Benchmark for Reliable LLM-as-a-Judge Evaluation},
  author = {Truong, Khanh T. M. and Bui, Lam T.},
  year   = {2026}
}
```
