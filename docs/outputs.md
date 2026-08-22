# Outputs

Everything a run produces goes under `meta.output_dir`.

```
<output_dir>/
  optimization_summary.md    the run report
  recommended.yaml           the recommended pipeline configuration
  frontier/trial_NN.yaml     one file per Pareto frontier member
  exam.json                  the exam, generated once and reused
  run.log                    full log of the run, including every agent prompt and reply
  details/
    history.jsonl            one record per completed trial
    cost_breakdown.json      LLM spend of the whole run by category
    trial_cost_ledger.jsonl  LLM spend per trial by category
    candidates.json          every candidate question, with the reason for each rejection
    exam_cost.json           cost of exam generation, replayed on cached runs
    debug/                   exam-generation audits (composition log, span verification, probe audit)
  .cache/                    parsed corpus, corpus sample, duplicate clusters, chunks and embeddings
  lightrag/                  the knowledge graph, only with graph index types
  vllm_<model>.log           vLLM server logs, only with hosted_vllm models
```

## The report

`optimization_summary.md` opens with the recommended trial, its accuracy and cost per query, and a one-line run summary. Two sections are written by the optimizer model: `Recommendation` explains why that trial, and `What the search found` summarizes the trajectory in a few sentences. Everything else is generated from the data: the frontier table, an accuracy-versus-cost chart, the tradeoffs between neighbouring frontier members, and the full YAML of every frontier member. With `meta.cost_aware: false` the frontier sections are replaced by a table of trials sorted by accuracy. `--skip-final-report` skips the model-written sections, and the recommendation falls back to the highest-accuracy trial.

## Configurations

`recommended.yaml` and every `frontier/trial_NN.yaml` are complete pipeline configurations: chunking, embedding model, index type, top-k, reranker, expansion, compression, generator, temperature, and reasoning. Two comment lines at the top give the trial number, accuracy, and cost per query.

## History

`details/history.jsonl` has one JSON object per completed trial with the configuration, every question's result (prediction, correctness, retrieved chunks, failure mode), the aggregate metrics (`answer_accuracy`, exact match, F1, retrieval complete, partial, and miss counts, cost and token totals), whether the trial is on the frontier, the diagnosis, and the proposal rationale. Failed trials have no record. Convert it to a JSON array with `scripts/jsonl_to_json.py` for analysis.

## Cost

Both cost files use the same categories.

| Category | What it covers |
| --- | --- |
| `rag_eval` | Every pipeline LLM call made while answering exam questions |
| `exam_generation` | Composer, validation, and probe calls |
| `judge` | Answer grading and failure attribution during trials |
| `agent_proposal` | Diagnoser and proposer calls |
| `final_report` | The report and the recommendation |
| `graph_build` | Graph extraction, only with graph index types |
| `ground_exam` | The `ground-exam` command |
| `embedding_build` | Embedding token counts. Local embedders bill nothing |

Each category records dollars, prompt and completion tokens, cache tokens, and call counts. Dollars come from LiteLLM's price table and are zero for models it does not price.

## Re-running and cleaning

A second `optimize` on the same config reuses the parsed corpus, the chunk and embedding cache, the graph, and the exam. It resets `history.jsonl` and `run.log` and starts the trials from 1. `trial_cost_ledger.jsonl` and `debug/cache_events.jsonl` are appended, so their earlier lines survive.

```bash
uv run agentic-autorag clean --config configs/my_project.yaml
```

deletes `.cache/`, `details/`, `exam.json`, `recommended.yaml`, `optimization_summary.md`, `frontier/`, and `run.log` after a confirmation prompt (`--yes` skips it). It leaves `lightrag/` and the vLLM logs in place. Delete `lightrag/` by hand when the corpus or the graph settings change.
