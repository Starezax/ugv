# Handover: UGV text-guided tracking (AI course project)

Last updated: 2026-10-24 by Zakhar (29/30 sequences recorded, `record.py` + `run_seq.sh`); 2026-10-24 midterm fixes
(camera-jump fix, zoom for far objects, quick GT + metrics in `eval/`)

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
| Search + approach mode | first version works; 2026-10-24: camera-jump bug fixed, zoom tiles/ROI for far objects; only replay-tested -- **needs robot runs** (`--no-drive` first) |
| Camera calibration | done 2026-10-11, `calib.npz` in the repo (fx ~270 px, HFOV ~100 deg, RMS 2.0 px), see section 9 |
| Latency measurement | detector 102 ms/frame median on the laptop CPU (`eval/`); Pi end-to-end log still not analysed |
| Sequence recording (S1-S6) | 29/30 done 2026-10-18, only `S6_seq05` missing -- see section 6; quick GT on 308 frames in `eval/gt.json` |
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

### Recording evaluation sequences

One command per sequence: starts V1 tracking, waits for the model, records 20 s of every camera frame, stops V1.

```
./run_seq.sh S6_seq05 lab2                              # prompt "a dark box", 20 s
PROMPT="a red backpack" DURATION=30 ./run_seq.sh S1_seq07 lab3
```

Output: `rec/<name>/` (frames + `frames.csv` + `meta.json`) and `logs/<name>_v1.csv`. To redo a take:
`rm -r rec/<name> logs/<name>_v1.csv`. `record.py` alone records without tracking.

Recorded so far (prompt `a dark box`, 20 s each, ~15 fps):

| Scene | Robot position | Split | Sequences |
|---|---|---|---|
| `lab1` | facing the blue board | validation | S1_seq02-03, S2-S6 seq01-02 |
| `lab2` | turned to the opposite side (red chair) | test | S1_seq04-06, S2-S5 seq03-05, S6_seq03-04 |

**Still to record:** `./run_seq.sh S6_seq05 lab2` (robot facing the lab2 side, lights dimmed).
`S1_seq01` is an old test take and is not part of the dataset. The recordings (~400 MB) are not in git;
Zakhar has them.

Other tools:

```
python snapshot.py --robot 192.168.50.5 --out view.jpg --prompt "a red backpack"
python calibrate.py --robot 192.168.50.5 --cols 9 --rows 6 --square 0.025
```

Headless, time-limited test: `laptop_approach.py --headless --duration 20`. Without the robot, stream a recording
with `replay_bridge.py --seq rec/S1_seq04 --loop` and point the laptop script at `--robot 127.0.0.1` (the image
does not react to commands, so this checks that everything runs, not the control). `snapshot.py`,
`gimbal_test.py` are still missing from git -- whoever has them, please push.

## 7. Files

| File | Runs on | Purpose |
|---|---|---|
| `robot_bridge.py` | Pi | camera stream, command forwarding, latency log, drive speed cap + stop watchdog |
| `gimbal_test.py` | Pi | checks UART, gimbal directions, angle feedback |
| `laptop_v1.py` | laptop | V1 baseline (gimbal only) |
| `record.py` | laptop | records every frame + capture time + gimbal angles to `rec/<name>/` |
| `run_seq.sh` | laptop | V1 + `record.py` in one command, for the S1-S6 sequences |
| `laptop_approach.py` | laptop | search + gimbal + chassis approach, `--headless --duration` |
| `detector.py` | laptop | YOLO-World text detector + zoom tiles (search) / ROI (far target), optional CLIP attribute check |
| `replay_bridge.py` | laptop | fake bridge streaming a recording, for tests without the robot |
| `eval/` | laptop | quick ground truth, detector benchmark, metrics, object-size study (`eval/README.md`) |
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
| `--gain` | multiplier on the measured angular error per move | 0.8 for both, now with calibration. (Gain 2.0 worked before calibration because the assumed 65 deg FOV made every error ~1.85x too small -- see section 9) |
| `--max-step` | max gimbal change per correction, deg (approach) | 35 since 2026-10-24 (100 let one bad bearing swing the camera 55 deg; 8 capped tracking at ~20 deg/s) |
| `--miss-frames` / `--lost-frames` | missed detector frames ignored / held still (LOST) before searching again; frames, not seconds (a tiles frame takes 0.7 s) | 3 / 6 |
| `--local-offsets`, `--local-rounds` | search positions around the area where the target was last seen, and how often to repeat them before a full sweep | 0,-20,20,-40,40 / 2 |
| `--sweep`, `--look-frames`, `--sweep-dwell` | full sweep positions (snake order); sharp frames to check at each position; max s per position | -90..90 step 30 / 2 / 4.0 |
| `--spin-speed`, `--spin-time` | chassis turn after a failed full sweep | 0.05 / 1.2 s |
| `--attr-check` (+ `--attr-conf`) | experimental: detect the noun, keep crops CLIP finds closer to the full prompt than to the noun | off (removes the target for "a dark box", see section 9) |
| `--no-tiles` | no zoomed tiles in SEARCH (tiles cost ~7x per frame) | off |
| `--roi-px` | zoomed crop around the target while it is lower than this, px | 60 |
| `--settle` | s to wait after a move is applied before trusting detections | 0.15 (lower values change nothing: the wait is dominated by command transit + next frame capture, ~65 ms/frame) |
| `--settle-dps` | assumed gimbal speed; adds `|move| / this` to the settle time | 200 for both (approach also worked with 100). Tracking speed is currently limited by the uncalibrated camera and the frame rate (~15 fps, ~0.4 s per correction cycle), not by these flags |
| `--no-settle` | old behaviour, correct on every detection (oscillates) | only for comparison |
| `--yaw-comp` | gimbal counter-rotation while chassis turns | 1.0 default |
| `--arrive-dist` | stop when the object's base on the floor is this close to the camera, m | 0.45 (2026-10-24; before, the 0.40 box-ratio rule stopped ~1.2 m from a chair) |
| `--cam-height` | camera lens height above the floor, m -- **measure it**, the distance scales with it | 0.25 (guess!). The S1-S6 recordings were made with the robot on a desk (~1.0 m) |
| `--slow-dist` | slow down over this distance before the stop point, m | 1.0 |
| `--arrive-ratio` | safety stop: box height / image height >= this (objects not on the floor, e.g. on a table) | 0.85 |

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
- **Camera calibration (2026-10-11).** Two runs agree: fx = 267-273 px, fy = 264-274 px, cx ~ 331-338, cy ~ 241-243,
  horizontal FOV ~ 100 deg (not 65 deg as assumed before), strong barrel distortion (k1 ~ -0.21).
  - The old 65 deg guess means fx ~ 500, so error angles were ~1.85x too small -- that is why gain 2 "worked".
  - RMS is 2.0 px, not < 0.5 px. Accepted: 2 px at fx ~ 270 is ~0.4 deg, below the tracking deadband (1-1.5 deg).
    Likely causes if someone wants it lower: motion blur from the hand-held board, 640x480 JPEG stream,
    default 5-coefficient model on a 100 deg lens (try `cv2.CALIB_RATIONAL_MODEL`).
  - Both laptop scripts now undistort the box center (`cv2.undistortPoints`) before computing the angle.
    At the image edge this matters: true angle ~60 deg vs 48 deg pinhole-only (and 32 deg with the old guess).
- **`pan_fb` / `tilt_fb` from the ESP32 are not usable as the real gimbal angle**: seen `pan_fb = -180` while
  the command was -69. That is why the code uses the commanded angle + settle gating instead.
- **Approach mode camera jumps (fixed 2026-10-24).** The bridge echoes the new `pan_cmd` as soon as it forwards a
  command, but the servo takes ~0.4 s. Frames in between show the old view labelled with the new angle, so
  `pan_cmd + error` counts the error twice. The correction was gated, but `last_pan_seen` and the chassis bearing
  were still updated from those frames, and one missed detection (LOST) sent the gimbal there at full speed
  (`approach_log.csv`: -96 -> -151 deg, 14 jumps > 20 deg in 40 s). Now: bearing only from settled frames, short gaps
  ignored, LOST holds the gimbal, max step 35 deg, nearest-box association.
- **"Finds it but keeps rotating" (robot test 2026-10-24, `runs/gimba_dist.csv`).** Found in a frame taken mid-sweep
  -> direction not updated -> local search restarted at the old angle (57 deg) while the box was at 32 deg, three
  times in a row. Also 0.35 s / 1.0 s timeouts were shorter than one 0.7 s tiles frame, so one miss = LOST.
  Fixed: remembered search area (`area_pan`), timeouts in frames, slower stop-and-look sweep.
- **YOLO-World ignores attribute words (2026-10-24).** "a yellow chair" boxes the red chair and the beige armchair;
  it acts as a "chair" detector. Tried: detect the noun, keep crops that CLIP finds closer to the full prompt than
  to the bare noun (`--attr-check`). Separates yellow from red chairs, but for "a dark box" it drops the real box
  (82% missed, the distractors are dark too). Not general -> off by default; distractors are V4's job (template).
- **Detector "sees" the target when it is not there.** With "a dark box" at conf 0.10, 76% of frames without the
  target still have a box (black devices, monitors) -> wrong lock in S4. Motivation for V4 (identity check).
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
1. ~~Record ~30 sequences of 10-30 s covering S1-S6~~ -- 29/30 done, `S6_seq05` left (section 6).
2. Annotate in CVAT (every 5th frame + interpolation), split validation/test by scene.
3. Evaluation harness: replay recordings with throttled detector rate (1, 2, 5, 10 fps), compute center error, IoU, time in frame, recovery time, wrong-lock rate.
4. Run the pipeline on the Pi itself (`--robot 127.0.0.1`) and on a Jetson if the club has one.

## 11. Safety

- Test any driving change with the tracks off the ground first.
- Clear area, someone watching, hand on SPACE. No obstacle avoidance exists.
- The bridge caps track speed at 0.25 m/s and stops the tracks if no drive command arrives for 1.5 s. Don't raise these without agreeing as a team.
- Emergency stop from any terminal: `ssh ws@192.168.50.5 "pkill -f robot_bridge.py"`, or power off.
