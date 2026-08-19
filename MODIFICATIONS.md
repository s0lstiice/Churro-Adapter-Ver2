# Modification notice

This repository is not an unmodified copy of CHURRO or Qwen2.5-VL.

- The weights in `adapter/adapter_model.safetensors` are newly trained LoRA
  parameters for `stanford-oval/churro-3B`.
- The adapter was trained on mixed full-page and aligned-line handwriting data
  from the Library of Congress Abraham Lincoln Papers.
- The adapter configuration targets the attention projection modules listed in
  `adapter/adapter_config.json`.
- `scripts/finetune.py` and `scripts/transcribe.py` are independent training and
  inference utilities prepared for this adapter package.
- The upstream CHURRO/Qwen base weights are unchanged and are not redistributed
  in this repository.

See `README.md`, `training/`, and `evaluation/` for the exact configuration,
data-audit summary, losses, and preliminary evaluation limitations.
