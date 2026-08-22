# Exam generation

The optimizer scores every trial against an exam: open-ended questions with ground-truth answers, written from your corpus by the examiner model. This page explains how the exam is built, what the settings under `examiner:` do, and how to read the neighborhood weighting, which is the setting people most often get wrong.

If you already have questions with known answers, point `examiner.custom_exam_path` at them and generation is skipped. See [custom_exam.md](custom_exam.md).

## The pipeline

**Parse.** Every file under `meta.corpus_path` is converted to text by Docling (PDF, DOCX, XLSX, PPTX, HTML, CSV, Markdown, plain text, AsciiDoc, and images through OCR). The parsed corpus is cached under `.cache/` and re-parsed only when a file changes. Large corpora are trimmed to `meta.corpus_word_budget` words by a seeded sample of files.

**Deduplicate.** Near-duplicate documents (for example a PDF and its OCR'd page images) are collapsed to one representative for question writing. The trial index still contains every document.

**Chunk.** The exam chunker splits each document into chunks of at most `max_chunk_words` words, drops documents shorter than `min_doc_words`, and drops chunks whose section is in `excluded_section_types` (references, acknowledgments, and author blocks by default). This chunking is separate from the retrieval chunking the optimizer tunes per trial.

**Sample anchors.** The examiner picks `exam_size × initial_question_multiplier` anchor chunks (120 at the defaults), weighted by chunk length and without replacement, with a seed derived from `meta.project_name`.

**Build a neighborhood per anchor.** Each anchor is expanded into a neighborhood, the set of chunks the composer sees in one call. The next section explains how.

**Compose.** One call to `agent.examiner_model` per neighborhood. The composer writes every strong question the chunks support, cites the chunks and the verbatim spans each question depends on, and picks the question type. A neighborhood with nothing worth asking returns no questions.

**Gate.** Each candidate must pass: self-contained phrasing (no "the document", "this study"), every cited span located in its document (exact, whitespace-tolerant, or fuzzy at `source_fact_verify_fuzzy_threshold`), an answerability check where `agent.judge_model` answers from the cited spans alone and is graded against the canonical answer, a check that every cited span of a multi-hop question is needed, and a formula check for numeric types.

**Probe.** With `probe_selection: true` the survivors are answered by up to four probe pipelines built from the weakest to the strongest options in your search space. A question every probe answers cannot separate configurations and is dropped. Questions no probe answers are kept but capped at 12% of the exam. The rest are weighted toward the harder ones and sampled down to `exam_size`.

**Cache.** The exam is written to `exam.json` at the root of `meta.output_dir` and reused by every later run of the project. Fewer than half of `exam_size` surviving questions is an error.

## Question types

The composer chooses one type per question from what the chunks support. There are no per-type quotas.

| Type | What it asks |
| --- | --- |
| `extraction` | A fact stated verbatim in one span: a name, value, date, or short phrase. |
| `definitional` | A definition or short description as the text states it. |
| `numeric_single` | A value computed from two or more numbers in one chunk. Comes with a formula. |
| `inference` | A fact the spans make true but never state. The reader must combine two or more spans. |
| `bridge` | An entity is described indirectly in one span and identified in another. Asks for an attribute of that entity. |
| `comparison` | A value read from each of two or more spans and compared. |
| `numeric` | Arithmetic across spans: a difference, sum, ratio, or duration. Comes with a formula. |

`bridge`, `comparison`, and `numeric` need two or more cited spans. The spans may come from different chunks of the same document, from different documents, or from two places in one chunk.

## Neighborhood weighting

A neighborhood is the anchor chunk plus a fixed number of extra chunks. Its size is the smaller of two floors: `neighborhood_min_chunks` chunks, or as many chunks as it takes to reach `neighborhood_min_words` words. With long chunks the word floor wins, so a neighborhood on a corpus of PDFs is about six chunks. With short chunks the chunk floor wins and the neighborhood has twelve.

`neighborhood_same_doc_weight` and `neighborhood_cross_doc_weight` split the extra slots between chunks from the anchor's own document and chunks from other documents. Only the ratio matters: 0.8/0.2 and 4/1 are the same setting. At the defaults with twelve chunks, nine slots go to the anchor's document and two to other documents.

The same-document slots are filled from the start of the anchor's document in reading order, so the composer sees the document head (title, abstract, introduction) next to the anchor. The cross-document slots are filled with the chunks from other documents that share the most words and phrases with the anchor and its same-document chunks. This similarity is lexical (word n-grams) on purpose. Dense embeddings would pick the same chunks a dense retriever finds, and the exam would then favour dense retrieval over every other configuration.

There is no similarity floor on the cross-document side. The slots are filled whenever other documents exist, even when the best match shares only a topic word. The composer is instructed to refuse questions that link two chunks by topic coincidence, and on such a neighborhood it writes within-document questions instead. The weights therefore control only the material the composer sees. They do not control how many cross-document questions survive.

This is why the default favours the same document. Most real corpora are reports, manuals, papers, or contracts, and their documents rarely reference each other in a way that supports a genuine two-document question. On such a corpus nearly every surviving question stays within one document, spread across different chunks of it, even at 0.8/0.2. Raising the cross-document weight there does not produce cross-document questions. It replaces useful same-document context with unrelated chunks and lowers the number of questions the composer can write. Lower `neighborhood_same_doc_weight` (down to 0.0, together with a larger `neighborhood_min_chunks`) only when your documents reference each other by design, for example a corpus of short encyclopedia paragraphs where one paragraph names an entity that another paragraph describes.

When the anchor's document has fewer chunks than its share, the unused slots go to other documents. When there are too few other documents, the neighborhood is smaller.

To see what happened on your corpus: `details/debug/composition_log.json` records every neighborhood with a `position_kind` of `anchor`, `same_doc`, or `cross_doc` per chunk, `run.log` records the configured `same_doc_ratio` after the neighborhoods are built, and in `exam.json` a question is cross-document exactly when `source_doc_ids` names more than one document. `agentic-autorag generate-exam` prints that count as `multi-doc: N/M`.

## Settings

| Field | Default | Meaning |
| --- | --- | --- |
| `exam_size` | `80` | Questions in the final exam. Generation cost scales with this, not with corpus size. |
| `custom_exam_path` | `null` | Path to your own exam JSON. When set, nothing on this page runs. |
| `initial_question_multiplier` | `1.5` | Anchors to sample, as a multiple of `exam_size`. Each anchor can yield several questions or none. Raise it when the exam under-fills. |
| `probe_selection` | `true` | Run the probe ladder and keep discriminating questions. Turn off only for debugging. |
| `save_debug_artifacts` | `true` | Write the composition log, span verification, and rejection audits to `details/debug/`. |
| `composition_temperature` | `1.0` | Temperature of the composer call. Some models accept only 1.0. |
| `neighborhood_min_chunks` | `12` | Chunk floor for a neighborhood. |
| `neighborhood_min_words` | `5000` | Word floor for a neighborhood. The smaller floor wins. |
| `neighborhood_same_doc_weight` | `0.8` | Share of extra slots from the anchor's document. |
| `neighborhood_cross_doc_weight` | `0.2` | Share of extra slots from other documents. |
| `source_fact_verify_fuzzy_threshold` | `0.9` | Minimum overlap for a cited span to count as found when it is not verbatim. |
| `chunk_relevance_min_overlap_chars` | `50` | Character overlap for a retrieved chunk to count as containing a span. |
| `chunk_relevance_ngram_size` | `5` | N-gram size of the span-in-chunk matcher. |
| `chunk_relevance_overlap_threshold` | `0.5` | N-gram coverage that counts as a match. |
| `chunk_relevance_min_run` | `5` | Consecutive matching n-grams that also count as a match. |
| `max_chunk_words` | `1000` | Word budget per exam chunk. |
| `min_doc_words` | `200` | Documents shorter than this are skipped. |
| `excluded_section_types` | `[references, acknowledgments, author_info]` | Sections never used for questions. Valid labels: `body`, `abstract`, `methods`, `results`, `discussion`, `references`, `acknowledgments`, `author_info`. |

Two models take part. `agent.examiner_model` writes the questions. `agent.judge_model` runs the answerability check during generation and grades answers during trials, so pick one at least as strong as the generators in your search space. Both run with `agent.concurrency` parallel requests. Set `agent.examiner_reasoning_effort` to `null` for models without a reasoning mode.

## Regenerating the exam

`agentic-autorag generate-exam --config <yaml>` runs everything up to the exam and prints a summary: the probe ladder solve rates, how many probes solved each question, counts by type and hop count, and the multi-document count. Add `--regen` to delete the cached exam and candidates and write a new one. The corpus and embedding caches stay. The exam cache consists of `exam.json` and `details/exam_cost.json`. Delete both, or use `--regen`, when you want a fresh exam.
