---
base_model: stanford-oval/churro-3B
library_name: peft
pipeline_tag: image-text-to-text
license: other
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

This repository contains a PEFT/LoRA adapter for
[`stanford-oval/churro-3B`](https://huggingface.co/stanford-oval/churro-3B),
specialized for full-page and single-line nineteenth-century English
handwriting from Library of Congress material.

It is suitable as a **research preview and human-assisted transcription tool**.
It is not an official Library of Congress transcription guideline, and its
output should not be published as archival ground truth without review.

This is an independent project. It is not an official release from Stanford
OVAL, Qwen/Alibaba Cloud, or the Library of Congress.

## Result compared with upstream CHURRO

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

The final selected Epoch 19 outputs contained zero truncations, zero detected
explicit omission placeholders, and zero detected incomplete XML documents.
Five pages triggered completeness retries and all five produced a syntactically
complete retry.

These numbers do **not** prove that every visible line was transcribed. A model
can silently close valid XML early. Some reference boundaries in this
development set also remain under visual audit, so this benchmark is appropriate
for model development—not a final generalization claim. See
[`evaluation/README.md`](evaluation/README.md).

## What is included

```text
adapter/                 Epoch 19 LoRA weights and processor/tokenizer files
scripts/transcribe.py    Standalone page/line inference with completeness retries
scripts/finetune.py      Mixed-scale QLoRA trainer
scripts/progress_client.py
evaluation/              Aggregate base/Epoch 3/Epoch 19 measurements
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
not be directly compatible. The recommended next experiment is a dedicated
line/polygon proposer plus a CTC-style line recognizer, optionally using Epoch
19 only for alignment or confidence. See [`ARCHITECTURE_ROADMAP.md`](ARCHITECTURE_ROADMAP.md).

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

