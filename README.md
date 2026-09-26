# Clef

**Clef** is a family of small open-weights language models that are experts in symbolic music. It is built as a learning project: a reproducible data pipeline, a benchmark with programmatic checkers, fine-tuned models, and an honest write-up of what worked.

The first model is a **notation expert**: it reads, checks, transforms and converts scores in a text format. Later models cover comprehension and harmony, arrangement, and composition.

**Status:** planning. See [docs/PLAN.md](docs/PLAN.md) for goals, tasks, benchmark design, data sources, training approach and the roadmap. No results yet.

## Layout

| Path | Purpose |
|---|---|
| `docs/` | The plan, and later the data card, model card and experiment notes |
| `eval/` | Benchmark, verifiers and baseline runners |
| `data/` | Scripts and configs that build datasets (raw data is not committed) |
| `train/` | Training configs and scripts |
| `serve/` | Export and serving scripts |
| `experiments/` | Run logs and result tables |
| `tests/` | Tests for parsers, generators and verifiers |

## Next steps

Phase 0 and 1 of the plan: choose and license-check two or three corpora, build the verifiers and the first benchmark, and measure baselines before any training.

## License

Not chosen yet.
