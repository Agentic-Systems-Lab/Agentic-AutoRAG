# Search space

`search_space` lists what the optimizer may change, grouped by pipeline stage. Every trial is one pick from each dimension. A dimension with a single allowed value is pinned: the agent never sees it and every trial uses that value.

Numeric dimensions take either a range or a grid:

```yaml
top_k: { min: 3, max: 20 }
chunk_token_size: { values: [128, 256, 384, 512] }
```

`temperature` takes a range only.

Changing chunking or the embedding model rebuilds the index, so the first trial with a new chunking and embedder pair pays for embedding the corpus. Chunks and embeddings are cached per pair under `.cache/`, and every later trial with the same pair reuses them. Everything else (index type, top-k, reranker, expansion, compression, generator) swaps without rebuilding.

## Chunking

| Key | Values | Default |
| --- | --- | --- |
| `chunking.strategies` | `recursive`, `fixed` | `[recursive]` |
| `chunking.chunk_token_size` | tokens per chunk, in the embedding model's tokenizer | `[128, 256, 384, 512]` |
| `chunking.chunk_token_overlap` | tokens shared by consecutive chunks | `[0, 32, 48, 64, 128]` |

`recursive` splits on paragraphs, then lines, then words, so chunks end at natural boundaries. `fixed` splits on lines and words only. The overlap must be smaller than the chunk size, and the chunk size must fit the embedding model's token limit, which the agent reads from the knowledge base.

## Embedding

| Key | Values | Default |
| --- | --- | --- |
| `embedding.models` | Hugging Face model ids loadable by Sentence Transformers | required |

Models run locally in half precision. Each one you add costs one embedding pass over the corpus per chunking setting the optimizer tries with it.

## Retrieval

| Key | Values | Default |
| --- | --- | --- |
| `retrieval.index_types` | `vector_only`, `hybrid_bm25_vector`, `graph_only`, `hybrid_graph_vector` | `[vector_only]` |
| `retrieval.top_k` | chunks fetched from the index | `3` to `20` |
| `retrieval.hybrid_alpha` | weight of the vector side in hybrid retrieval, `0.0` is pure BM25 and `1.0` is pure vector | `0.0` to `1.0` |
| `retrieval.bm25_vector_fusion` | `alpha`, `rrf` | `[alpha]` |
| `retrieval.long_context_reorder` | `true`, `false` | `[false]` |

`vector_only` is nearest-neighbour search over the embeddings. `hybrid_bm25_vector` adds a BM25 full-text index and fuses the two result lists, either as a weighted blend of normalized scores controlled by `hybrid_alpha`, or by reciprocal rank fusion (`rrf`), which has no parameter. `hybrid_alpha` is used only with `alpha` fusion. The graph types query a LightRAG knowledge graph and need a `graph:` block, see [configuration.md](configuration.md). `hybrid_graph_vector` merges graph results with vector results by reciprocal rank fusion.

`long_context_reorder` repeats the top-scored passage at the end of the context so it sits next to the question. It does nothing when a compressor has collapsed the context to one passage.

## Query expansion

| Key | Values | Default |
| --- | --- | --- |
| `query_expansion.strategies` | `none`, `hyde`, `multi_query`, `query_decompose` | `[none]` |
| `query_expansion.models` | LLM pool for the expansion call | `[]` |

`hyde` writes a hypothetical answer and retrieves with both the question and that passage. `multi_query` writes three rephrasings and retrieves with all of them. `query_decompose` splits a multi-hop question into sub-questions that replace the original. Results from every variant are merged and deduplicated before reranking. Any strategy other than `none` needs a non-empty `models` pool, and each adds one LLM call per query to the cost of the pipeline. When the pool has one model it is picked automatically. With more, `expander_llm` becomes a dimension the agent tunes.

## Reranker

| Key | Values | Default |
| --- | --- | --- |
| `reranker.models` | `none` or cross-encoder model ids loadable by Sentence Transformers | `[none]` |
| `reranker.top_n` | chunks kept after reranking | `3` to `10` |

With a reranker the pipeline fetches three times `top_k` chunks, scores each against the question with the cross-encoder, and keeps `reranker_top_n`. `reranker_top_n` must not exceed `top_k`. Reranking runs locally and does not count toward the per-query cost.

## Passage compression

| Key | Values | Default |
| --- | --- | --- |
| `passage_compressor.strategies` | `none`, `tree_summarize`, `refine` | `[none]` |
| `passage_compressor.models` | LLM pool for compression | `[]` |

`tree_summarize` summarizes the retrieved passages in batches and merges the summaries. `refine` walks the passages one at a time, refining a running answer, and is the slower of the two. Both need a non-empty `models` pool and add LLM calls per query. `compressor_llm` follows the same rule as `expander_llm`.

## Generator

| Key | Values | Default |
| --- | --- | --- |
| `generator.models` | LLM model strings | required |
| `generator.reasoning` | `true` lets the agent toggle reasoning per trial | `true` |
| `generator.reasoning_effort` | `low`, `medium`, `high`, applied when reasoning is on | `medium` |

The generator writes the final answer from the context and is the model your users talk to. Its cost dominates the per-query cost. `reasoning` is a per-trial lever when `generator.reasoning` is true and the chosen model supports it. `reasoning_effort` is fixed for the project. Reasoning is never applied to expansion or compression calls, and is not available for `ollama/` models.

## Temperature

| Key | Values | Default |
| --- | --- | --- |
| `temperature` | `{min, max}` range | `1.0` to `1.0` |

One value per trial, applied to every LLM call in the pipeline. The default pins it to 1.0 because several current models reject any other value. Widen the range only if every model in your pools accepts it.

## Graph retrieval

| Key | Values | Default |
| --- | --- | --- |
| `graph_retrieval.graph_query_modes` | `local`, `global`, `hybrid` | `[local, global, hybrid]` |
| `graph_retrieval.graph_top_k` | graph nodes explored | `20` to `100` |

Valid only when `retrieval.index_types` includes a graph type. `local` follows facts about named entities, `global` follows relations between entities, `hybrid` does both.

## What is not searched

Prompts, the document parser, the graph build settings, and the vector store are fixed. Set them once per project.

## Sizing the space

Every dimension you add widens what the agent has to cover in `meta.max_trials` trials. The starter config is a good shape: three embedding models, `vector_only` and `hybrid_bm25_vector`, one reranker plus `none`, and three generators spanning a price range. Add query expansion or compression only when you are willing to pay an extra LLM call per query in production.
