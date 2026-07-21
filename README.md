# Franka Fruit-Pick Demo

A compact, end-to-end **reference pipeline** for scripted-to-learned robotic manipulation,
built on the [Genesis](https://github.com/Genesis-Embodied-AI/Genesis) physics engine. A
Franka Panda picks fruit (banana / lemon / plum) off a table and places it into a bowl. The
demo walks the full path from a hand-built scene to a trained, closed-loop visuomotor policy.

## Pipeline (M1 -> M5)


| Stage                                    | What it does                                                                       | Key files                                                                 |
| ---------------------------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| **M1 — Scene**                           | Build the Franka + table + YCB-object scene; verify/populate assets.               | `scene_config.py`, `setup_assets.py`, `build_scene.py`                    |
| **M2 — Scripted policy + randomization** | Per-episode reset randomization and a scripted, IK-based pick-and-place.           | `randomize.py`, `grasp_demo.py`                                           |
| **M3 — Data**                            | Record scripted episodes into a LeRobot dataset; merge per-object sets.            | `record_dataset.py`, `aggregate_datasets.py`                              |
| **M4 — Domain randomization**            | Layer-A (build-time appearance/intrinsics) + Layer-B (runtime physics/extrinsics). | inside `build_scene.py` / `randomize.py` (preview: `tools/dr_preview.py`) |
| **M5 — Train & eval**                    | Train a LeRobot policy, then run closed-loop evaluation and checkpoint sweeps.     | `train_policy.py`, `eval_policy.py`, `eval_sweep.py`                      |


## Repository layout

```
franka_fruit_pick_demo/
  franka_fruit_pick/          # all pipeline code (import + run in place)
    paths.py                  # single source of truth for on-disk layout
    scene_config.py  setup_assets.py  build_scene.py       # M1
    randomize.py     grasp_demo.py                          # M2
    record_dataset.py  aggregate_datasets.py                # M3 / M4 data
    train_policy.py  eval_policy.py  eval_sweep.py          # M5
    tools/                    # helper / debug / one-off utilities
      motion_probe.py         # scripted-motion quality diagnostics
      dr_preview.py           # domain-randomization visual preview
      recheck_dataset.py      # re-validate a dataset vs the current success criterion
      scale_ycb.py            # bake scaled copies of YCB meshes (dev-only)
      test_kitchen_table_scene.py  # standalone scene visual smoke test
  assets/                     # bundled meshes / robot model (see below)
  datasets/                   # recorded / aggregated LeRobot datasets
  outputs/                    # generated eval results, videos, frames (gitignored)
```

All scripts use flat sibling imports and add the package directory to `sys.path`
themselves, so you can run any of them directly from the repo root — no `pip install`
step is required to run the demo.

## Quickstart

```bash
# 1. Install dependencies (physics engine + demo deps)
#    Requires Python 3.12 (matches the ROCm torch wheels below).
uv sync

# 2. Install a policy backend for M5
#    This project runs on an AMD Radeon (ROCm) GPU — use the prebuilt ROCm 7.2.1 wheels (Python 3.12):
wget https://repo.radeon.com/rocm/manylinux/rocm-rel-7.2.1/torch-2.9.1%2Brocm7.2.1.lw.gitff65f5bc-cp312-cp312-linux_x86_64.whl
wget https://repo.radeon.com/rocm/manylinux/rocm-rel-7.2.1/torchvision-0.24.0%2Brocm7.2.1.gitb919bd0c-cp312-cp312-linux_x86_64.whl
wget https://repo.radeon.com/rocm/manylinux/rocm-rel-7.2.1/triton-3.5.1%2Brocm7.2.1.gita272dfa8-cp312-cp312-linux_x86_64.whl
wget https://repo.radeon.com/rocm/manylinux/rocm-rel-7.2.1/torchaudio-2.9.0%2Brocm7.2.1.gite3c6ee2b-cp312-cp312-linux_x86_64.whl

uv pip uninstall torch torchvision triton torchaudio
uv pip install \
    torch-2.9.1+rocm7.2.1.lw.gitff65f5bc-cp312-cp312-linux_x86_64.whl \
    torchvision-0.24.0+rocm7.2.1.gitb919bd0c-cp312-cp312-linux_x86_64.whl \
    torchaudio-2.9.0+rocm7.2.1.gite3c6ee2b-cp312-cp312-linux_x86_64.whl \
    triton-3.5.1+rocm7.2.1.gita272dfa8-cp312-cp312-linux_x86_64.whl

uv pip install "lerobot[smolvla]==0.4.4"

# 3. Verify assets are in place (M1)
uv run python franka_fruit_pick/setup_assets.py

# 4. Smoke-test the scene (renders one frame per camera)
uv run python franka_fruit_pick/build_scene.py --steps 50 --save-frames
```

### Run the pipeline

```bash
# M3 — record scripted episodes into a LeRobot dataset
uv run python franka_fruit_pick/record_dataset.py --episodes 50 --pick 011_banana --repo-id demo/banana_pick

# (optional) merge per-object datasets into one training set
uv run python franka_fruit_pick/aggregate_datasets.py \
    --dataset-root datasets/banana_pick --dataset-root datasets/lemon_pick \
    --dataset-root datasets/plum_pick   --out datasets/fruit_pick

# M5 — train, then evaluate closed-loop / sweep checkpoints
uv run python franka_fruit_pick/train_policy.py smolvla --repo-id demo/fruit_pick --dataset-root datasets/fruit_pick
uv run python franka_fruit_pick/eval_policy.py --run-dir <train-out> --repo-id demo/fruit_pick --episodes 50 --save-video
uv run python franka_fruit_pick/eval_sweep.py  --run-dir <train-out> --repo-id demo/fruit_pick
```

## Assets & datasets

- `**assets/**` is bundled so the scene runs out of the box. `setup_assets.py` is idempotent:
if the required meshes / robot model are already present it does nothing; otherwise it tries
to populate them from local source datasets (development only).
- `**datasets/**` holds recorded and aggregated LeRobot datasets. It ships empty; recording
writes new datasets here (`datasets/<repo-name>/`).
- Both directories hold **large binaries**. If you push them to GitHub, use
[git-lfs](https://git-lfs.com/) or host them separately and document the download step.

## Notes

- **Rendering**: the default per-camera rasterizer works on any Genesis GPU backend
(NVIDIA/CUDA, AMD/ROCm). `build_scene.py --batch-render` enables the Madrona batch
renderer for all-envs-in-one-pass observation rendering, which is **NVIDIA-CUDA only**.
- `**outputs/`** (eval results, videos, scripted-demo frames) is generated and gitignored.

