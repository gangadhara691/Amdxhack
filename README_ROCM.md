# Running lerobot on AMD ROCm (gfx1150) – working recipe

This is the exact sequence that brought a clean ROCm setup to a working state on an AMD Ryzen AI 9 HX 370 (gfx1150) laptop. All commands are intended to be run from the repository root (`~/Downloads/AmdxHack/lerobot`) inside a fresh virtual environment.

## 1) System prerequisites

```bash
sudo apt-get update
sudo apt-get install -y python3-venv cmake build-essential libegl1-mesa-dev libgl1-mesa-dev
```

## 2) Virtual environment and base tools

```bash
cd ~/Downloads/AmdxHack/lerobot
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip setuptools wheel typing-extensions filelock jinja2 networkx pillow
```

## 3) Install the ROCm 6.2 PyTorch stack

```bash
pip install --no-deps --force-reinstall \
  --index-url https://download.pytorch.org/whl/rocm6.2 \
  torch==2.5.1+rocm6.2 torchvision==0.20.1+rocm6.2 torchaudio==2.5.1+rocm6.2 pytorch-triton-rocm==3.1.0
```

## 4) Pin supporting Python deps

```bash
pip install sympy==1.13.1 "numpy<2.3" "fsspec<=2025.9.0" tokenizers==0.21.0
```

## 5) Install GPU-side extras expected by lerobot/libero

```bash
# Transformers fork required by lerobot
pip install --no-deps git+https://github.com/huggingface/transformers.git@fix/lerobot_openpi

# Peft and AWS pins to stop resolver backtracking
pip install --no-deps peft==0.17.1 boto3==1.34.162 botocore==1.34.162

# egl_probe (needs cmake)
pip install --no-build-isolation --no-deps git+https://github.com/huggingface/egl_probe.git@master

# Libero without pulling its own deps (avoids reintroducing egl-probe/boto)
pip install --no-deps git+https://github.com/huggingface/lerobot-libero.git@main
```

## 6) Install project dependencies (skip problematic lines)

Skip `flash-attn`, `egl-probe`, `libero`, any pinned `transformers==`, `peft==`, `boto3`, `botocore`, and `lerobot[pi]` lines when installing requirements. The following filters them out:

```bash
grep -Ev '^(flash-attn|egl-probe|libero[ @]|hf-libero|transformers==|peft==|boto3|botocore|lerobot|lerobot\[pi\])' requirements-ubuntu.txt > /tmp/req-final.txt
pip install --no-deps -r /tmp/req-final.txt

# Install lerobot itself (editable)
pip install --no-deps -e .

pip check
```

## 7) Verify ROCm is visible

```bash
python - <<'PY'
import torch
print("cuda available:", torch.cuda.is_available())
if torch.cuda.is_available():
    print("device:", torch.cuda.get_device_name(0))
PY
```

## 8) Run inference (AMD GPU)

If an old eval cache exists, move it aside:

```bash
mv ~/.cache/huggingface/lerobot/Ganga008/eval_act_so101_black_tape2 \
   ~/.cache/huggingface/lerobot/Ganga008/eval_act_so101_black_tape2.bak.$(date +%s)
```

Then launch inference. Keep `HSA_OVERRIDE_GFX_VERSION=11.0.0` for gfx1150; disable the viewer if it crashes by adding `export RERUN_DISABLE=1`.

```bash
source .venv/bin/activate
export HSA_OVERRIDE_GFX_VERSION=11.0.0

lerobot-record \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyACM1 \
  --robot.id=my_awesome_follower_arm \
  --robot.calibration_dir=/home/amddemo/.cache/huggingface/lerobot/calibration/robots/so101_follower \
  --robot.cameras='{ top: {type: opencv, index_or_path: /dev/video4, width: 640, height: 480, fps: 30}, gripper: {type: opencv, index_or_path: /dev/video6, width: 640, height: 480, fps: 30} }' \
  --display_data=true \
  --dataset.repo_id=Ganga008/eval_act_so101_black_tape2 \
  --dataset.single_task="Describe the eval task" \
  --policy.path=Ganga008/act_so101_black_tape2 \
  --policy.device=cuda \
  --policy.use_amp=false
```

If the viewer segfaults, set `export RERUN_DISABLE=1` and change `--display_data=false`.
