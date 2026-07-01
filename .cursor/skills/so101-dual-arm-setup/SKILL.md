---
name: so101-dual-arm-setup
description: Hardware setup, calibration, teleoperation, and dataset recording for a specific dual-follower SO-101 rig (one leader + two follower arms, top/wrist/side cameras). Use when running lerobot-calibrate, lerobot-teleoperate, or lerobot-record on this SO-101 setup, or when the user mentions their follower/leader arms, the brush/solar-panel task, or arm IDs like dg_follower_arm / jg_follower_arm / dg_leader_arm.
---

# SO-101 Dual-Arm Setup

Concrete, copy-paste workflow for this physical SO-101 rig: **one leader arm** teleoperating **two interchangeable follower arms**, with top + wrist + side cameras. For the general SO-101 cheat-sheet (install, training, eval, data tips), see [`AGENT_GUIDE.md`](../../../AGENT_GUIDE.md) §4–§8; this skill captures the rig-specific values, the **mandatory pre-flight test**, and the **known-good recording command**.

> **Never store secrets here.** HF tokens, W&B keys, and GitHub PATs go through `hf auth login` / `wandb login` / the OS keychain — never in tracked files. Authenticate via the commands in "Accounts" below.

## Environment (READ FIRST — this machine is the exception)

`uv` / the repo `.venv` is **broken on this Intel Mac**: uv resolves `macosx_26_0_x86_64` and PyTorch only ships arm64 mac wheels, so `uv run …` fails and `.venv` is empty. **Use the working conda env `lerobot` instead.** Do NOT use `uv run` here.

```bash
# Option A: explicit binaries (robust in non-interactive shells / agents)
LRBIN=/Users/dhirajdgandhi/opt/anaconda3/envs/lerobot/bin
$LRBIN/python -c "import torch; print(torch.__version__, torch.backends.mps.is_available())"   # 2.10.0 True

# Option B: activate it (interactive terminals)
conda activate lerobot
```

All commands below assume `$LRBIN` is set (Option A). If you activated the env (Option B), drop the `$LRBIN/` prefix.

## Hardware map

USB ports are **machine-specific** — they can change when you replug or move to another machine. Always re-discover with `lerobot-find-port` (unplug the arm when prompted). The values below are the last-known ports for this rig:

| Role          | Arm ID (calibration key) | Last-known port (verify!)          |
| ------------- | ------------------------ | ---------------------------------- |
| Leader        | `dg_leader_arm`          | `/dev/tty.usbmodem5B140300941`     |
| Follower (DG) | `dg_follower_arm`        | `/dev/tty.usbmodem5B3D0444431`     |
| Follower (JG) | `jg_follower_arm`        | `/dev/tty.usbmodem5B610340841`     |

```bash
LEADER_PORT=/dev/tty.usbmodem5B140300941
DG_FOLLOWER_PORT=/dev/tty.usbmodem5B3D0444431
JG_FOLLOWER_PORT=/dev/tty.usbmodem5B610340841
```

Cameras (OpenCV `index_or_path`) — discover with `lerobot-find-cameras opencv`, then map:

| Index | View / device                | Native format                  |
| ----- | ---------------------------- | ------------------------------ |
| 0     | top (generic USB2.0_CAM1)    | 16:9 — does 640×360 / 1280×720 |
| 1     | wrist (Logitech C925e)       | 16:9 — does 640×480 / 1280×720 |
| 2     | MacBook FaceTime (built-in)  | —                              |
| 3     | side / full setup            | —                              |

> **Camera resolution gotcha (cost us many failed runs — do not repeat):** the two recording cameras are **different models with different native aspect ratios**. If you force a 4:3 size like **640×480**, the top cam intermittently emits **640×360**, tripping a strict dimension check that kills the read thread mid-episode (`frame ... do not match configured ...`). **Both cameras share exactly one stable native resolution: `1280×720`.** Always record both at **1280×720** on this rig.

## MANDATORY pre-flight test (be a pilot: check everything, THEN run)

Run **all** of these before any `lerobot-record`. Abort and fix if any check fails — never burn episodes on a broken setup.

```bash
LRBIN=/Users/dhirajdgandhi/opt/anaconda3/envs/lerobot/bin

# 1) Serial ports present (expect leader + the follower you'll use)
ls /dev/tty.usb*

# 2) BOTH arms actually answer on the bus (expect {1000000: [1, 2, 3, 4, 5, 6]} for each).
#    {} means the arm is UNPOWERED or a 3-pin cable is loose — fix power/cabling, not software.
$LRBIN/python -c "from lerobot.motors.feetech import FeetechMotorsBus; print('leader',  FeetechMotorsBus.scan_port('$LEADER_PORT'))"
$LRBIN/python -c "from lerobot.motors.feetech import FeetechMotorsBus; print('follower',FeetechMotorsBus.scan_port('$DG_FOLLOWER_PORT'))"

# 3) Hugging Face auth (expect user=...). If "Not logged in", run `hf auth login`.
NO_COLOR=1 $LRBIN/hf auth whoami

# 4) Cameras: capture sample frames, THEN VISUALLY inspect them.
$LRBIN/lerobot-find-cameras opencv          # writes outputs/captured_images/opencv_<idx>.png
#    Open outputs/captured_images/opencv_0.png and opencv_1.png and confirm each one
#    actually shows the WORKSPACE (panel + brush + gripper), not the ceiling/wall.
#    Apply the rule: "Can you do the task from this camera view alone?" If no, re-aim and re-run.
```

**Pre-flight pass criteria:**
- [ ] Leader **and** follower each return `{1000000: [1, 2, 3, 4, 5, 6]}`.
- [ ] `hf auth whoami` shows the expected user.
- [ ] `opencv_0.png` (top) and `opencv_1.png` (wrist) both clearly show the task workspace.
- [ ] macOS keyboard control: Terminal/Cursor has **Input Monitoring + Accessibility** permission (System Settings → Privacy & Security). Otherwise you'll see `WARNING ... This process is not trusted! Input event monitoring will not be possible` and the **→ / ← / ESC keys won't work** — recording still proceeds on the per-episode timer, but you lose manual next/redo/stop. macOS reads this permission at process start, so grant it *before* launching.

## Required code patch for this rig (already applied; keep it)

macOS AVFoundation + webcam autofocus causes brief (~520 ms) frame-delivery stalls that exceed the hardcoded 500 ms peek limit in `OpenCVCamera.read_latest()`, crashing recording. We bumped the tolerance in the follower's observation read:

```python
# src/lerobot/robots/so_follower/so_follower.py  (get_observation)
obs_dict[cam_key] = cam.read_latest(max_age_ms=2000)   # was read_latest() -> 500 ms default
```

A genuinely dead read thread still raises regardless of this value, so it only absorbs transient hiccups. If you ever `git checkout` this file, re-apply the patch.

## Calibrate (one-time per arm; reuse the `id` everywhere)

```bash
$LRBIN/lerobot-calibrate --teleop.type=so101_leader  --teleop.port=$LEADER_PORT       --teleop.id=dg_leader_arm
$LRBIN/lerobot-calibrate --robot.type=so101_follower --robot.port=$DG_FOLLOWER_PORT --robot.id=dg_follower_arm
$LRBIN/lerobot-calibrate --robot.type=so101_follower --robot.port=$JG_FOLLOWER_PORT --robot.id=jg_follower_arm
```

## Teleoperate (sanity check, no recording)

```bash
$LRBIN/lerobot-teleoperate \
  --robot.type=so101_follower --robot.port=$DG_FOLLOWER_PORT --robot.id=dg_follower_arm \
  --teleop.type=so101_leader  --teleop.port=$LEADER_PORT     --teleop.id=dg_leader_arm
```

Swap `$DG_FOLLOWER_PORT` / `dg_follower_arm` for `$JG_FOLLOWER_PORT` / `jg_follower_arm` to drive the other follower.

## Accounts (run once per machine — no tokens in files)

```bash
$LRBIN/hf auth login          # paste token interactively; do NOT hardcode it
$LRBIN/wandb login            # paste W&B key interactively

HF_USER=$(NO_COLOR=1 $LRBIN/hf auth whoami --format json 2>/dev/null | $LRBIN/python -c "import sys,json; print(json.load(sys.stdin)['user'])")
echo "$HF_USER"
```

## Record a dataset — KNOWN-GOOD COMMAND (start here)

This exact configuration ran stably on this rig (both cameras `1280×720`, streaming encoding **off**, with the `max_age_ms=2000` patch above). Adjust only `num_episodes`, `repo_id`, and `single_task`.

```bash
LRBIN=/Users/dhirajdgandhi/opt/anaconda3/envs/lerobot/bin

$LRBIN/lerobot-record \
  --robot.type=so101_follower --robot.port=$DG_FOLLOWER_PORT --robot.id=dg_follower_arm \
  --robot.cameras="{ top: {type: opencv, index_or_path: 0, width: 1280, height: 720, fps: 15}, wrist: {type: opencv, index_or_path: 1, width: 1280, height: 720, fps: 15} }" \
  --teleop.type=so101_leader --teleop.port=$LEADER_PORT --teleop.id=dg_leader_arm \
  --display_data=false \
  --dataset.fps=10 \
  --dataset.repo_id=${HF_USER}/solar-panel-clean-2cam \
  --dataset.num_episodes=20 \
  --dataset.single_task="Use the brush attached to the wrist to clean the solar panel and return to base position while holding the brush intact" \
  --dataset.streaming_encoding=false \
  --dataset.episode_time_s=20 \
  --dataset.reset_time_s=10
```

Notes that make this work (don't "optimize" them away without re-testing):
- **`1280×720` for BOTH cameras** — the only shared stable native resolution (see camera gotcha above).
- **`--dataset.streaming_encoding=false`** — real-time H.264/AV1 encoding starves this Intel Mac's CPU and causes camera-frame stalls. With it off, frames are written during capture and videos are encoded at the end. (A "running slower than target FPS / CPU starvation" warning is expected and harmless here.)
- `lerobot-record` **auto-appends a timestamp** to the repo id (e.g. `solar-panel-clean-2cam_20260630_081122`), so re-runs never collide and prior datasets are untouched.
- Recording keys: **→** next, **←** redo, **ESC** finish & upload (require the macOS permission from pre-flight).
- The side camera (index 3) is for documentation only; it is not part of the policy observation.

## Manual upload fallback

If auto-upload fails at the end of recording, push the local dataset folder by hand:

```bash
ls ~/.cache/huggingface/lerobot/${HF_USER}/                       # find the timestamped dataset folder
$LRBIN/hf upload ${HF_USER}/<dataset_folder> ~/.cache/huggingface/lerobot/${HF_USER}/<dataset_folder> --repo-type dataset
```
