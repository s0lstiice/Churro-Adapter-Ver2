---
base_model: stanford-oval/churro-3B
library_name: peft
pipeline_tag: image-text-to-text
license: qwen-research
tags:
  - base_model:adapter:stanford-oval/churro-3B
  - qlora
  - handwriting-recognition
  - historical-documents
  - document-transcription
  - library-of-congress
  - churro
---

# LOC Mixed-Scale CHURRO LoRA — Epoch 19

> **Improved using Qwen.** This adapter is distributed for **noncommercial
> research and evaluation only** under the Qwen Research License. It is not
> cleared for production deployment. Government, nonprofit, or educational
> status alone does not expand the license beyond research and evaluation.

This repository contains a PEFT/LoRA adapter for
[`stanford-oval/churro-3B`](https://huggingface.co/stanford-oval/churro-3B),
specialized for full-page and single-line nineteenth-century English
handwriting from Library of Congress material.

It is suitable as a **research preview and human-assisted transcription tool**.
It is not an official Library of Congress transcription guideline, and its
output should not be published as archival ground truth without review.

This is an independent project. It is not an official release from Stanford
OVAL, Qwen/Alibaba Cloud, or the Library of Congress.

Anyone evaluating operational use should review [`LICENSE`](LICENSE),
[`NOTICE`](NOTICE), and [`MODIFICATIONS.md`](MODIFICATIONS.md) first. The Qwen
license includes indemnification, governing-law, and jurisdiction provisions
that may require separate institutional or federal-agency legal review. A
different license from Alibaba Cloud is required for use outside the granted
noncommercial research/evaluation scope.

## Why use Epoch 19?

Epoch 19 is the strongest checkpoint from this training run on the protected
100-page development comparison. Its main demonstrated benefit is not merely
producing longer text: it makes substantially fewer character and word edits
while bringing generated output much closer to the amount of text in the page
references.

- Against upstream CHURRO, it removes **5,922 character edits** and **1,218 word
  edits** across the 100 pages.
- Against the earlier Epoch 3 adapter, it removes another **4,233 character
  edits** and **874 word edits**.
- Its output contains 97.8% as many characters as the references, compared with
  85.2% for upstream CHURRO and 91.5% for Epoch 3. This is evidence of reduced
  aggregate under-generation, although it does not prove that every visible
  line was captured.
- It retains the upstream tokenizer and base model while adding only a 14.8 MB
  LoRA adapter, so deployment still uses the normal CHURRO generation path.

Epoch 19 is therefore the recommended checkpoint in this repository for
human-assisted transcription of similar LOC handwriting. It is still an OCR
model—not an archival authority—and individual pages or phrases can be worse
than earlier checkpoints. The unscored notebook example below deliberately
shows one such regression alongside the broader improvements.

## Measured comparison with upstream CHURRO

The same deterministic full-page evaluator and completeness-retry policy were
used for both models on the current 100-page, source-protected development set.
Scoring is case-insensitive and punctuation-insensitive.

| Model | CER ↓ | WER ↓ | Output/target character ratio |
|---|---:|---:|---:|
| Upstream CHURRO 3B | 40.45% | 49.90% | 0.852 |
| Epoch 3 adapter | 38.38% | 47.59% | 0.915 |
| **Epoch 19 adapter** | **33.19%** | **41.73%** | **0.978** |

Relative to upstream CHURRO, Epoch 19 reduced:

- CER by **17.96%** (7.26 percentage points);
- WER by **16.37%** (8.17 percentage points);
- character edits from 32,974 to 27,052; and
- word edits from 7,442 to 6,224.

Relative to Epoch 3, Epoch 19 reduced CER by **13.53%** and WER by
**12.31%** (5.19 and 5.86 percentage points, respectively). In absolute terms,
the reference set contains 81,518 characters; upstream CHURRO generated 12,078
fewer characters than that total, Epoch 3 generated 6,948 fewer, and Epoch 19
generated only 1,820 fewer. Because some reference page boundaries remain under
audit, these length differences are diagnostics rather than standalone accuracy
measurements.

The final selected Epoch 19 outputs contained zero truncations, zero detected
explicit omission placeholders, and zero detected incomplete XML documents.
Five pages triggered completeness retries and all five produced a syntactically
complete retry.

These numbers do **not** prove that every visible line was transcribed. A model
can silently close valid XML early. Some reference boundaries in this
development set also remain under visual audit, so this benchmark is appropriate
for model development—not a final generalization claim. See
[`evaluation/README.md`](evaluation/README.md).

## Qualitative page comparison without an official transcript

The following comparison was produced from a 1935 notebook scan supplied after
Epoch 19 was trained. It was **not** part of the scored 100-page comparison and
has **no verified official page transcript available here**. The scan's source
record and Rights and Access URL were not supplied, so the image itself is
intentionally **not redistributed in this repository**. Consequently, this
text-only diagnostic has no CER, WER, win/loss label, or claimed ground truth.

All three runs used the identical full-page image, deterministic decoding,
`max_pixels=1605632`, `max_new_tokens=1536`, and the same completeness-retry
policy. Each completed on its first attempt with closed XML and no explicit
omission marker.

The author's visual note for the page heading exposes an important difference,
but it is not an official transcript and cannot be independently verified from
this repository because the source image is withheld:

| Source | Heading transcription |
|---|---|
| Human visual reading (diagnostic, not an official transcript) | `1935` / `Jan 1. Tuesday. New Year Day.` |
| Upstream CHURRO | `1935` / `1` / `June 1 Tuesday. New York.` |
| Epoch 3 | `1935` / `1` / `Jan 1 Tuesday New Year Day.` |
| Epoch 19 | `1935` / `Jan 1 Tuesday New York` |

Epoch 3 reads this particular heading best. Epoch 19 correctly improves
`June 1` to `Jan 1`, but incorrectly changes `New Year Day` to `New York`.
Inspection of the training manifest found many `New York` examples and no
`New Year` examples, making this a useful illustration of language-prior bias:
the later checkpoint is better in aggregate but not monotonically better on
every phrase.

The opening body lines also differ materially:

| Model | Beginning of generated body text |
|---|---|
| Upstream CHURRO | `Yesterday I sent another letter to the editor of the St. Louis Star & Sunday News...` |
| Epoch 3 | `Yesterday I sent another letter to the friends of Kate Hennel...` |
| Epoch 19 | `Yesterday I sent another letter to the friends of State Street...` |

Without a verified transcript, none of these body variants should be declared
correct from model agreement alone. The complete outputs are preserved in
[`evaluation/examples/1935-notebook-page/README.md`](evaluation/examples/1935-notebook-page/README.md)
as a transparent behavior record. This text-only example is not visual evidence
and does not replace the protected 100-page measurement. The image may be added
later only with a verified source record, Rights and Access link, and credit
line.

## What is included

```text
adapter/                 Epoch 19 LoRA weights and processor/tokenizer files
scripts/transcribe.py    Standalone page/line inference with completeness retries
scripts/finetune.py      Mixed-scale QLoRA trainer
scripts/progress_client.py
evaluation/              Aggregate base/Epoch 3/Epoch 19 measurements
  examples/              Unscored qualitative page comparisons
training/                Dataset audit, loss histories, and run configuration
ARCHITECTURE_ROADMAP.md  Autoregression rationale and successor-model plan
CITATIONS.bib            CHURRO and Qwen2.5-VL citations
LICENSE                  Upstream Qwen Research License terms
NOTICE                   Attribution and data notice
MODIFICATIONS.md         Changes relative to upstream CHURRO/Qwen
requirements.txt
SHA256SUMS
```

The approximately 7.5 GB CHURRO base checkpoint and all training scans and
transcripts are intentionally excluded. The adapter weight file is about 14.8
MB and contains 3,686,400 trainable parameters (rank 8).

## Quick start

The tested runtime uses Python 3.12, CUDA, and 4-bit `bitsandbytes` loading.

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
```

Transcribe one page:

```bash
python scripts/transcribe.py path/to/page.jpg --output outputs/page
```

Transcribe a directory:

```bash
python scripts/transcribe.py path/to/pages --recursive --output outputs/pages
```

Transcribe existing line crops:

```bash
python scripts/transcribe.py path/to/lines --recursive --task line --output outputs/lines
```

The script writes `predictions.jsonl`, one extracted `.txt` file per image, and
one raw `.xml` response per image. It downloads `stanford-oval/churro-3B` on the
first run. Set `HF_TOKEN` if Hugging Face authentication or higher download
limits are needed.

The default `--max-pixels 1605632 --max-new-tokens 1536 --retries 2` reproduces
the measured inference policy. On a 6 GB GPU, 4-bit loading may work only with a
smaller `--max-pixels` value; reducing it can lower recognition accuracy.

## Training data

Only official/human transcript text was used as the optimization target. OCR or
CHURRO predictions could assist alignment and filtering but were never copied
into target labels.

| Split | Lines | Unique full pages | Page samples per epoch | Total examples |
|---|---:|---:|---:|---:|
| Train | 2,400 | 171 | 600 | 3,000 |
| Validation | 264 | 17 | 17 | 281 |

Page samples were restricted to direct per-page LOC text. Aggregate item-level
descriptions and heuristic whole-file transcript assignments were excluded.
Capitalization and punctuation were preserved. Training and validation were
source-disjoint, and protected evaluation sources had zero overlap with either.

The Library of Congress identifies the digitized Abraham Lincoln Papers scans
used here as public domain. Its released Abraham Lincoln transcription dataset
is also marked Public Domain/CC0. See the official
[collection Rights and Access statement](https://www.loc.gov/collections/abraham-lincoln-papers/about-this-collection/rights-and-access/)
and [transcription dataset record](https://www.loc.gov/item/2025475020/).
Credit line: Library of Congress, Manuscript Division, Abraham Lincoln Papers.

## Training method

The adapter began as a clean LoRA on the upstream CHURRO base and was optimized
with mixed page/line examples. The Qwen tokenizer and upstream base weights were
not replaced.

| Setting | Value |
|---|---:|
| Base | `stanford-oval/churro-3B` |
| LoRA targets | `q_proj`, `k_proj`, `v_proj`, `o_proj` |
| Rank / alpha / dropout | 8 / 16 / 0.05 |
| Trainable / total parameters | 3,686,400 / 3,758,309,376 |
| Gradient accumulation | 8 |
| Maximum sequence length | 2,048 |
| Page / line maximum pixels | 602,112 / 401,408 |
| Seed | 3407 |

Validation loss declined from 0.78377 at Epoch 1 to 0.61539 at Epoch 19. At
Epoch 19, line validation loss was 0.91962 and page validation loss was 0.31116.
The final learning rate was `5.95e-8`; the late epochs were refinement, not 19
full epochs at a large learning rate.

Machine-readable histories and the continuation configuration are under
[`training/`](training/).

## Fine-tuning

The trainer expects UTF-8 JSONL rows with `id`, `image`, `text`, and
`task_type` (`page` or `line`). `text` must be a verified human/official target.

```json
{"id":"page-001","image":"/data/page-001.jpg","text":"Verified page text...","task_type":"page"}
{"id":"line-001","image":"/data/line-001.png","text":"Exact line text.","task_type":"line"}
```

For a new adapter, start from upstream CHURRO and use early stopping. Do not
assume that a larger learning rate can safely replace all 19 epochs; test the
schedule on a locked, source-disjoint validation set.

```bash
python scripts/finetune.py \
  --train-manifest data/train.jsonl \
  --validation-manifest data/validation.jsonl \
  --output runs/new_adapter \
  --epochs 8 \
  --learning-rate 1e-5 \
  --early-stopping-patience 2 \
  --gradient-accumulation 8 \
  --line-max-pixels 401408 \
  --page-max-pixels 602112 \
  --max-sequence-length 2048 \
  --lora-rank 8 \
  --lora-alpha 16
```

A safer fast experiment is one epoch at `1e-5`, followed by `5e-6` with early
stopping, while selecting checkpoints using generated CER/WER and visual
coverage—not validation loss alone.

## Why CHURRO is autoregressive

CHURRO inherits Qwen2.5-VL's causal language-model decoder. It generates each
text token conditioned on the image and preceding tokens. This supports
variable-length transcription, spelling and punctuation priors, flexible XML
structure, and prompt-selected output schemas. It also permits premature valid
XML closure and silent omission.

Autoregression cannot be removed with a generation flag. A non-autoregressive
successor would need a new decoder and new training objective; this LoRA would
not be directly compatible.

## Intended use and limitations

- Use for research and assisted transcription of similar English historical
  handwriting.
- Always retain the scan and review the output before archival use.
- The model may omit, duplicate, hallucinate, or misorder text.
- Well-formed XML is not proof of visual coverage.
- XML coordinates are not a reliable trained line or polygon segmenter.
- Accuracy is not established for other writers, periods, languages, printed
  documents, or degraded scans.
- CER/WER ignore case and punctuation in the reported comparison; the generated
  text itself retains both.

## License and citations

The upstream model card identifies the Qwen Research License. Review
[`LICENSE`](LICENSE), [`NOTICE`](NOTICE), and the
[upstream CHURRO model card](https://huggingface.co/stanford-oval/churro-3B)
before use or redistribution.

Please cite the [CHURRO paper](https://almond-static.stanford.edu/papers/emnlp2025_historical_ocr.pdf)
and the [Qwen2.5-VL technical report](https://arxiv.org/abs/2502.13923).
BibTeX entries are provided in [`CITATIONS.bib`](CITATIONS.bib).
