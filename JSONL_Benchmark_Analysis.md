# Benchmark Results Analysis

**Archive analysed:** `results(1).zip`  
**Report date:** 2026-10-22  
**Scope:** all JSONL files under `results/`, nested experiment folders included; CSV summaries used as supplementary evidence.

## 1. Executive summary

- **Data coverage:** 38 JSONL files, **1,230 parsed records**, 38 distinct run IDs, and **0 malformed/non-object lines**. Every nonblank JSONL line parsed successfully.
- **Experiment mix:** 604 single-needle (NIAH) records, 90 distractor records, 176 multi-hop records, and 360 negative-retrieval records.
- **Best-supported result:** Qwen3 4B achieved 60/60 strict exact matches on the distractor benchmark and 120/120 on multi-hop retrieval at nominal 4k/8k contexts. This is strong evidence for these generated test cases, not proof of general long-context reasoning.
- **Small-model strength and boundary:** Qwen3 0.6B scored 87/90 (96.7%) on one NIAH run spanning 8k, 16k, and 32k contexts. In a separate 32k/36k/40k run, results fell from 5/5 at 32k to 3/5 at 36k and 2/5 at 40k. The latter sample is small, but the degradation is material.
- **Largest capability gap:** Qwen3 0.6B scored 34/50 (68.0%) on the larger multi-hop run, versus 19/20 (95.0%) on its main distractor run. Following a single explicit needle is much easier here than joining information across linked facts.
- **Negative retrieval remains weak:** the model often fails to extract a present answer, and prompt tightening mostly improves abstention when evidence is absent. On the tight-prompt 4k test, absent-answer refusal was 20/20 (100%), but correct extraction was only 3/20 (15%).
- **Do not pool every file into one model score:** there are seven schema versions, partial runs, repeated/overlapping test designs, different prompts, and inconsistent runtime metadata. The CSV summaries also conflict with some JSONL runs and have no model/run ID, so they must be reported separately.

## 2. What is in the archive

| Dimension | Count / observation |
|---|---:|
| JSONL files | 38 |
| Parsed JSONL records | 1,230 |
| Distinct run IDs | 38 |
| Malformed/non-object JSONL lines | 0 |
| Response records explicitly marked non-`ok` | 1 |
| Schema versions | v1.0: 380, v1.1: 20, v1.2: 4, v1.3: 20, v1.4: 166, v1.5: 68, v1.6: 572 |
| Models represented | llama3.2:1b: 6, llama3:8b: 385, qwen3:0.6b: 610, qwen3:4b: 229 |

### Records by experiment family

| Experiment | Records | What it tests |
|---|---:|---|
| `NIAH` | 604 | Retrieve a single inserted fact from a long context |
| `NIAH_distractor` | 90 | Retrieve the target while ignoring similar competing facts |
| `NIAH_multihop` | 176 | Link facts/relationships to determine the requested value |
| `negative_retrieval` | 360 | Decide whether evidence exists; answer with a value or abstain using `NO_EVIDENCE_FOUND` |

### Metrics found

The schemas do not use one uniform metric set. Across records, the metric-key combinations were:

| Metric keys | Records | Interpretation / caveat |
|---|---:|---|
| `lexical_exact_match`, `instruction_compliance`, `llm_judge` | 360 | Exact answer, format/instruction compliance, and judge score |
| `lexical_exact_match`, `instruction_compliance` | 86 | Exact answer and instruction compliance; no judge score |
| `lexical_exact_match`, `llm_judge`, `semantic_similarity` | 399 | Older/newer NIAH-style scores; metrics are not equivalent to answerability scoring |
| `lexical`, `llm_judge`, `semantic_similarity` | 24 | Legacy name for the lexical score |
| `lexical_exact_match`, `semantic_similarity` | 1 | No judge score |
| `answerability_exact_match` | 240 | Strict expected-value / expected-abstention correctness |
| `answerability_exact_match`, `llm_judge` | 120 | Answerability exact match plus judge |

For this report, `lexical_exact_match` (and legacy `lexical`) is used for NIAH-family exact-match rates. Negative-retrieval rates use `answerability_exact_match`. These are **strict** metrics: a correct value embedded in an explanation or a reformatted code can fail exact-match scoring.

## 3. Main findings by experiment

### 3.1 Single-needle retrieval (NIAH)

#### Qwen3 0.6B — the best full NIAH sweep

The most informative run is `478efbee30ab` (schema v1.4, 90 records). It covers 8k, 16k, and 32k target contexts, with five requested insertion depths and repeated needles.

| Nominal target context | Exact matches | Exact-match rate | Median response latency |
|---:|---:|---:|---:|
| 8,000 | 30/30 | 100.0% | 5.28 s |
| 16,000 | 29/30 | 96.7% | 8.56 s |
| 32,000 | 28/30 | 93.3% | 19.01 s |
| **Total** | **87/90** | **96.7%** | **8.56 s overall median** |

Interpretation:

- Performance remained high through the 32k cases in this run, while median latency increased substantially as context length increased.
- One of the three exact-match failures was a formatting variant: expected `QUANTUM-LEAP-99`, predicted `quantum leap 99`; its LLM-judge score was 0.99. So strict exact match slightly understates semantic retrieval in this case.
- The other two failures were genuine: at 32k and depth 0.0, the model answered using unrelated narrative details instead of the inserted secret. This points to a possible interaction between long contexts and early-position retrieval.
- `instruction_compliance` averaged about 0.711 in this run. High retrieval accuracy did not always mean that the response followed the expected terse answer format.

#### Qwen3 0.6B — degradation near the edge of the tested context range

Run `85d237650c49` has 15 records across 32k, 36k, and 40k. It is a separate run/design and should not be merged with the 90-record run above.

| Nominal target context | Exact matches | Exact-match rate | Median response latency |
|---:|---:|---:|---:|
| 32,000 | 5/5 | 100.0% | 69.28 s |
| 36,000 | 3/5 | 60.0% | 97.95 s |
| 40,000 | 2/5 | 40.0% | 67.46 s |

The five errors in this run were concentrated at depth 0.0, 0.25, and 0.5; all three cases at 0.75 and all three at 1.0 succeeded. That pattern is consistent with a positional/recency effect as well as a context-length effect. But each context has only five cases, and the run lacks per-response `prompt_eval_count`, so treat the exact boundary as preliminary rather than a precise capability limit.

A separate 18-record run at nominal 32k (`81e709779b80`) scored 17/18 (94.4%). The differing results and latency across separate 32k runs reinforce the need to pin the exact prompts, context configuration, and runtime settings before making a single claim for “32k accuracy.”

#### Other NIAH model runs

| Model / run | Records | Exact matches | Contexts / positions | Interpretation |
|---|---:|---:|---|---|
| Qwen3 4B, all NIAH JSONL runs combined descriptively | 49 | 49/49 (100%) | Mostly 4k; includes 3 cases at 32k | Promising, but the 32k evidence is only 3/3. The 32k run had median latency about 201 s, so the result is not cheap to obtain. |
| Llama 3 8B, run `2bfcd3f32b11` (schema v1.0) | 330 | 260/330 (78.8%) | 7k–8k; 11 insertion depths | Large sample, but a striking edge-position weakness: depth 0.0 was 2/30 (6.7%) and depth 1.0 was 19/30 (63.3%), while many middle positions were 90–97%. |
| Llama 3 8B, run `7de845de1d9d` (schema v1.4) | 36 | 34/36 (94.4%) | 7k/8k at depths 0.1, 0.5, 0.9 | The value was found in 34 cases, but instruction compliance was 0/36 because every response added prose. This is retrieval success, not format compliance; it is not a controlled comparison with the 330-record run. |
| Llama 3 8B, run `de2cecc1c5b7` | 18 | 18/18 (100%) | About 4k; 3 insertion depths | Small 4k run only; should not be generalized to longer contexts. |
| Llama 3.2 1B, run `d0771fde605b` | 6 | 4/6 (66.7%) | Nominal 128k | Proof-of-concept only: six cases, roughly 105.6 s median latency, and no per-response prompt token count to independently verify effective input length. |

**Important caution about the Llama 3 8B result:** the 330-record v1.0 run has a very unusual positional profile. The later 36-record run does not reproduce the same catastrophic edge-position failure, but the two runs differ in schema, insertion depths, and case generation. Do not interpret this as a model-version improvement without rerunning identical case fingerprints under identical runtime configuration.

### 3.2 Distractor retrieval

| Model / run | Records | Exact matches | Nominal contexts | Findings |
|---|---:|---:|---|---|
| Qwen3 0.6B, primary run `9c14a3eac905` | 20 | 19/20 (95.0%) | About 4k; depths 0.1–0.9 | One failure at depth 0.7: the model claimed no code for Brightwater was present. The other 19 cases were exact. |
| Qwen3 0.6B, earlier run `23837261779c` | 10 | 10/10 (100%) | About 4k; depth 0.5 only | Smaller and narrower test; corroborative but not an independent broad sweep. |
| Qwen3 4B, run `aadb46540671` | 60 | 60/60 (100%) | About 4k and 8k; depths 0, 0.5, 1.0 | Perfect exact match across all tested context/depth groups. Median latency was about 23.2 s. |

Within this generated distractor suite, the 0.6B model is already strong; the 4B run removes the one observed failure in a larger sample. This does **not** mean distractors are universally solved: the test uses structured, synthetic entities and a limited set of templates.

### 3.3 Multi-hop retrieval

| Model / run | Records | Exact matches | Rate | By insertion depth / note |
|---|---:|---:|---:|---|
| Qwen3 0.6B, run `54630557c081` | 50 | 34/50 | 68.0% | Depth 0.1: 4/10; 0.3: 6/10; 0.5: 8/10; 0.7: 9/10; 0.9: 7/10. Median latency about 2.43 s. |
| Qwen3 0.6B, small run `354d0aa9cca9` | 6 | 4/6 | 66.7% | Consistent with the larger run, but far too small for a precise estimate. |
| Qwen3 4B, run `63b9fdb9f911` | 120 | 120/120 | 100.0% | 4k and 8k contexts at depths 0, 0.5, 1.0; 20 cases per group. Median latency about 45.4 s; LLM-judge score was 1.0 throughout. |

This is the clearest small-model failure mode in the suite. In the 0.6B run, the model often returns a wrong neighboring code, a partial relation, a refusal despite sufficient evidence, or explanatory prose instead of the required value. The 4B model solved all tested cases, but at a much larger generation-time cost.

### 3.4 Negative retrieval: answer versus abstain

Negative retrieval is evaluated differently from NIAH. Each case is either **answerable** (a target value is present) or **unanswerable** (the correct output is exactly `NO_EVIDENCE_FOUND`). The table separates those two abilities; overall accuracy hides an important imbalance.

| Folder / run | Records | Answerable extraction | Correct abstention | Overall strict score |
|---|---:|---:|---:|---:|
| `quick_negative_retrieval` (`b0ee1fb9c668`) | 40 | 0/20 (0%) | 0/20 (0%) | 0/40 (0%) |
| `quick_negative_retrieval_tight_prompt` (`bd41b94deb6e`) | 40 | 3/20 (15%) | 20/20 (100%) | 23/40 (57.5%) |
| `quick_negative_retrieval_with_judge` (`54c6c18eeebc`) | 40 | 3/20 (15%) | 19/20 (95%) | 22/40 (55.0%) |
| `negative_retrieval_4k_8k` (`a46a1ab0866e`) | 80 | 5/40 (12.5%) | 33/40 (82.5%) | 38/80 (47.5%) |
| Extended 4k–32k run (`45c286ad0e18`) | 160 | 0/80 (0%) | 0/80 (0%) | 0/160 (0%) |

For the 80-record 4k/8k run, breakdown by nominal target length was:

| Target context | Correct extraction when answerable | Correct abstention when unanswerable |
|---:|---:|---:|
| 4k | 1/20 (5%) | 16/20 (80%) |
| 8k | 4/20 (20%) | 17/20 (85%) |

Findings:

- The original prompt failed completely under exact answerability scoring. Responses frequently explained that evidence was missing even when it was present, or invented/extracted an unrelated value when it was absent.
- The tighter prompt created a strong **abstention bias**: absent facts were handled perfectly in the 4k test, but the model still found only 3 of 20 present values. Correct refusal is not the same as good retrieval.
- With a judge enabled, the answerable extraction rate stayed at 15%; the judge did not fix the central capability gap. The judge-enabled prompt variant scored 19/20 on abstention versus 20/20 for the tight variant.
- On the 4k/8k prompt, extraction improved from 5% at 4k to 20% at 8k, but that difference is based on 20 cases per subgroup and may reflect generated case composition, placement, or prompt effects rather than a true benefit from more context.
- The 160-record extended run scored 0/160 by the same strict metric. Its response examples are consistent with both false abstention (for answerable cases) and verbose/invented responses instead of the required sentinel (for unanswerable cases).

**Conclusion for negative retrieval:** Qwen3 0.6B is not reliable at the combined task “find the exact value if present, otherwise abstain.” Prompt tightening can teach it to abstain more consistently, but does not solve extraction. Evaluate the two subproblems separately and optimize their balance.

## 4. CSV summaries: useful, but separate from JSONL evidence

The archive includes two report/CSV pairs. Their CSVs contain only context length, depth, needle, prediction, binary success, latency, and a context-length bin. They do not identify the model, run ID, schema version, prompt hash, or exact runtime configuration.

### `reports/benchmarking_raw.csv` and `reports/summary_report.md`

| Nominal context bin | Successes | Accuracy | Depth pattern | Median latency |
|---:|---:|---:|---|---:|
| 4k | 9/9 | 100.0% | 3/3 at each of depths 0, 0.5, 1.0 | 3.20 s |
| 8k | 9/9 | 100.0% | 3/3 at each of depths 0, 0.5, 1.0 | 6.13 s |
| 16k | 1/9 | 11.1% | 0/3 at depths 0 and 0.5; 1/3 at depth 1.0 | 6.75 s |
| 32k | 1/9 | 11.1% | 0/3 at depths 0 and 0.5; 1/3 at depth 1.0 | 6.49 s |

This CSV suggests a sharp failure beyond 8k. However, it conflicts with the Qwen3 0.6B `478efbee30ab` JSONL run (29/30 at 16k and 28/30 at 32k), and with some other JSONL runs. Because the CSV does not identify its model or run, this conflict cannot be resolved from the CSV alone. Treat it as a separate, currently unattributed experiment.

### `reports/2000-8000/benchmarking_raw1.csv` and `summary_report1.md`

| Nominal context bin | Successes | Accuracy | Median latency |
|---:|---:|---:|---:|
| 2k | 9/9 | 100.0% | 1.12 s |
| 4k | 9/9 | 100.0% | 3.08 s |
| 6k | 9/9 | 100.0% | 3.75 s |
| 8k | 9/9 | 100.0% | 4.04 s |

This dataset shows perfect retrieval through 8k across all 36 cases. As above, it is not attributable to a specific model from the available metadata.

## 5. Cross-cutting conclusions

### What the results support

1. **Task type matters more than nominal context length alone.** The 0.6B model handles a single inserted fact well at 8k–32k in one controlled sweep, but struggles on multi-hop linking and on the answerability/abstention task. A single NIAH score is not a useful stand-in for “long-context understanding.”
2. **Model scale appears to help on these tasks.** Qwen3 4B had perfect exact-match results on the 60-case distractor and 120-case multi-hop runs, while Qwen3 0.6B reached 95% on distractors and 68% on the main multi-hop run. This is a strong within-suite signal, but not a fully controlled model-size experiment because case counts and run designs differ.
3. **The 0.6B model shows a practical edge near 36k–40k.** In the 15-record extension, accuracy fell from 100% at 32k to 60% at 36k and 40% at 40k. This suggests degradation near the tested upper range, but needs repeated, identical cases to set a defensible boundary.
4. **Insertion position can matter.** Several failures occurred with needles near the beginning of long prompts, while late positions succeeded in the same extended run. The older Llama 3 8B run also failed disproportionately at the first and last depths. Because these patterns do not reproduce consistently across runs, they should trigger controlled follow-up tests rather than be treated as a universal positional curve.
5. **A refusal prompt can improve abstention while damaging usefulness.** The tight negative-retrieval prompt achieved 100% correct abstention on its 20 unanswerable cases, but retrieved only 3/20 answerable values. Accuracy must be reported as answerable extraction and unanswerable abstention separately.
6. **Latency rises with longer contexts and harder tasks, but these timings are run-dependent.** In the main Qwen3 0.6B NIAH run, median latency grew from 5.28 s at 8k to 19.01 s at 32k. Qwen3 4B multi-hop median latency was about 45.4 s. Different run settings and response lengths mean these values are observations, not universal throughput benchmarks.

### What the results do not yet support

- They do not establish a reliable universal maximum context length for any model. The 36k/40k run is small, and effective context configuration is not consistently verifiable.
- They do not justify saying the 4B model “solves long-context reasoning” in general. The tested cases are synthetic, structured, and generated from limited schemas/templates.
- They do not justify merging all runs into one aggregate accuracy by model. Run versions, test cases, depths, prompt hashes, metrics, generation settings, and evaluation settings differ.
- The CSV drop at 16k/32k cannot be assigned to a model without the missing run metadata.

## 6. Data-quality and evaluation caveats

### 6.1 Schema and run comparability

- JSONL records span schema versions 1.0 through 1.6. Older versions lack some fields such as prompt hashes, case fingerprints, effective token counts, or runtime settings.
- Small partial runs (one to six records) are useful as smoke tests but should not carry the same evidential weight as 50-, 90-, 120-, or 330-record runs.
- Some case designs overlap in purpose but not necessarily in exact inputs. Do not average across all 1,230 records to claim a model accuracy.

### 6.2 Context metadata needs reconciliation

There is an important internal consistency issue: several JSONL records report a `generation.num_ctx` smaller than the response `prompt_eval_count` and/or smaller than the benchmark target length. For example:

- Qwen3 0.6B NIAH run `478efbee30ab` records `num_ctx = 8704`, while response `prompt_eval_count` reaches about 32,096 for the 32k cases.
- Some nominal 4k/8k Qwen3 4B runs record `num_ctx = 4608`, while `prompt_eval_count` reaches around 8,126.
- The extended Qwen3 0.6B run `85d237650c49` records a requested context of 40,000 but `num_ctx = 32768`, and it does not include per-response `prompt_eval_count` to resolve what was actually passed/evaluated.
- The older 128k Llama 3.2 1B run has `num_ctx = 128512` but also lacks per-response `prompt_eval_count`.

This means `num_ctx` cannot be blindly treated as the effective window, and the target length cannot automatically be treated as the number of tokens seen by the model. The observed `prompt_eval_count` is useful evidence where present, but its relationship to the recorded `num_ctx` should be verified against the runner implementation. Until reconciled, mark context-length claims above the unambiguous evaluated input length as **provisional**.

### 6.3 Judge / metric reliability

- One record in the three-case Qwen3 0.6B 32k run (`0ca55bd787be`) is marked `evaluation_error` because the judge returned `100.0` where the scorer required a value in `[0, 1]`. The generated answer still exactly matched the target and its lexical/semantic metrics are 1.0. Count it as an answer match, but exclude it from judge-score summaries.
- Judge models differ across runs (for example, Qwen3 4B versus Llama 3 8B), and many runs have no judge. Judge scores should not be compared across runs as if they were one calibrated scale.
- Exact string scoring can mark a semantically correct reformatted code as wrong. Keep separate fields for normalized value correctness, strict output-format compliance, and semantic/grounded judgment.

### 6.4 Test-set limitations

- Synthetic needles and registry/access-code entities are useful for regression testing, but they do not cover realistic document structure, conflicting evidence, paraphrases, distractor density, or multi-document reasoning breadth.
- Repeated templates can make a benchmark easier or less representative than the real task. Randomize secret/entity formats, entities, and document filler; retain deterministic case seeds so runs remain comparable.
- NIAH and negative retrieval are different capabilities: a model can retrieve an inserted code but still fail to bind the right entity to the right value or decide when a query is unanswerable.

## 7. Recommended next steps

1. **Fix and validate effective-context logging first.** Record the exact API options sent to the model, effective `num_ctx`, model-native input token count, benchmark tokenizer/count, prompt hash, and whether any truncation occurred. Add an assertion/report warning when these fields disagree.
2. **Freeze one current benchmark schema and suite.** Use the same case fingerprints, prompts, tokenizer, context sizes, positions, generation settings, and evaluation settings for every model comparison. Preserve legacy files for history, but do not pool them with the controlled suite.
3. **Repeat boundary tests.** For each model, run identical cases at 8k, 16k, 24k, 32k, 36k, and 40k where supported; use several unique needles at each position; run at least three independent seeds/repeats. Report binomial uncertainty intervals, not just percentages.
4. **Score position separately.** Use a balanced grid of depths (for example 0, 0.1, 0.25, 0.5, 0.75, 0.9, 1.0) for every context size. Include the same exact cases at each position so position effects are not confounded with different needle values.
5. **Split negative retrieval into two primary scores.** Report answerable extraction recall and unanswerable correct-abstention rate separately, plus a confusion matrix. Only combine them into a headline score if the weighting is explicit. Keep `NO_EVIDENCE_FOUND` as an exact normalized sentinel.
6. **Separate correctness from formatting.** Normalize case and permitted punctuation for the secret/value comparison, while separately enforcing “value only” format. Save whether a response contains the correct value even if it fails exact-output format.
7. **Make judge evaluation robust.** Validate all judge outputs are numeric and within `[0,1]`, record parse/validation failures separately, and use one frozen judge model/prompt if cross-run judge comparisons are needed.
8. **Add more realistic tests after fixing the harness.** Include multiple facts with similarly named entities, contradictory records, paraphrased relations, distractor counts, no-answer controls, and real document passages. Retain NIAH as a smoke test, not the entire long-context evaluation.

## 8. Bottom line

**The data supports a useful but bounded conclusion:** Qwen3 0.6B is strong at synthetic single-needle retrieval up to 32k in one 90-case run, but multi-hop linkage and evidence-aware abstention are substantially harder. Qwen3 4B performed extremely well on the available structured distractor and multi-hop tests, at much higher latency. Performance appears to deteriorate near 36k–40k for the 0.6B model, although the exact boundary needs a better-controlled repeat.

Before making a strong public claim about long-context capability, resolve the runtime/context metadata inconsistency, standardize the benchmark version, and rerun the same cases across models. The highest-priority model-side weakness to investigate is **negative retrieval and entity-to-value binding**, not basic NIAH retrieval.

## Appendix A. Per-file JSONL inventory

The following inventory includes all JSONL files found in the ZIP. “Score” uses the file’s lexical exact-match metric, or answerability exact-match for negative retrieval. Do not compare score rows across different experiments as if they were one benchmark.

| Relative file | Model | Run ID | Schema | Records | Target-token range | Score |
|---|---|---|---:|---:|---:|---:|
| `raw/long_range_dependency_NIAH_llama3.2_1b_d0771fde605b.jsonl` | llama3.2:1b | `d0771fde605b` | 1.0 | 6 | 128,000–128,000 | 4/6 (66.7%) |
| `raw/long_range_dependency_NIAH_llama3_8b_2bfcd3f32b11.jsonl` | llama3:8b | `2bfcd3f32b11` | 1.0 | 330 | 7,000–8,000 | 260/330 (78.8%) |
| `raw/long_range_dependency_NIAH_llama3_8b_7de845de1d9d.jsonl` | llama3:8b | `7de845de1d9d` | 1.4 | 36 | 7,000–8,000 | 34/36 (94.4%) |
| `raw/long_range_dependency_NIAH_llama3_8b_dd952d8eb194.jsonl` | llama3:8b | `dd952d8eb194` | 1.4 | 1 | 4,000–4,000 | 1/1 (100.0%) |
| `raw/long_range_dependency_NIAH_llama3_8b_de2cecc1c5b7.jsonl` | llama3:8b | `de2cecc1c5b7` | 1.4 | 18 | 4,000–4,000 | 18/18 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_0ca55bd787be.jsonl` | qwen3:0.6b | `0ca55bd787be` | 1.0 | 3 | 32,000–32,000 | 3/3 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_244c174decc5.jsonl` | qwen3:0.6b | `244c174decc5` | 1.4 | 1 | 4,000–4,000 | 1/1 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_25b5a77e0523.jsonl` | qwen3:0.6b | `25b5a77e0523` | 1.2 | 2 | 4,000–4,000 | 2/2 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_478efbee30ab.jsonl` | qwen3:0.6b | `478efbee30ab` | 1.4 | 90 | 8,000–32,000 | 87/90 (96.7%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_6aaaec8736ab.jsonl` | qwen3:0.6b | `6aaaec8736ab` | 1.1 | 3 | 4,000–4,000 | 3/3 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_76decd0173a3.jsonl` | qwen3:0.6b | `76decd0173a3` | 1.5 | 2 | 4,000–4,000 | 2/2 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_81e709779b80.jsonl` | qwen3:0.6b | `81e709779b80` | 1.3 | 18 | 32,000–32,000 | 17/18 (94.4%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_85d237650c49.jsonl` | qwen3:0.6b | `85d237650c49` | 1.1 | 15 | 32,000–40,000 | 10/15 (66.7%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_95b95812a975.jsonl` | qwen3:0.6b | `95b95812a975` | 1.4 | 18 | 4,000–4,000 | 18/18 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_96051ea9667a.jsonl` | qwen3:0.6b | `96051ea9667a` | 1.1 | 2 | 4,000–4,000 | 2/2 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_b0f41ecd9b68.jsonl` | qwen3:0.6b | `b0f41ecd9b68` | 1.4 | 2 | 4,000–4,000 | 2/2 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_b9fc2be30af6.jsonl` | qwen3:0.6b | `b9fc2be30af6` | 1.0 | 1 | 32,000–32,000 | 1/1 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_d1b56b87ed7e.jsonl` | qwen3:0.6b | `d1b56b87ed7e` | 1.3 | 1 | 4,000–4,000 | 1/1 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_db8099fc714c.jsonl` | qwen3:0.6b | `db8099fc714c` | 1.2 | 2 | 4,000–4,000 | 2/2 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_e86bb61d049d.jsonl` | qwen3:0.6b | `e86bb61d049d` | 1.3 | 1 | 4,000–4,000 | 1/1 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_0.6b_eeb54d8c6156.jsonl` | qwen3:0.6b | `eeb54d8c6156` | 1.0 | 3 | 4,000–4,000 | 3/3 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_4b_07d9a7cc8d6b.jsonl` | qwen3:4b | `07d9a7cc8d6b` | 1.0 | 2 | 4,000–4,000 | 2/2 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_4b_55a05b8418e9.jsonl` | qwen3:4b | `55a05b8418e9` | 1.0 | 12 | 4,000–4,000 | 12/12 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_4b_6c868540c282.jsonl` | qwen3:4b | `6c868540c282` | 1.0 | 10 | 4,000–4,000 | 10/10 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_4b_77c80d5140c2.jsonl` | qwen3:4b | `77c80d5140c2` | 1.0 | 3 | 32,000–32,000 | 3/3 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_4b_af1087817b4e.jsonl` | qwen3:4b | `af1087817b4e` | 1.6 | 12 | 4,000–8,000 | 12/12 (100.0%) |
| `raw/long_range_dependency_NIAH_qwen3_4b_f163218e8378.jsonl` | qwen3:4b | `f163218e8378` | 1.0 | 10 | 4,000–4,000 | 10/10 (100.0%) |
| `raw/long_range_dependency_NIAH_distractor_qwen3_0.6b_23837261779c.jsonl` | qwen3:0.6b | `23837261779c` | 1.5 | 10 | 4,000–4,000 | 10/10 (100.0%) |
| `raw/long_range_dependency_NIAH_distractor_qwen3_0.6b_9c14a3eac905.jsonl` | qwen3:0.6b | `9c14a3eac905` | 1.6 | 20 | 4,000–4,000 | 19/20 (95.0%) |
| `raw/long_range_dependency_NIAH_distractor_qwen3_4b_aadb46540671.jsonl` | qwen3:4b | `aadb46540671` | 1.6 | 60 | 4,000–8,000 | 60/60 (100.0%) |
| `raw/long_range_dependency_NIAH_multihop_qwen3_0.6b_354d0aa9cca9.jsonl` | qwen3:0.6b | `354d0aa9cca9` | 1.5 | 6 | 4,000–4,000 | 4/6 (66.7%) |
| `raw/long_range_dependency_NIAH_multihop_qwen3_0.6b_54630557c081.jsonl` | qwen3:0.6b | `54630557c081` | 1.5 | 50 | 4,000–4,000 | 34/50 (68.0%) |
| `raw/long_range_dependency_NIAH_multihop_qwen3_4b_63b9fdb9f911.jsonl` | qwen3:4b | `63b9fdb9f911` | 1.6 | 120 | 4,000–8,000 | 120/120 (100.0%) |
| `negative_retrieval_4k_8k/raw/long_context_understanding_negative_retrieval_qwen3_0.6b_a46a1ab0866e.jsonl` | qwen3:0.6b | `a46a1ab0866e` | 1.6 | 80 | 4,000–8,000 | 38/80 (47.5%) |
| `raw/long_context_understanding_negative_retrieval_qwen3_0.6b_45c286ad0e18.jsonl` | qwen3:0.6b | `45c286ad0e18` | 1.6 | 160 | 4,000–32,000 | 0/160 (0.0%) |
| `raw/quick_negative_retrieval/raw/long_context_understanding_negative_retrieval_qwen3_0.6b_b0ee1fb9c668.jsonl` | qwen3:0.6b | `b0ee1fb9c668` | 1.6 | 40 | 4,000–4,000 | 0/40 (0.0%) |
| `raw/quick_negative_retrieval_tight_prompt/raw/long_context_understanding_negative_retrieval_qwen3_0.6b_bd41b94deb6e.jsonl` | qwen3:0.6b | `bd41b94deb6e` | 1.6 | 40 | 4,000–4,000 | 23/40 (57.5%) |
| `raw/quick_negative_retrieval_with_judge/raw/long_context_understanding_negative_retrieval_qwen3_0.6b_54c6c18eeebc.jsonl` | qwen3:0.6b | `54c6c18eeebc` | 1.6 | 40 | 4,000–4,000 | 22/40 (55.0%) |

## Appendix B. Explicitly non-`ok` response status

Only one response was explicitly marked non-`ok`:

| File / run | Case | Status | Details |
|---|---|---|---|
| `raw/long_range_dependency_NIAH_qwen3_0.6b_0ca55bd787be.jsonl`, run `0ca55bd787be` | nominal 32k, depth 0.5 | `evaluation_error` | The judge emitted `100.0` instead of a score in `[0,1]`; prediction exactly matched the expected value. Treat the lexical result as correct but the judge result as failed. |

---

*End of report.*
