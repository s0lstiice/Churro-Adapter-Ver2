# Evaluation notes

`base_epoch03_epoch19_comparison.json` is the aggregate comparison used in the
repository README. `base_summary.json` and `epoch19_summary.json` retain the
underlying evaluator summaries.

## Protocol

- 100 full-page images
- source-protected from training and validation manifests
- identical images, references, image resolution, generation limit, and retry
  policy for base CHURRO and the adapters
- `max_pixels=1605632`
- `max_new_tokens=1536`
- at most two incompleteness retries
- deterministic generation (`do_sample=false`)
- case-insensitive and punctuation-insensitive CER/WER

## Results

| Model | CER | WER | Character edits | Word edits | Output/target chars |
|---|---:|---:|---:|---:|---:|
| Base CHURRO | 0.404500 | 0.498961 | 32,974 | 7,442 | 0.8518 |
| Epoch 3 | 0.383780 | 0.475897 | 31,285 | 7,098 | 0.9148 |
| Epoch 19 | **0.331853** | **0.417298** | **27,052** | **6,224** | **0.9777** |

Epoch 19 is materially better than upstream CHURRO on this development set.
It is not accurate enough to be treated as unattended archival ground truth.

## Important limitations

1. This is a development comparison, not a final untouched benchmark. Pages
   have been inspected during model development.
2. Some reference page boundaries remain under visual review. LOC item-level or
   paragraph-oriented formatting can include text outside the photographed page.
3. A syntactically complete generation can still silently omit visible text.
   The current detector catches token-limit truncation, unclosed XML, and known
   omission phrases; it does not prove image coverage.
4. Output/target length is diagnostic only because reference boundaries may be
   imperfect.
5. CER/WER measure text recognition, not XML coordinate or polygon accuracy.

A publication-quality result requires a newly held-out, source-disjoint set with
visually verified page boundaries and a separate first/last-line coverage audit.

