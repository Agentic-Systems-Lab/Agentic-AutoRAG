# Configuration

A project is one YAML file. `configs/starter_example.yaml` is the minimal starting point, `configs/full_example.yaml` lists every field with its default, and `agentic_autorag/config/models.py` is the schema. Invalid configs fail at parse time with the field path and the reason.

Top-level sections: `meta`, `parsing`, `agent`, `examiner`, `search_space`, `model_aliases`, `graph`, `vllm`. Only `search_space` and `agent` are required. Unknown keys at the top level are ignored. Unknown keys inside `search_space` are an error, so a typo there fails loudly.

## `meta`

| Field | Default | Meaning |
| --- | --- | --- |
| `project_name` | `my-rag-project` | Name used in reports and as the seed for anchor sampling. |
| `corpus_path` | `./data/corpus/` | Directory of source documents, walked recursively. |
| `corpus_description` | `""` | Free text shown to the optimizer when it proposes the first trial. Describe the domain, the document types, and who asks the questions. |
| `output_dir` | `./experiments/` | Where every artifact and every cache goes. |
| `max_trials` | `30` | Number of trials. The only stopping condition. |
| `cache_max_gb` | `5.0` | Size cap for the chunk and embedding cache. The least recently used entries are evicted. |
| `cost_aware` | `true` | `true`: optimize accuracy and cost per query, and show the frontier to the agent. `false`: accuracy only, cost is recorded but hidden from the agent. |
| `failure_sample_seed` | `null` | Seed for the failure sample shown to the diagnoser. `null` derives it from the trial number. |
| `corpus_word_budget` | `2000000` | Word cap for the corpus. Files are sampled until the cap is reached. `null` disables it. |
| `corpus_sample_seed` | `42` | Seed for that sample. |
| `hv_delta_window` | `3` | Trials over which the frontier's hypervolume change is reported to the agent. Informational. |

## `parsing`

Not part of the search space. Set once per project.

| Field | Default | Meaning |
| --- | --- | --- |
| `parser` | `docling` | The only parser. |
| `ocr` | `true` | OCR for scanned PDFs and images. Turn off for text-only corpora to parse faster. |
| `table_structure` | `true` | Recover table structure in PDFs. |
| `near_duplicate_threshold` | `0.85` | Containment ratio above which two documents count as duplicates. |
| `near_duplicate_detection_enabled` | `true` | Detect duplicates. Affects question writing only. Every document stays in the index. |

## `agent`

The models that run the optimizer, not the models it chooses between.

| Field | Default | Meaning |
| --- | --- | --- |
| `optimizer_model` | required | Proposes configurations, diagnoses trials, writes the final report, and picks the recommendation. |
| `examiner_model` | required | Writes the exam questions. |
| `judge_model` | required | Checks answerability during exam generation and grades trial answers. Choose one at least as strong as the generators in the search space. |
| `optimizer_reasoning_effort` | `medium` | `low`, `medium`, `high`, or `null`. Dropped for models without reasoning. |
| `examiner_reasoning_effort` | `medium` | Same, for the examiner. |
| `concurrency` | `10` | Parallel LLM calls. Lower it when your provider rate-limits. |

## `examiner`

Exam size, neighborhood settings, and validation thresholds. See [exam_generation.md](exam_generation.md) for every field. The two you set most often are `exam_size` (default `80`) and `custom_exam_path` (default `null`).

## `search_space`

What the optimizer may change. See [search_space.md](search_space.md).

## `model_aliases`

Maps a name used anywhere in the config to the model string called at runtime. Use it for Azure deployment names or for models that need their own endpoint.

```yaml
model_aliases:
  azure/gpt-4o-mini: azure/gpt-4o-mini-deployment-1
  my-local-model:
    model: hosted_vllm/Qwen/Qwen2.5-7B-Instruct
    api_base: http://localhost:8000/v1
```

A value is either a model string or a mapping with `model` and optional `api_base`, `api_key`, `api_version`.

## `graph` (experimental)

Graph retrieval is experimental. No example config uses it and it receives far less testing than vector and hybrid retrieval, so expect rough edges. The section is needed when `search_space.retrieval.index_types` includes `graph_only` or `hybrid_graph_vector`. The knowledge graph is built once by LightRAG before the first trial and cached under `lightrag/` in the output directory. It is fixed for the run, not searched.

| Field | Default | Meaning |
| --- | --- | --- |
| `extraction_model` | required | LLM that extracts entities and relations. |
| `embedding_model` | `sentence-transformers/all-MiniLM-L6-v2` | Embedder for the graph's own vector stores. |
| `chunk_token_size`, `chunk_overlap_token_size` | `null` | LightRAG chunking. `null` uses the LightRAG defaults. |
| `entity_types` | `null` | Restrict extraction to these entity types. |
| `max_parallel_insert`, `llm_model_max_async`, `embedding_func_max_async` | `2`, `4`, `8` | Build concurrency. |
| `llm_model_max_retries`, `default_llm_timeout`, `default_embedding_timeout` | `3`, `180`, `30` | Retries and timeouts in seconds. |
| `extraction_call_timeout_s`, `extraction_retry_backoff_base_s`, `extraction_retry_backoff_max_s` | `45.0`, `5.0`, `30.0` | Per-call extraction timeout and backoff. |
| `build_batch_size`, `embedding_batch_size` | `20`, `64` | Documents per insert batch, texts per embedding batch. |

Changing `extraction_model`, `embedding_model`, the chunk sizes, or `entity_types` invalidates the cached graph. The build refuses to reuse a graph built with different settings, so delete `lightrag/` to rebuild. Concurrency and timeout fields do not invalidate it. `agentic-autorag clean` does not delete `lightrag/`.

## `vllm`

Read only when a `hosted_vllm/...` model appears in the config. The server is started as a subprocess before the first use and swapped when a trial needs another model, so keep the number of different `hosted_vllm/` models small.

| Field | Default | Meaning |
| --- | --- | --- |
| `max_model_len` | `null` | Context length. `null` lets vLLM read it from the model. |
| `gpu_memory_utilization` | `0.90` | Fraction of GPU memory vLLM may use. |
| `enforce_eager` | `true` | Skip CUDA graphs for faster model swaps. |
| `port` | `8000` | Server port. |
| `startup_timeout` | `180` | Seconds to wait for the server. |
| `extra_args` | `[]` | Extra `vllm serve` arguments, for example `["--reasoning-parser", "qwen3"]`. |
| `binary` | `vllm` | The vLLM executable. Install it with `uv sync --extra dev`. |

## Providers and credentials

Model strings use the LiteLLM prefix convention. At startup the required variables are checked for every model in the config, and the run stops with the name of the missing variable. Put them in `.env` (see `.env.example`).

| Prefix | Variables |
| --- | --- |
| `openai/` | `OPENAI_API_KEY` |
| `anthropic/` | `ANTHROPIC_API_KEY` |
| `gemini/` | `GEMINI_API_KEY` |
| `mistral/` | `MISTRAL_API_KEY` |
| `cohere/` | `COHERE_API_KEY` |
| `azure/` | `AZURE_API_KEY`, `AZURE_API_BASE` |
| `azure_ai/` | `AZURE_AI_API_KEY`, `AZURE_AI_API_BASE` |
| `vertex_ai/` | `VERTEXAI_PROJECT`, `VERTEXAI_LOCATION`, plus application default credentials |
| `bedrock/` | `AWS_REGION_NAME`, with either `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, or `AWS_PROFILE`, or an attached IAM role |
| `ollama/` | none. Start `ollama serve` and pull each model yourself. Reasoning mode is not available for Ollama models. |
| `hosted_vllm/` | none. The server is managed for you, see `vllm` above. |

Two checks run before the first trial. If a model string is not in the LiteLLM catalog, the config parser sends it a one-token request to confirm it exists. Then every model is pinged once, and a success is cached for 30 days in `~/.cache/agentic-autorag/llm_verification.json`. Failures are not cached. After you fix an endpoint, run with `--force-verify` (or set `AGENTIC_AUTORAG_FORCE_VERIFY=1`) to ping again.

## The knowledge base

The optimizer does not choose models blind. `knowledge_base/` holds benchmark scores, prices, throughput, and context limits for several hundred LLMs, retrieval and reranking scores for several hundred embedding models, a curated table of rerankers, and a description of every search-space parameter. The agent sees the rows for the models in your search space, and the exam generator uses the same rankings to build its probe ladder.

You do not edit the knowledge base to add a model. Add the model string to the search space and the matching row is looked up by name. A model without a row still appears to the agent, without scores. Rerankers outside the curated table have no scores and are ranked last for probes.

To see the block the agent reads for your config:

```bash
uv run python scripts/show_knowledge_base.py configs/my_project.yaml
```

To refresh the LLM and embedding tables from their sources (needs `ARTIFICIAL_ANALYSIS_API_KEY`):

```bash
uv run python scripts/build_knowledge_base.py
```

The knowledge base is read from the repository checkout, so run the optimizer from a clone of the repo.
