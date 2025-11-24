# Prostate MRI Self-Supervised Pretraining

This project inherits MM-DINOv2 and provides a focused workflow for prostate MRI SSL.

## Layout

- `configs/` – experiment recipes merged on top of `dinov2/configs/ssl_default_config.yaml`.
- `data/` – dataset helpers.
- `engine/` – lightweight training loop mirroring `dinov2/train/train.py`.
- `scripts/` – entry points (pretraining, split generation, plotting).
- `splits/` – patient ID lists for train/val/test.

## Quick Start (10k patients)

1. **Prepare data**  
   Each patient under `X:/AIS_processed_20251015update/<patient_id>/` with files:
   ```
   ax_t2wi.nii
   ax_adc.nii
   ax_dwi_1500.nii
   roi_Prostate.nii   # mask required for tumor-centered cropping
   ```
   `.nii.gz` or same-named subfolders are also accepted.

2. **Generate splits (deterministic, easy to redo after data updates)**  
   ```
   uv run python Prostate_MRI_SSL_pretrain/scripts/create_splits.py \
       --dataset-root X:/AIS_processed_20251015update \
       --train-ratio 0.9 --val-ratio 0.05 --test-ratio 0.05 \
       --seed 42 --overwrite
   ```
   Re-run after新增数据即可快速重划分；输出到 `Prostate_MRI_SSL_pretrain/splits/train.txt|val.txt|test.txt`。

3. **Check/adjust config**  
   - `configs/prostate_ssl.yaml` points to the dataset root above and uses sequences
     `ax_t2wi, ax_adc, ax_dwi_1500` with `segmentation_key=roi_Prostate`.
   - `auto_epoch_length=true` will infer steps/epoch from the train split size; defaults are tuned for单卡24GB。需要时可用 CLI 覆盖：
     ```
     uv run python Prostate_MRI_SSL_pretrain/scripts/run_pretraining.py \
         --config Prostate_MRI_SSL_pretrain/configs/prostate_ssl.yaml \
         train.batch_size_per_gpu=4 train.num_workers=8
     ```

4. **Launch pretraining**
   ```
   uv run python Prostate_MRI_SSL_pretrain/scripts/run_pretraining.py \
       --config Prostate_MRI_SSL_pretrain/configs/prostate_ssl.yaml
   ```
   Outputs go to `out/pre_ssl/prostate_ssl_pretrain/` (resolved config, metrics JSONL, checkpoints).

5. **Monitor losses**
   ```
   uv run python Prostate_MRI_SSL_pretrain/scripts/plot_metrics.py \
       --metrics out/pre_ssl/prostate_ssl_pretrain/training_metrics.jsonl \
       --rolling-window 20 --include-lr \
       --output out/pre_ssl/loss_curve.png
   ```

## Notes

- `segmentation_key` is required for tumor-centered cropping; ensure mask文件存在且命名匹配。
- Checkpoint frequency and logging cadence are configurable via YAML/CLI.
- Splits are simple text files; rerun the split script whenever数据更新，使用同一 seed 可保持可重复性。
