<p align="center">
  <img src="https://raw.githubusercontent.com/Agentic-Systems-Lab/Agentic-AutoRAG/HEAD/assets/Agentic_AutoRAG.png" alt="Agentic AutoRAG">
</p>

<p align="center">
  <a href="https://arxiv.org/abs/2610.08452"><img alt="arXiv" src="https://img.shields.io/badge/arXiv-2610.08452-b31b1b.svg"></a>
  <a href="https://github.com/Agentic-Systems-Lab/Agentic-AutoRAG/blob/main/LICENSE"><img alt="License: MIT" src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
  <img alt="Python 3.12+" src="https://img.shields.io/badge/python-3.12%2B-blue.svg">
</p>

# Agentic AutoRAG

Agentic AutoRAG tunes a Retrieval-Augmented Generation (RAG) pipeline for your documents. You give it a corpus and a search space, the options it may pick from for chunking, embedding, retrieval, reranking, generation and more. It returns a recommended configuration and the Pareto frontier of configurations that trade answer accuracy against cost per query.

LLM agents run the search. Each trial builds one pipeline and scores it on an exam that the tool writes from your corpus, with answers grounded in the source text. A diagnoser explains why the pipeline missed the questions it missed. A proposer reads that diagnosis and the earlier trials and chooses what to change next. Any provider that LiteLLM supports can supply the models, including local ones through Ollama or vLLM.

## News

- **2026-10:** The paper is on [arXiv](https://arxiv.org/abs/2610.08452).
- **2026-09:** Accepted at the REALM workshop at EMNLP 2026 and the ML for Systems workshop at NeurIPS 2026.

## How it works

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/Agentic-Systems-Lab/Agentic-AutoRAG/HEAD/assets/architecture-dark.png">
  <img alt="Architecture of Agentic AutoRAG" src="https://raw.githubusercontent.com/Agentic-Systems-Lab/Agentic-AutoRAG/HEAD/assets/architecture-light.png">
</picture>

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

## Reproducing the paper

Benchmark repository: [lassebaerlandstrand/Agentic-AutoRAG-Bench](https://github.com/lassebaerlandstrand/Agentic-AutoRAG-Bench)

## Citation

```bibtex
@misc{strand2026agenticautorag,
  title         = {Agentic AutoRAG: RAG Pipeline Optimization through Reasoning-Driven Agents},
  author        = {Lasse B. Strand and Robert Jakob and Kevin O'Sullivan and Markus Kreft},
  year          = {2026},
  eprint        = {2610.08452},
  archivePrefix = {arXiv},
  primaryClass  = {cs.CL},
  url           = {https://arxiv.org/abs/2610.08452}
}
```

## License

MIT. See [LICENSE](LICENSE).
