# Bring your own exam

By default Agentic AutoRAG generates a synthetic exam from your corpus and optimizes against it. If you already have questions with known answers (a hand-written set, an exported evaluation set, or a labelled QA dataset) you can optimize against those instead. Point the examiner at a JSON file and generation is skipped.

```yaml
examiner:
  custom_exam_path: path/to/exam.json
```

When `custom_exam_path` is set, `optimize` loads your file, skips corpus composition and probe selection, and runs the whole reasoning loop against your questions. Nothing is dropped. Every question you provide is evaluated.

## Exam format

The file is a JSON list of question records. Each record has at minimum a unique `id`, the `question`, and its `canonical_answer`. How much evidence you attach to a question defines its grounding tier, and the tier decides how much diagnostic detail the optimizer can give you.

| Tier | You provide | The optimizer can score |
| ---- | ----------- | ----------------------- |
| **A** | question + answer | answer accuracy. A judge attributes each wrong answer to retrieval or generation. |
| **B** | + `supporting_doc_ids` | the above plus document-level retrieval (did the right documents come back?) |
| **C** | + verbatim evidence `source_spans` | the above plus span-level retrieval (did the exact evidence come back?) |

You can mix tiers in one file. Fields:

- `id` (str, required). Unique per question. Duplicate ids are rejected.
- `question` (str, required).
- `canonical_answer` (str, required). The primary correct answer.
- `answer_variants` (list of str, optional). Other acceptable surface forms.
- `supporting_doc_ids` (list of str). Ids of the corpus documents that support the answer (tier B). An id is a corpus file name without its extension. Allowed on any tier.
- `source_doc_ids` and `source_spans` (parallel lists of str). Verbatim evidence (tier C). Giving `source_doc_ids` without `source_spans` is a validation error. Use `supporting_doc_ids` for document ids alone.
- `reasoning_type` (str, optional). One of `extraction`, `definitional`, `inference`, `bridge`, `comparison`, `numeric_single`, `numeric`. Used only to pick the answer-format hint and to group failures in the diagnosis. If omitted, the model is asked for the most concise answer that still fully answers the question, usually a name, value, or short phrase.

The file must be a non-empty JSON list of objects. Anything else fails at load time.

### Tier A: question + answer

```json
[
  { "id": "q1", "question": "In what year did Apollo 11 land humans on the Moon?", "canonical_answer": "1969" }
]
```

Tier A questions cost one extra judge call per wrong answer, which is how the failure is attributed to retrieval or generation.

### Tier B: add the supporting documents

```json
[
  {
    "id": "q2",
    "question": "What is the capital of the country that hosted the 2016 Summer Olympics?",
    "canonical_answer": "Brasília",
    "answer_variants": ["Brasilia"],
    "supporting_doc_ids": ["summer_olympics_2016", "brazil"]
  }
]
```

### Tier C: add verbatim evidence spans

Tier C adds two aligned lists with one entry per piece of evidence: `source_doc_ids` (which document each span comes from) and `source_spans` (a verbatim substring copied from that document).

```json
[
  {
    "id": "q2",
    "question": "What is the capital of the country that hosted the 2016 Summer Olympics?",
    "canonical_answer": "Brasília",
    "answer_variants": ["Brasilia"],
    "source_doc_ids": ["summer_olympics_2016", "brazil"],
    "source_spans": [
      "The 2016 Summer Olympics were held in Rio de Janeiro, Brazil.",
      "The capital of Brazil is Brasília, inaugurated in 1960."
    ]
  }
]
```

You rarely need to hand-write tier C. See below.

## Unanswerable (abstention) questions

Some questions have no answer in your corpus, and the correct behavior is to abstain rather than guess. Mark a question unanswerable by writing its `canonical_answer` as a clear statement of insufficiency, for example exactly `Insufficient information.`, and giving it no `supporting_doc_ids`, `source_spans`, or `reasoning_type`. There is no separate flag. The gold answer speaks for itself.

```json
[
  { "id": "q3", "question": "How many employees did the firm have in 1850?", "canonical_answer": "Insufficient information." }
]
```

The answerer is allowed to reply that the context is insufficient. The judge reads the gold. A system answer that likewise abstains is graded correct, and a confident factual claim is graded wrong (an over-answering failure). This lets an exam measure calibrated abstention alongside ordinary accuracy.

## Upgrade a tier-B exam to tier C automatically

If your questions already name their supporting documents (tier B), let an LLM extract the evidence spans for you and verify that each one is present in its document:

```bash
uv run agentic-autorag ground-exam \
  --config configs/my_project.yaml \
  --exam my_exam_tierB.json \
  --output my_exam_tierC.json
```

The command reads the documents from your config's `meta.corpus_path` and uses its examiner model (override with `--extractor-model`). It reads the files directly, so it supports only corpora of `.md` and `.txt` files. Every question whose spans all verify is upgraded to tier C. Any question that cannot be verified is kept unchanged as tier B. Nothing is dropped. A provenance file next to the output records the counts. Then point `custom_exam_path` at the output.
