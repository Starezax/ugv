# Handover: UGV text-guided tracking (AI course project)

Last updated: 2026-10-10 by Ivan (settle gating fix, keys)

## 1. What the project is

A Waveshare UGV Beast keeps its Pan-Tilt camera locked on one object described by text (e.g. "a red backpack").
The detector (YOLO-World, open-vocabulary) is slow, so the project builds fast components around it and measures
how much each one helps:

- V1: gimbal driven directly by the latest detection (baseline) -- DONE, works on the robot
- V2: + fast tracker (CSRT) between detections + PID -- TODO
- V3: + Kalman filter with delay compensation (late detections applied at capture time) -- TODO
- V4: + CLIP identity check for re-acquisition (distractors of the same class) -- TODO (Ivan)

Extension (not part of the graded scope unless the mentor agrees): search for the object and drive to it
(`laptop_approach.py`) -- first version works, see section 6.

Full plan, scenarios S1-S6 and metrics: see the proposal PDF.

## 2. Status

| Item | Status |
|---|---|
| Camera stream Pi -> laptop | done |
| Gimbal control from Python | done |
| V1 baseline on the robot | done; oscillation on a stationary target fixed with settle gating (2026-10-10) |
| Search + approach mode | first version works; settle gating + new gimbal defaults tested with `--no-drive` (gimbal only) 2026-10-10; driving not re-tested |
| Camera calibration | NOT done (needs printed 9x6 inner-corner checkerboard) |
| Latency measurement | logging is in place, numbers not analysed yet |
| Sequence recording (S1-S6) | not started (`--record` option is ready) |
| Jetson availability | not confirmed (ask UGV club) |
| V2 / V3 / V4 | not started |

## 3. Hardware and how it is wired

- **Raspberry Pi** (on the robot): runs `robot_bridge.py` -- camera capture + timestamps, streams JPEG frames,
  forwards commands to the ESP32 over UART (`/dev/ttyAMA0`), logs latency.
- **ESP32** (on the robot): drives the gimbal (JSON `T:133`) and the tracks (JSON `T:1`).
- **Laptop**: runs YOLO-World and all control logic (`laptop_v1.py`, `laptop_approach.py`).

Communication: ZeroMQ. Pi publishes frames on tcp 5555, laptop sends commands to tcp 5556.

## 4. Networks -- read this, it cost us time

The robot has TWO Wi-Fi networks. Use the Pi one.

| SSID | Password | Device | IP | Use |
|---|---|---|---|---|
| `AccessPopup` | `1234567890` | Raspberry Pi | `192.168.50.5` | SSH, Jupyter (:8888), stock web app (:5000), our bridge |
| `UGV` | `12345678` | ESP32 | `192.168.4.1` | ESP32 web page only. NO SSH, NO Jupyter. Don't use. |

- Pi login: `ws` (password `ws` on the stock image; confirm with the team if it was changed).
- Neither network has internet. Download the model and install packages BEFORE connecting to the robot,
  or use phone USB tethering for internet while Wi-Fi is on `AccessPopup`.
- To use internet + robot at the same time, stop the robot network from taking the default route (once):

```
nmcli connection modify AccessPopup ipv4.never-default yes
nmcli connection modify AccessPopup ipv6.never-default yes
nmcli connection down AccessPopup
nmcli connection up AccessPopup
```

Check:

```
ip route
ping -c 3 8.8.8.8
ping -c 3 192.168.50.5
```

## 5. Setup on a new laptop (Linux)

On internet Wi-Fi:

```
sudo apt install python3-venv
python3 -m venv ~/ugv-laptop-env
source ~/ugv-laptop-env/bin/activate
pip install ultralytics pyzmq opencv-python
mkdir -p ~/Desktop/ugv
cd ~/Desktop/ugv
python -c "from ultralytics import YOLOWorld; m = YOLOWorld('yolov8s-worldv2.pt'); m.set_classes(['a red backpack']); print('ok')"
```

Wait for `ok` -- this downloads the model weights and CLIP. Put the project files into `~/Desktop/ugv`.

Optional, passwordless SSH (needed for Claude Code):

```
ssh-keygen -t ed25519
ssh-copy-id ws@192.168.50.5
```

The Pi side is already set up (pyzmq installed in `~/ugv_rpi/ugv-env`, scripts in `~/`).

## 6. How to run

### Terminal 1 -- robot (SSH)

```
ssh ws@192.168.50.5
pkill -f app.py
~/ugv_rpi/ugv-env/bin/python ~/robot_bridge.py --serial /dev/ttyAMA0
```

`pkill -f app.py` stops the stock Waveshare app (it holds the UART and camera). Expect `camera fps: ...` lines.

### Terminal 2 -- laptop

```
source ~/ugv-laptop-env/bin/activate
cd ~/Desktop/ugv
python laptop_v1.py --robot 192.168.50.5 --prompt "a red backpack" --no-control
python laptop_v1.py --robot 192.168.50.5 --prompt "a red backpack"
```

Search and approach -- first with the robot on a box (tracks off the ground), then on the floor in a clear area:

```
python laptop_approach.py --robot 192.168.50.5 --prompt "a red backpack" --no-drive
python laptop_approach.py --robot 192.168.50.5 --prompt "a red backpack"
```

Keys -- **click the video window first**; on macOS keys typed in the terminal never reach OpenCV:

| Key | `laptop_v1.py` | `laptop_approach.py` |
|---|---|---|
| `q` / ESC | quit | stop tracks and quit |
| `c` | re-center gimbal + pause tracking | stop tracks, re-center gimbal, HOLD (no tracking / search) |
| SPACE | pause / resume tracking | pause / resume driving; leaves HOLD |
| Ctrl+C (terminal) | quit | stop tracks and quit |

`c` pauses on purpose: before, the next detection turned the gimbal straight back to the target, so `c` looked broken.

Other tools:

```
python snapshot.py --robot 192.168.50.5 --out view.jpg --prompt "a red backpack"
python calibrate.py --robot 192.168.50.5 --cols 9 --rows 6 --square 0.025
python laptop_v1.py --robot 192.168.50.5 --prompt "a red backpack" --no-control --record rec/S1_seq01
```

Headless, time-limited test (for SSH / Claude Code) -- NOT in the repo yet: `--headless`, `--duration` and
`--save-dir` are not implemented in `laptop_approach.py`, and `snapshot.py` / `gimbal_test.py` are missing from
the initial commit. Whoever has them, please push.

## 7. Files

| File | Runs on | Purpose |
|---|---|---|
| `robot_bridge.py` | Pi | camera stream, command forwarding, latency log, drive speed cap + stop watchdog |
| `gimbal_test.py` | Pi | checks UART, gimbal directions, angle feedback |
| `laptop_v1.py` | laptop | V1 baseline (gimbal only), `--record` for sequences |
| `laptop_approach.py` | laptop | search + gimbal + chassis approach, headless options |
| `calibrate.py` | laptop | checkerboard calibration -> `calib.npz` (loaded automatically) |
| `snapshot.py` | laptop | save one frame (optionally with detections) |
| `CLAUDE.md` | -- | context and safety rules for Claude Code |

Logs: `latency_log.csv` on the Pi (capture -> command sent, Pi clock), `v1_log.csv` / `approach_log.csv` on the laptop.
Copy the Pi log:

```
scp ws@192.168.50.5:~/latency_log.csv ~/Desktop/ugv/
```

## 8. Tuning flags (fill in what works on our robot)

| Flag | Meaning | Our value |
|---|---|---|
| `--pan-sign` | -1 if gimbal pans away from target | ? |
| `--tilt-sign` | -1 if gimbal tilts away from target | ? |
| `--turn-sign` | -1 if chassis turns away from target | ? |
| `--drive-sign` | -1 if "forward" drives backward | ? |
| `--gimbal-spd` / `--gimbal-acc` | gimbal speed on ESP32 (0 = fastest) | 0 / 0 (the old 50 / 10 was a workaround for the overshoot, not needed with settle gating) |
| `--gain` | multiplier on the measured angular error per move | V1: 0.8, approach: 2.0 (tested 2026-10-10). Gain > 1 should overshoot in theory; most likely `--hfov 65` underestimates the real FOV, so errors are measured too small. Re-check after calibration, expect ~0.8-1.0 |
| `--max-step` | max gimbal change per correction, deg (approach) | 60 (was 8: with settle gating, 8 deg per ~0.4 s capped tracking at ~20 deg/s) |
| `--settle` | s to wait after a move is applied before trusting detections | 0.15 (lower values change nothing: the wait is dominated by command transit + next frame capture, ~65 ms/frame) |
| `--settle-dps` | assumed gimbal speed; adds `|move| / this` to the settle time | V1: 200, approach: 100 (tested) |
| `--no-settle` | old behaviour, correct on every detection (oscillates) | only for comparison |
| `--yaw-comp` | gimbal counter-rotation while chassis turns | 1.0 default |
| `--arrive-ratio` | stop when box height / image height >= this | 0.40 default -- fine for a backpack; a standing person is already 0.5-0.6 at a few metres, so for people use ~0.8 or the robot never drives |

## 9. Problems we hit and fixes

- `source: not found` on the Pi: shell is `sh`. Use `. ~/ugv_rpi/ugv-env/bin/activate` or call `~/ugv_rpi/ugv-env/bin/python` directly.
- SSH "connection refused" to 192.168.4.1: that is the ESP32 network (`UGV`). Connect to `AccessPopup`, use 192.168.50.5.
- `laptop_v1.py` hangs with no window: Ultralytics checks internet on import. Fixed in code (`YOLO_OFFLINE=true`). If it fails at model loading, the model/CLIP wasn't downloaded -- run the `print('ok')` command on internet.
- Scripts must run on the right machine: `robot_bridge.py` / `gimbal_test.py` on the Pi only.
- **V1 camera swung back and forth on a stationary target (fixed 2026-10-10).** After a gimbal command it takes
  ~0.4 s (~6 frames) before the image shows the move. V1 computed "angle at capture + error" on every frame,
  so the same -24 deg error was added 6 times: pan went +35 -> -68 -> +93 ... (see `v1_log.csv`). Fix:
  *settle gating* -- after a move, ignore detections until a frame is captured with that move already applied
  (the frame header echoes the bridge's applied `pan_cmd`/`tilt_cmd`) plus `--settle` + `|move|/--settle-dps`.
  Same logic in `laptop_approach.py` (uses an index of sent commands, because yaw feed-forward changes the
  angle every frame; feed-forward keeps running while waiting). Log column `moving` = frame was gated.
- **`pan_fb` / `tilt_fb` from the ESP32 are not usable as the real gimbal angle**: seen `pan_fb = -180` while
  the command was -69. That is why the code uses the commanded angle + settle gating instead.
- Approach mode: gimbal was too fast -> SPD/ACC limits + max step per update. Camera lost the target when the chassis turned -> yaw feed-forward (gimbal counter-rotates), gentler turning, local search around the last seen bearing. Still jerky -- the real fix is V2/V3.

## 10. Next steps (by owner, from the proposal)

**Arsen** -- gimbal control and fast loop
1. Camera calibration (`calibrate.py`), RMS < 0.5 px.
2. Latency numbers: mean and p95 of `latency_ms`, plus `det_ms`.
3. V2: CSRT tracker between detections + PID; detector in a separate thread so the gimbal updates every frame.
4. V3: Kalman filter (constant velocity on box center/size) with a frame/state history buffer; late detections applied at capture time and replayed to now.

**Ivan** -- detection and identity
1. Prompt study with YOLO-World (and OWLv2 as heavier alternative): which phrasings work, detection speed.
2. V4: store a template crop at first lock; on re-acquisition score candidates by CLIP similarity to text + template; tune threshold on validation sequences.

**Zakhar** -- data, evaluation, deployment
1. Record ~30 sequences of 10-30 s covering S1-S6 (`--record rec/<scenario>_seqNN`).
2. Annotate in CVAT (every 5th frame + interpolation), split validation/test by scene.
3. Evaluation harness: replay recordings with throttled detector rate (1, 2, 5, 10 fps), compute center error, IoU, time in frame, recovery time, wrong-lock rate.
4. Run the pipeline on the Pi itself (`--robot 127.0.0.1`) and on a Jetson if the club has one.

## 11. Safety

- Test any driving change with the tracks off the ground first.
- Clear area, someone watching, hand on SPACE. No obstacle avoidance exists.
- The bridge caps track speed at 0.25 m/s and stops the tracks if no drive command arrives for 1.5 s. Don't raise these without agreeing as a team.
- Emergency stop from any terminal: `ssh ws@192.168.50.5 "pkill -f robot_bridge.py"`, or power off.
