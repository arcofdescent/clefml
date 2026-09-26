# Notation Expert: Plan

A plan for building, evaluating and releasing small open-weights language models that are experts in symbolic music. The first model is a **notation expert**. Later models cover comprehension, harmony, arrangement and composition.

This is a learning and portfolio project. The goal is to do each step properly and to report honestly what worked and what did not, not to beat a frontier model.

Status: draft. Everything here is a proposal to be tested, not a commitment. Facts about datasets, tools and licenses must be re-checked when used.

## 1. Goals and non-goals

**Goals**

- A reproducible pipeline from public-domain scores to verified training examples.
- A public benchmark of notation tasks with programmatic checkers, and a held-out test set.
- A fine-tuned small model that measurably beats its own base model on that benchmark, with ablations that explain why.
- A released dataset (or its build scripts), model weights, and a model card that states the limits.

**Non-goals (for the first model)**

- Beating hosted frontier models overall.
- Generating pleasing music. That is later work and needs different evaluation.
- Audio, or reading scanned scores (OMR). This is symbolic music only.

## 2. Principles

1. **Evaluation first.** The benchmark exists, and baselines are measured, before any training.
2. **Verifiable tasks.** Prefer tasks whose answers code can check. Notation has many: durations must sum to the meter, output must parse, a transposition can be recomputed.
3. **Programmatic data over model-generated data.** Generate examples from real scores with code so answers are correct by construction. Use a teacher model only where code cannot (natural-language explanations), and check its terms first.
4. **No leakage.** Split by piece and by composer or collection, never by example. Test data is never used for tuning.
5. **Reproducible.** Fixed seeds, versioned configs, pinned dependencies, logged runs.
6. **Honest reporting.** Report failures, variance across seeds, and what the model cannot do.

## 3. Scope of model 1: what "notation expert" means

A model that reads and writes symbolic notation reliably. Concretely, given a score (or a fragment) in a text format and an instruction, it produces a correct answer or a correct transformed score, and says so when the request cannot be done.

It is **not** required to understand why the music works. That is model 2.

## 4. Representation

The first design decision, because it fixes token cost and difficulty.

| Format | For | Against |
|---|---|---|
| MusicXML | The standard interchange; rich | Very verbose, so it is expensive in tokens and easy to get wrong |
| ABC | Compact; models have seen it | Limited for multi-voice, piano and complex scores |
| Humdrum \*\*kern | Compact; strong for analysis; a large scholarly corpus | Less familiar to general models |
| A custom compact text (as the Maestro editor's compact notation) | Designed for this task | New to any base model, so more to learn |
| MIDI-like event text | Simple | Loses notation (spelling, beams, voices) |

**Plan:** run a small, early experiment. Take one task, such as transposition, and measure a base model zero-shot and after a small tune on each of two or three formats. Choose the primary format from that evidence. Keep conversion between formats as a task in its own right.

## 5. Task taxonomy

Each task has a generator (builds examples from real scores), a verifier (scores an answer), and a difficulty tier.

| ID | Task | Input → output | Verifier |
|---|---|---|---|
| T1 | Read | Score + question → answer (key, meter, bar count, notes in a bar, highest note) | Compare with values computed by a parser |
| T2 | Transform | Score + instruction → score (transpose by interval, change clef, halve or double durations, change key) | Parse, recompute the transformation, compare musically |
| T3 | Validate | Score with an injected error → location and fix (bar does not add up, note out of range, wrong accidental) | Injected errors are known; check the fix restores validity |
| T4 | Convert | Score in format A → format B | Parse both, compare musical content; round-trip |
| T5 | Edit | Score fragment + natural-language edit → edited fragment (move a voice, merge tied notes, delete a range) | Recompute the edit with code; compare |
| T6 | Describe | Fragment → factual description (rhythm pattern, contour, texture) | Check stated facts against values extracted by code |
| T7 | Refuse | Impossible or ambiguous instruction → a question or a reason, no change | Output leaves the score unchanged and says why |

Tiers: **easy** (one bar, one voice), **medium** (several bars, two voices), **hard** (long, multi-voice, unusual meter or key). Each task is reported per tier.

T5 and T7 mirror the Maestro editor's assistant, so the same cases can later be used to compare with it. Keep that as a separate, optional track: **tool-use** (calling an editor's tools) is different from **writing notation** and should not be mixed into model 1's first results.

## 6. Benchmark design

- **Splits:** by piece and by collection. Reserve whole collections or composers for test, so success cannot come from memorising a piece.
- **Sizes:** enough cases per task and tier that differences are measurable. Report confidence intervals, and repeated runs with different seeds for anything sampled.
- **Metrics:** parse validity, exact match where the answer is unique, musical equivalence (compare parsed content, not text) where it is not, and pass@k for sampled outputs.
- **Baselines:** the base model zero-shot and few-shot, a larger open model, and a hosted frontier model.
- **Hygiene:** deduplicate near-duplicate pieces across splits (public corpora contain many arrangements of the same tune). Check that test items are not in the base model's likely training data where that can be estimated, and note it as a limit where it cannot.

## 7. Data

**Candidate sources.** Each needs a license and terms check before use, and its current availability confirmed.

- PDMX, a large collection of public-domain MusicXML scores.
- OpenScore collections (for example Lieder and string quartets).
- KernScores (Humdrum \*\*kern).
- Nottingham and other ABC folk-tune collections.
- Lakh MIDI and similar, for MIDI-derived material (weaker as notation).

**Pipeline.** Ingest → normalise → filter (parse cleanly, sane length, remove duplicates) → generate tasks → verify every example with its checker → deduplicate → split by piece → write a data card.

**Start small.** A few thousand verified examples across all tasks is enough to learn what works. Scale only when an experiment shows more data helps.

**Tools.** music21 or partitura for parsing and analysis; Verovio or LilyPond to render scores for inspection.

## 8. Training

**Base model.** A small open-weights model (roughly 1B to 9B parameters). Choose on license, tokenizer behaviour on the chosen format, and how it does zero-shot on the benchmark. Compare two or three.

**Method.** Supervised fine-tuning with LoRA or QLoRA. Sweep only a few settings (rank, learning rate, epochs, data size) and record all of them.

**Tooling options.** MLX (mlx-lm) on an Apple-silicon Mac; Hugging Face TRL and PEFT, or Unsloth or Axolotl, on a rented GPU. Pick one stack and stay with it.

**Compute.** Use the Mac for the data pipeline and small runs. Rent a GPU for anything larger. Track cost as part of the report.

**Tracking.** Log every run (Weights & Biases or plain files) with its config, seed, data version and benchmark scores.

## 9. Later training stages (optional, in this order)

1. **Rejection sampling.** Sample many answers from the tuned model, keep those that pass the verifier, and tune on them.
2. **Preference tuning (DPO).** Pairs of passing and failing answers from the verifier.
3. **RL with verifiable rewards (for example GRPO)** on the checkable tasks, using the verifiers as the reward.
4. **Distillation from a teacher** for the natural-language parts (T6 explanations). Check the teacher's terms before releasing a dataset built from its output; an open-weights teacher with a permissive license avoids the question.

## 10. Ablations to run

- Data representation (Section 4).
- Data size (for example 1k, 5k, 20k examples).
- Task mix (all tasks together against each alone).
- LoRA rank and full fine-tuning at small scale.
- With and without verified-only filtering.
- Base model choice.

## 11. Serving

Export to GGUF for Ollama and llama.cpp, or run through MLX. Quantise and re-measure the benchmark after quantisation, since it can change results.

## 12. Repository layout

```
README.md            what it is, results table, how to reproduce
docs/                this plan, data card, model card, experiment notes
eval/                benchmark, verifiers, baseline runners
data/                build scripts and configs (not the raw data)
train/               training configs and scripts
serve/               export and serving scripts
experiments/         run logs and result tables
tests/               tests for parsers, generators and verifiers
```

Test the verifiers and generators heavily: a wrong checker silently corrupts every result.

## 13. Roadmap

Each phase has an exit criterion. Do not start the next phase before it is met.

| Phase | Work | Exit criterion |
|---|---|---|
| 0 | Repo, environment, choose 2-3 candidate corpora, check licenses | Data can be ingested and parsed reproducibly |
| 1 | Verifiers and generators for T1-T3; benchmark v0; baseline results | Baselines reported with confidence intervals |
| 2 | Representation experiment (Section 4) | Primary format chosen with evidence |
| 3 | Data pipeline for all tasks; first SFT run | Tuned model beats its base on held-out test |
| 4 | Ablations; error analysis; add T4-T7 as needed | Written findings on what helped and what did not |
| 5 | Optional: rejection sampling, DPO, RLVR | Measured gain, or a documented negative result |
| 6 | Release: weights, data build scripts, model card, write-up | Someone else can reproduce the main table |

## 14. Risks

| Risk | Mitigation |
|---|---|
| Wrong verifier gives wrong results | Unit-test verifiers with known-good and known-bad cases; spot-check by hand and by rendering |
| Test leakage | Split by piece and collection; near-duplicate detection; freeze the test set early |
| License problems in data | Check every source; record it in the data card; prefer public domain or permissive |
| Tokenizer handles the format badly | Include tokenizer behaviour in base-model and representation experiments |
| Model memorises pieces | Reserve unseen collections for test; report seen versus unseen |
| Too much scope | Ship phases 0-4 before considering later stages or other models |
| Results that do not replicate | Multiple seeds; report variance; pin dependencies |

## 15. Future models

- **Comprehension and harmony.** Chord labelling, cadences, key changes and voice-leading checks. Partly verifiable against analysed corpora, though human annotation can disagree, so treat labels as noisy.
- **Arrangement.** Reduce or expand a score. Checks can cover playability, range and voice-leading rules, but quality is subjective.
- **Composition.** Little can be checked by code. It needs human or model judges, and its evaluation is a research problem.

For each, define the task suite and benchmark first, exactly as for model 1.

## 16. Open questions

- Which representation wins for a small model (Section 4)?
- What is the smallest base model that is useful?
- How much do the verifiable tasks transfer to the fuzzy ones?
- Does one multi-task model beat separate task-specific adapters?
- Can the same benchmark cases compare with the Maestro editor's assistant fairly?

## 17. Portfolio deliverables

- The benchmark, open and documented, with a results table.
- The dataset build scripts, plus a data card.
- Model weights with a model card that states limits.
- A write-up covering the representation experiment, ablations and failures.
- Reproduction instructions that work from a clean checkout.
