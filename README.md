<p align="center">
  <img src="https://raw.githubusercontent.com/lassebaerlandstrand/Agentic-AutoRAG/HEAD/assets/Agentic_AutoRAG.png" alt="Agentic AutoRAG">
</p>

# Agentic AutoRAG

Agentic AutoRAG is a reasoning-driven optimizer for Retrieval-Augmented Generation (RAG) pipelines. Instead of grid search or Bayesian optimization, it runs a two-stage LLM agent loop: a diagnoser analyses why a trial configuration fails, and a proposer chooses what to change next from that diagnosis and the history of prior trials. The optimization signal is a synthetic exam, open-ended questions with ground-truth answers generated from your corpus on the first run and cached for reuse. Retrieval is database-agnostic: vector, hybrid BM25 plus vector, and, as an experimental option, graph or hybrid graph plus vector.

## How it works

1. **Exam generation.** On the first run the examiner model reads your corpus and writes an exam of typed questions (extraction, definitional, inference, bridge, comparison, and two numeric types) paired with answers grounded in verbatim source spans. Every question is validated and run through a ladder of probe pipelines, and the exam keeps the questions that separate weak configurations from strong ones. The exam is cached to `exam.json`.
2. **Reasoning loop.** Each trial builds the proposed pipeline and evaluates it against the exam. The diagnoser explains why the configuration scored as it did (retrieval misses versus generation errors, which question types failed). The proposer then picks the next configuration from the diagnosis, the full trial history, and a knowledge base of model benchmarks and prices.
3. **Selection.** In cost-aware mode trials are scored on answer accuracy and LLM cost per query. The non-dominated trials form a Pareto frontier, and the optimizer model picks the recommended configuration from it, capable and cheap rather than top score at any price. Every frontier member is written out as a ready-to-run configuration.

The details are in [docs/how_it_works.md](docs/how_it_works.md).

## Setup

Requirements: Python 3.12+ and [uv](https://docs.astral.sh/uv/).

```bash
uv sync                  # runtime dependencies
uv sync --extra dev      # add tests, lint, and vLLM
```

Copy `.env.example` to `.env` and fill in the keys for the providers you use. Only the keys for models in your config are checked, and a missing one is reported by name at startup.

| Provider prefix   | Env vars                                                          |
| ----------------- | ----------------------------------------------------------------- |
| `openai/...`      | `OPENAI_API_KEY`                                                  |
| `anthropic/...`   | `ANTHROPIC_API_KEY`                                               |
| `gemini/...`      | `GEMINI_API_KEY`                                                  |
| `mistral/...`     | `MISTRAL_API_KEY`                                                 |
| `cohere/...`      | `COHERE_API_KEY`                                                  |
| `azure/...`       | `AZURE_API_KEY`, `AZURE_API_BASE`                                 |
| `azure_ai/...`    | `AZURE_AI_API_KEY`, `AZURE_AI_API_BASE`                           |
| `vertex_ai/...`   | `VERTEXAI_PROJECT`, `VERTEXAI_LOCATION`                           |
| `bedrock/...`     | `AWS_REGION_NAME` plus access keys, `AWS_PROFILE`, or an IAM role |
| `ollama/...`      | none. Start `ollama serve` and `ollama pull` each model            |
| `hosted_vllm/...` | none. vLLM is started for you (install via `uv sync --extra dev`)  |

Azure note: `AZURE_API_BASE` is `https://<resource>.cognitiveservices.azure.com/` or `https://<resource>.openai.azure.com/`. `AZURE_AI_API_BASE` is `https://<resource>.services.ai.azure.com/models`. If Azure returns one shared key, use it for both `AZURE_API_KEY` and `AZURE_AI_API_KEY`.

Sanity-check your environment:

```bash
uv run agentic-autorag info
```

## Corpus

Point `meta.corpus_path` at a directory of documents. Supported formats: PDF, DOCX, XLSX, PPTX, HTML, CSV, Markdown, plain text, AsciiDoc, and images (PNG, JPG, TIFF, BMP, WEBP, through OCR). Subdirectories are walked recursively, one file per source document.

The example configs use `./data/corpus/unidoc/`. Download it with:

```bash
uv run python scripts/download_unidoc_corpus.py
```

## Configure and run

Copy `configs/starter_example.yaml`, set the corpus path, the models that drive the optimizer, and the search space, then run:

```bash
uv run agentic-autorag optimize --config configs/my_project.yaml
```

To inspect the exam before spending on trials:

```bash
uv run agentic-autorag generate-exam --config configs/my_project.yaml
```

To start over on a clean output directory:

```bash
uv run agentic-autorag clean --config configs/my_project.yaml
```

## Outputs

Everything is written under `meta.output_dir`. The files you will read are `optimization_summary.md` (the report, with the frontier table and the reasons for the recommendation), `recommended.yaml` (the recommended pipeline configuration), `frontier/` (one YAML per frontier member), and `exam.json`. Trial history, cost ledgers, and exam audits live under `details/`. See [docs/outputs.md](docs/outputs.md).

## Documentation

- [docs/how_it_works.md](docs/how_it_works.md): the three phases, the reasoning loop, scoring, and how the recommendation is chosen.
- [docs/exam_generation.md](docs/exam_generation.md): how the exam is built, the question types, the neighborhood weighting and when to change it, every `examiner` setting.
- [docs/configuration.md](docs/configuration.md): every config section and field, providers and credentials, the knowledge base.
- [docs/search_space.md](docs/search_space.md): every dimension the optimizer can tune, its options, and its constraints.
- [docs/outputs.md](docs/outputs.md): the output tree, the report, history and cost files, re-running, and cleaning.
- [docs/custom_exam.md](docs/custom_exam.md): optimizing against your own questions.

## License

MIT. See [LICENSE](LICENSE).
