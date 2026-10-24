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
| `--acquire-conf` | first lock / re-acquisition: detector score needed | 0.15 (tracking afterwards: 0.05 next to the target) |
| `--colour-min`, `--no-colour-check` | first lock: min CLIP probability of the prompt's hue (red, blue, yellow, ...; not black/white/dark), among the prompt in 11 colours | 0.15 |
| `--confirm-frames`, `--confirm-window` | first lock: the candidate must pass in this many of that many frames, robot standing still (state CONFIRM, orange box) | 3 / 6 |
| `--lock-conf` | once locked, weaker detections (down to this) next to the last box also count; boxes elsewhere are ignored (no switching to another object) | 0.05 |
| `--recheck-every`, `--recheck-min`, `--recheck-fails` | while tracking a hue prompt: re-check the colour every N frames; min probability; drop the lock after this many failures in a row | 5 / 0.08 / 3 |
| `--trk-frames`, `--trk-score` | template tracker (`tracker.py`) bridges at most this many frames after the last real detection; min match score | 5 / 0.6 (0 = off) |
| `--trk-frames-close`, `--close-ratio` | tracker budget when the target is close (box >= this share of the image height) | 25 / 0.18 (run07: below ~1 m the detector hardly fires) |
| `--max-turn` | max track speed difference for turning | 0.035 (was 0.06: with ~1 frame/s it over-turned) |
| `--attr-check` (+ `--attr-conf`) | experimental: detect the noun, keep crops CLIP finds closer to the full prompt than to the noun | off (removes the target for "a dark box", see section 9) |
| `--no-tiles` | no zoomed tiles in SEARCH / while stopped after a miss (tiles cost ~7x per frame) | off |
| `--roi-px` | zoomed crop around the target while its box is lower than this, px | 130 (was 60; run08: the pouf was found almost only with zoom) |
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
- **Lost the target up close (robot runs 2026-10-24, `runs/run05-06.csv`, red pouf).** Runs 1-3 arrived (stopped at
  0.44 / 0.43 / 0.45 m for 0.45 m). In 5-6 the detector found the pouf in only ~47% of tracking frames (score around
  0.10); on each miss the code switched to tiles (~1 frame/s) and kept turning with the last command -> the body
  over-turned (camera swung -125 -> +32 deg) and lost it. Fixed: lock-on hysteresis (`--lock-conf`), template
  tracker for up to 5 missed frames, body stops on a miss, tiles only in SEARCH, slower turning.
- **Locked onto the wrong object (run_10: "a blue chair" -> the red pouf, ARRIVED in 1.5 s).** First lock was as loose
  as tracking (one frame, score 0.10) and YOLO-World ignores colour words. Now: strict first lock (score 0.15, CLIP
  hue check, 3 of 6 frames standing still), relaxed tracking that ignores boxes away from the target. Checked on the
  lab2 chairs: "a blue chair" rejects all (<= 0.03), "a red chair" accepts only red ones (0.80-0.93).
- **Lock moved to another object while tracking (run11, ~39 s).** A box 5x smaller than the chair, ~150 px away, was
  'near' (centre within 200 px) -> new reference -> white box -> later the pouf. Now 'same object' = overlap, or close
  relative to the box size and similar size (weak boxes must overlap), using the box position predicted from the
  gimbal motion; plus a colour re-check every 5 frames while tracking.
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


---

# UPDATE 2026-10-24 evening -- full current state (added by Arsen + Claude Code)

Everything above is kept as it was. Everything below describes the code and results as of the evening of
2026-10-24 and **takes precedence where it differs** (e.g. tuning values in section 19 vs section 8 above).
**If you are finishing the midterm, start with section 12.**

## 12. TL;DR for the person finishing the midterm (deadline: form by end of 2026-10-25, talk 2026-10-26)

**State of the code:** `main` + branch `search-fixes` (PR open -- merge it first). The robot is given a text prompt,
searches with the pan-tilt camera, confirms the object, drives to it and stops ~0.45 m in front of it. On 2026-10-24
it was tested on the real robot in 14 driving runs; 7 reached the right object (section 20.4).

**What still has to be done for the midterm (in this order):**
1. Merge the PR `search-fixes` into `main` on GitHub (the GitHub link in the form should show everything).
2. Fill in "who did what" (section 23) -- each person presents their own part; absent = 0 points.
3. One sentence on the **scope change**: the proposal says autonomous navigation is out of scope; now the main demo
   is "find the object by text and drive to it". The tracking study (V1-V4) is still the core.
4. Jetson: say "not available -> plan B: laptop + Pi" (or ask the UGV club).
5. Slides: 5-7 (max 10), PDF, file named from the surnames, e.g. `Botsko_Veresniuk_Zarytskyi.pdf`.
   Section 12 = slide-by-slide outline, section 20 = all result tables.
6. Rehearse once with a timer (5 min; every extra minute = -1 point).
7. Submit the form: PDF + `https://github.com/Starezax/ugv`.

Optional if there is robot time: copy `latency_log.csv` from the Pi (`scp ws@192.168.50.5:~/latency_log.csv runs/`)
and compute end-to-end latency (mean / p95 of `latency_ms`) -- the only "before midterm" item of the proposal that is
still half done. Record `S6_seq05` (`./run_seq.sh S6_seq05 lab2`, lights dimmed).

---

## 13. What the project is

A Waveshare UGV Beast with a pan-tilt camera gets a text description ("a blue chair"), finds that object, locks the
camera on it and drives up to it. The detector is open-vocabulary (YOLO-World): the target is changed by typing, no
training. The core difficulty (and the research question of the proposal): the detector is **slow** (~0.1 s per
frame on the laptop CPU, 0.7-1.1 s with zoom), delayed by Wi-Fi, and **unreliable** (flickers, ignores colour words,
confuses similar objects), while the gimbal and the chassis need a smooth, correct signal.

Variants from the proposal (each adds one component, measured against the previous one):

| Variant | What | Status 2026-10-24 |
|---|---|---|
| V1 | gimbal driven directly by the latest detection (baseline) | done, measured (section 20.1) |
| V2 | + fast tracker between detections + PID, detector in its own thread | **partly**: a template tracker bridges detector misses (`tracker.py`); no separate thread, no PID |
| V3 | + Kalman filter with delay compensation (late detections applied at capture time) | not started (settle gating + gimbal-motion prediction are simple forms of "use the right time") |
| V4 | + identity check (CLIP template + text) for re-acquisition / distractors | **partly**: CLIP colour check at first lock + colour re-check while tracking; no visual template yet |
| Extension -> now main demo | search + drive to the object (`laptop_approach.py`) | works on the robot (section 20.4) |

Proposal: `Proposal_Veresniuk_Botsko_Zarytskyi-1.pdf` (scenarios S1-S6, metrics, plan, risks).

## 14. Status

| Item | Status |
|---|---|
| Camera stream Pi -> laptop, gimbal + track control from Python | done (`robot_bridge.py`, ZeroMQ) |
| Camera calibration | done 2026-10-11, `calib.npz` (fx ~ 273 px, HFOV ~ 100 deg, RMS 2.0 px) -- used for pixel->angle and the ground distance |
| V1 baseline | done; oscillation fixed with settle gating (2026-10-10); measured on recordings (section 20.1) |
| Sequences S1-S6 | 29/30 recorded 2026-10-18 (`S6_seq05` missing); quick hand-checked GT on 298 frames (`eval/`) |
| Detector choice | measured: YOLO-World v2 s @640 vs m/l/x, 1280 input, tiles, OWLv2, OWL-ViT (section 20.2) |
| Far objects | measured (object-size study) and improved with zoom (section 20.3) |
| Search + approach on the robot | works; 14 driving runs on 2026-10-24, arrivals at 0.43-0.45 m (section 20.4) |
| Latency | detector time measured (102 ms median); Pi end-to-end `latency_log.csv` **not analysed** |
| Jetson | not confirmed |
| CVAT annotation | not done (quick GT instead) |
| V2 / V3 / V4 full versions | after midterm (section 24) |

## 15. Setup on a new laptop (Linux, on internet)

```
git clone git@github.com:Starezax/ugv.git && cd ugv
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python -c "from ultralytics import YOLOWorld; m = YOLOWorld('yolov8s-worldv2.pt'); m.set_classes(['a chair']); print('ok')"
```
Wait for `ok` (downloads the YOLO-World weights and CLIP). CLIP ViT-B/32 is also expected in
`weights/clip/ViT-B-32.pt` (git-ignored; copy from Arsen's laptop, otherwise `clip` downloads it). The Pi side is
already set up (`~/robot_bridge.py`, venv `~/ugv_rpi/ugv-env`).

## 16. How to run

**Terminal 1 -- robot:**
```
ssh ws@192.168.50.5
pkill -f app.py                                         # the stock Waveshare app holds the UART and the camera
~/ugv_rpi/ugv-env/bin/python ~/robot_bridge.py --serial /dev/ttyAMA0
```
Expect `camera fps: ...` lines.

**Terminal 2 -- laptop:**
```
cd ~/ugv && source venv/bin/activate
# search + approach, gimbal only first (robot on a box):
python laptop_approach.py --robot 192.168.50.5 --prompt "a blue chair" --cam-height 0.27 --no-drive --log runs/test01.csv
# driving (floor, clear area, someone on SPACE):
python laptop_approach.py --robot 192.168.50.5 --prompt "a blue chair" --cam-height 0.27 --log runs/run15.csv
# V1 baseline (gimbal only):
python laptop_v1.py --robot 192.168.50.5 --prompt "a dark box"
```
`--cam-height` = camera lens height above the floor in metres (**measure it**; 0.27 was used on 2026-10-24).
Each run writes `runs/<name>.csv` (per-frame log) and `runs/<name>_args.json` (all settings incl. the prompt).

Keys (click the video window first): `q`/ESC = stop tracks and quit, SPACE = pause/resume driving (emergency stop),
`c` = stop, centre the camera, HOLD until SPACE. Ctrl+C in the terminal also stops.

On-screen box colours: **orange** = candidate being confirmed, **green** = detector, **cyan** = tracker.

**Without the robot** (tests of the laptop code):
```
python replay_bridge.py --seq rec/S1_seq06 --still 225 --sim --world-pan 50 --hide 22:32 &   # crude gimbal simulation
python laptop_approach.py --robot 127.0.0.1 --prompt "a dark box" --no-drive --headless --duration 45 --log /tmp/sim.csv
```
`--sim` places one recorded frame in a "world" at `--world-pan` deg; the camera view shifts with a simulated servo
(150 deg/s); `--hide A:B` removes the object for a while. Without `--sim` it just replays a recording.

**Recording sequences** (Zakhar's tool): `./run_seq.sh S6_seq05 lab2` (V1 + full-rate recording, 20 s).

**Evaluation** (offline; needs `rec/` = the recordings from Google Drive):
`python eval/run_detectors.py && python eval/metrics.py`, `python eval/size_study.py`. Details: `eval/README.md`.

## 17. How the search-and-approach code works now (`laptop_approach.py`, v7)

One loop iteration per received frame (always the newest frame; older ones are dropped).

### 17.1 Detection (`detector.py`)
- YOLO-World v2 **s**, the whole prompt as its only class, internal threshold 0.05 (filtered below).
- Three modes, chosen every frame:
  - `tiles`: whole frame + 4 overlapping tiles (2x2, 20% overlap), each upscaled to 640 -> ~1.8x zoom for small / far
    objects; ~0.7 s per frame (1.1 s on battery). Used in SEARCH, CONFIRM and while stopped after a miss.
  - `roi`: whole frame + a 320x240 crop around where the target should be, upscaled -> 2x zoom; ~0.3-0.45 s. Used while
    tracking a target lower than 130 px.
  - `full`: whole frame only; ~0.1-0.2 s. Tracking of large targets.
- Boxes from all crops are merged with NMS.

### 17.2 Strict first lock, relaxed tracking
- **Not locked** (SEARCH, CONFIRM, after a lost target): a box needs score >= 0.15 (`--acquire-conf`); if the prompt
  has a hue word (red, orange, yellow, green, blue, purple, pink, brown) CLIP must give that colour probability
  >= 0.15 (`--colour-min`) among the same prompt in 11 colours (the 8 hues + black, white, grey). Light/dark words
  (dark, black, white, grey) are not checked -- CLIP is unreliable there.
- **CONFIRM** (orange box, robot standing still): the same candidate (centre within 200 px of the previous one) must
  pass in 3 of 6 frames (`--confirm-frames`, `--confirm-window`); otherwise it is rejected and the search continues.
- **Locked** (TRACK / ARRIVED / LOST): a box counts only if it is the **same object** as the last box, *moved by the
  gimbal motion since* (~4.8 px per degree): IoU >= 0.3, or score >= 0.10 and 0.6-1.7x the height and centre closer
  than 0.75 x box size + 60 px. Weak boxes (0.05-0.10, `--lock-conf`) must overlap. Boxes elsewhere are ignored --
  it cannot switch to another object while tracking; a new object only through the strict first lock.
- **Template tracker** (`tracker.py`): when locked and the detector misses, the last detected patch is searched
  (normalised cross-correlation, 3 scales, score >= 0.6) around the predicted position; at most 5 frames after the
  last real detection (25 when the target is close: box >= 18% of the image height). Refreshed only by detections.
- **Colour re-check** (hue prompts): every 5th detection while locked; 3 failures (p < 0.08) in a row drop the lock.

### 17.3 Gimbal control (settle gating, from V1)
- A gimbal move needs ~0.4 s before the image shows it. Every command gets an index; the frame header echoes the angles
  applied at capture; a frame is "settled" only after the move is applied plus `--settle` + |move| / 200 deg/s.
- Only settled frames are used to compute the target bearing (`pan_cmd + error angle`; error angle from the pixel via
  the calibration, undistorted) and to make a correction: `new = old + 0.8 * (target - old)`, deadband 1.5 deg, max
  35 deg per step. Frames taken mid-move only keep the yaw feed-forward.
- Yaw feed-forward: while the chassis turns, the gimbal is counter-rotated by the estimated body rotation.

### 17.4 Chassis and arrival
- Bearing of the target relative to the body = settled gimbal pan + error angle; it must be < 2.5 s old to drive.
- Turn = 0.002 x bearing (deadband 5 deg), at most 0.035 (`--max-turn`); forward only if |bearing| < 12 deg.
- **Ground distance** to the object's base: `d = cam_height / tan(angle below horizon)`, angle = gimbal tilt + angle of
  the box's bottom edge (calibrated). Assumes the object stands on the floor.
- Speed 0.12 m/s, slowing over the last 1.0 m down to 25%. **ARRIVED** when d <= 0.45 m (`--arrive-dist`), or as a
  safety stop when the box is >= 85% of the image height (`--arrive-ratio`; objects not on the floor).

### 17.5 Losing and searching
- Up to 3 frames without the target (`--miss-frames`): body stops, camera holds, zoomed tiles. Up to 6
  (`--lost-frames`): LOST, still holding. Then SEARCH.
- SEARCH: first around the remembered area where the target was last seen (offsets 0, -20, +20, -40, +40 deg,
  2 rounds), then a full sweep -90..+90 deg in 30 deg steps (snake order), then (when driving) a short body turn and
  back to the remembered area. At each position it waits for 2 sharp (settled) frames (`--look-frames`, max 4 s).

### 17.6 Log columns (`runs/*.csv`)
`frame_id, t_capture, state, det_ms, found, box_ratio, dist_m, pan_cmd, tilt_cmd, yaw_ff_deg, L, R, moving, det_mode,
n_dets, src, conf, colour_p` -- `moving` = frame not settled; `src` = det / trk / empty; `conf` = score of the chosen
box; `colour_p` = CLIP colour probability (first-lock frames or re-check frames). Older logs have fewer columns.

## 18. Files

| File | Runs on | Purpose |
|---|---|---|
| `robot_bridge.py` | Pi | camera stream, command forwarding, latency log, speed cap + stop watchdog |
| `laptop_v1.py` | laptop | V1 baseline (gimbal only) |
| `laptop_approach.py` | laptop | search + confirm + track + drive (section 17) |
| `detector.py` | laptop | YOLO-World wrapper: full / tiles / roi modes, CLIP colour check, optional `--attr-check` |
| `tracker.py` | laptop | template tracker bridging detector misses |
| `replay_bridge.py` | laptop | fake bridge: replay a recording, or `--sim` crude gimbal simulation |
| `record.py`, `run_seq.sh` | laptop | record every frame (+ V1 tracking) for S1-S6 |
| `calibrate.py` | laptop | checkerboard calibration -> `calib.npz` |
| `eval/` | laptop | quick GT, detector benchmark (incl. OWLv2), metrics, object-size study; `eval/README.md` |
| `runs/` | -- | robot run logs of 2026-10-24 (section 20.4) |
| `README.md` | -- | short project overview + results |
| `snapshot.py`, `gimbal_test.py` | -- | mentioned in old notes, **not in git** (whoever has them: push) |

## 19. Tuning flags (`laptop_approach.py`)

| Flag | Meaning | Value (2026-10-24) |
|---|---|---|
| `--cam-height` | camera lens height above the floor, m -- **measure it** | 0.25 default; 0.27 used on the robot |
| `--arrive-dist`, `--slow-dist` | stop distance to the object's base; slow down over this distance | 0.45 / 1.0 m |
| `--arrive-ratio` | safety stop: box height / image height | 0.85 |
| `--acquire-conf` | first lock: detector score | 0.15 |
| `--colour-min`, `--no-colour-check` | first lock: min CLIP probability of the prompt's hue | 0.15 |
| `--confirm-frames`, `--confirm-window` | first lock: passes needed / frames allowed | 3 / 6 |
| `--conf`, `--lock-conf` | locked: normal score / weak boxes (must overlap) | 0.10 / 0.05 |
| `--recheck-every`, `--recheck-min`, `--recheck-fails` | colour re-check while tracking (hue prompts) | 5 / 0.08 / 3 |
| `--trk-frames`, `--trk-frames-close`, `--close-ratio`, `--trk-score` | tracker budget far / close; "close" box share; min match | 5 / 25 / 0.18 / 0.6 |
| `--roi-px`, `--no-tiles` | zoomed crop for targets lower than this; disable tiles | 130 px / off |
| `--miss-frames`, `--lost-frames` | missed frames ignored (stopped) / held as LOST before searching | 3 / 6 |
| `--local-offsets`, `--local-rounds` | search around the remembered area | 0,-20,20,-40,40 / 2 |
| `--sweep`, `--look-frames`, `--sweep-dwell` | full sweep; sharp frames per position; max s per position | -90..90 step 30 / 2 / 4 |
| `--spin-speed`, `--spin-time` | body turn after a failed full sweep | 0.05 / 1.2 s |
| `--max-speed`, `--max-turn`, `--k-turn`, `--align-deg`, `--turn-deadband` | chassis | 0.12, 0.035, 0.002, 12 deg, 5 deg |
| `--gain`, `--deadband`, `--max-step` | gimbal correction | 0.8, 1.5 deg, 35 deg |
| `--settle`, `--settle-dps`, `--settle-timeout` | settle gating | 0.15 s, 200 deg/s, 1.5 s |
| `--yaw-comp`, `--track-width` | gimbal counter-rotation while the body turns | 1.0, 0.20 m |
| `--pan-sign`, `--tilt-sign`, `--turn-sign`, `--drive-sign` | flip if a direction is wrong | all 1 (correct on our robot) |
| `--attr-check` | experimental noun + CLIP "prompt vs noun" check | off (fails on "a dark box") |
| `--headless`, `--duration`, `--log` | tests / log file | -- |

## 20. Results (for the slides)

### 20.1 V1 baseline, closed loop (recorded runs, 298 hand-checked frames, one every 2 s)
| scenario | target in frame | centring error mean / p95 | near centre (<10 deg) |
|---|---|---|---|
| S1 slow | 100% | 10.5 / 34.2 deg | 69% |
| S2 fast | 96% | 17.7 / 39.4 deg | 33% |
| S3 occlusion | 90% | 10.0 / 50.5 deg | 74% |
| S4 leaves frame | 71% | 20.3 / 54.1 deg | 38% |
| S5 distractor | 94% | 14.0 / 37.6 deg | 43% |
| S6 low light | 100% | 10.9 / 41.8 deg | 66% |
| **all** | **92%** | **13.3 / 46.0 deg** | **56%** |

GT = one person picked the target box among detector candidates (not CVAT; method and limits in `eval/README.md`).
Proposal goal for the full system: >= 90% in frame in S1-S4 -- V1 misses it in S4.

### 20.2 Detector choice (open loop, same frames, laptop CPU)
| detector | ms/frame | correct lock | wrong lock | false alarm (target absent) |
|---|---|---|---|---|
| **YOLO-World s @640 (chosen)** | **102** | **78%** | 15% | 76% |
| YOLO-World s @1280 | 425 | 59% | 26% | 80% |
| YOLO-World s @640 + tiles | 693 | 78% | 18% | 88% |
| YOLO-World m / l / x @640 | 265 / 551 / 806 | 74 / 50 / 57% | 9 / 3 / 4% | 56 / 36 / 48% |
| OWLv2 B/16 (proposal's alternative) | 4179 | 41% | 57% | 96% |
| OWL-ViT B/32 | 418 | 12% | 60% | 76% |
| s @640 + `--attr-check` | 275 | 5% | 13% | 44% |

Takeaways: s@640 is the fastest and the best; bigger models are slower and miss more at the same threshold; OWLv2 is
40x slower and locks onto look-alikes (best only in low light, S6 84%). The detector almost always "sees" something
when the target is absent (76%) -> wrong locks; this motivated the strict first lock and V4.

### 20.3 Far objects (object-size study: images shrunk 1-4x = the scene 1-4x farther away)
| config | ms/frame | box 80+ px | 40-80 px | 20-40 px | < 20 px |
|---|---|---|---|---|---|
| full frame (V1) | 112 | 69% | 68% | 27% | 3% |
| + tiles (used in SEARCH) | 748 | 67% | 79% | 44% | 22% |
| + ROI zoom (used in TRACK) | 295 | 78% | 75% | 60% | 44% |
| 1280 input | 456 | 64% | 46% | 36% | 24% |

With our 100 deg lens a 0.4 m object is ~22 px tall at 5 m and ~11 px at 10 m -> the "can't find it beyond 5 m"
problem; zoom recovers a large part.

### 20.4 Robot runs 2026-10-24 (logs in `runs/`)
Prompts were not logged that day (fixed now: `<log>_args.json`); the prompt column is from the chat, `?` = not sure --
**the operator should correct it**.

| log | code version | prompt | result |
|---|---|---|---|
| gimbal01 | v4 (main) | "a dark box" | gimbal only; "finds it but keeps rotating" -> v5 |
| gimbal_dist | v5 | ? | gimbal only, distance check, tracked the whole 40 s |
| run01 | v5 | ? | **arrived, stopped at 0.44 m** |
| run02 | v5 | ? | **arrived, 0.43 m** (detector saw the object in 49% of tracking frames) |
| run03 | v5 | ? | **arrived, 0.45 m** (99% detection) |
| run04 | v5 | ? | never found the object (searched 72 s) |
| run05 | v5 | red pouf ? | lost it up close (47% detection) |
| run06 | v5 | red pouf | drove 4 m, lost it at ~0.8 m -> v6 |
| run07 | v6 | "a red pouf" | **arrived 0.45 m**, but 6 needless LOST below 1 m -> close-range tracker budget |
| run08 | v6+ | "a red pouf" | detector saw the pouf only in 14/142 frames (mostly via zoom) -> zoom while stopped, wider ROI |
| run_09 | v6+ | ? | **arrived 0.44 m**, 100% detection, no losses |
| run_10 | v6+ | "a blue chair" | **wrong object**: locked onto the red pouf, "arrived" 0.35 m in 1.5 s -> v7 strict first lock |
| run11 | v7 | "a blue chair" | first take (overwritten): found the blue chair, then switched to a white box and the pouf -> same-object test + colour re-check. The current `run11.csv` is a re-run with the final code |
| run12 | v7 final | ? (no hue word) | **arrived 0.45 m**, no camera jumps |
| run13 | v7 final | hue prompt ? | confirmed + tracked, colour check passing (0.46-0.86); stopped by the operator before arrival |
| run14 | v7 final | ? (no hue word) | **arrived** by the 85% safety stop at 0.56 m (big object); 9 camera swings > 20 deg; re-acquired later |

**Summary for the slide:** 7 of 14 driving runs reached the right object; 6 of them stopped by the distance rule at
0.43-0.45 m (goal 0.45 m), 1 by the size safety stop at 0.56 m. One run locked onto a wrong object (fixed since);
the other failures (not found / lost up close / switched object) each led to a fix in section 21.

## 21. Work log 2026-10-24 (what was tried, why, what came out)

Commits: `main` (PR #2, merged) and `search-fixes` (PR open). In time order:

1. **Repo + evaluation** (PR #2): README, `requirements.txt`, `eval/` with quick GT on the recordings, V1 metrics,
   detector benchmark, object-size study, OWLv2 / OWL-ViT benchmark (section 20).
2. **Camera jumps in approach mode** (`approach_log.csv`: one missed detection swung the camera -96 -> -151 deg,
   14 jumps > 20 deg in 40 s). Cause: the bridge echoes a new pan as soon as it forwards the command, the servo needs
   ~0.4 s; frames in between counted the error twice, and the bearing / LOST logic used those frames. Fix: bearing
   only from settled frames, LOST holds the camera, max step 35 deg, nearest-box association.
3. **Far objects**: zoomed tiles (search) and ROI (tracking) in `detector.py`; measured in 10.3.
4. **YOLO-World ignores colour words**: "a yellow chair" boxed the red chair and the beige armchair. Tried a CLIP
   "prompt vs noun" check (`--attr-check`): good on chairs, but removes the real target for "a dark box" (82% missed)
   -> left optional. Later replaced by the hue-only colour check (point 9).
5. **Arrive closer**: the old rule "box >= 40% of the image height" stopped ~1.2 m from a chair (metres from a person).
   New: ground distance from camera height + tilt + calibration, stop at 0.45 m. Robot runs: 0.43-0.45 m.
6. **Robot test "finds it but keeps rotating"** (gimbal01): it found the box in a frame taken mid-sweep, did not
   update the direction, and the local search restarted at 57 deg while the box was at 32 deg, 3x in a row; timeouts
   in seconds (0.35 s) were shorter than one 0.7 s zoom frame. Fix (v5): remembered search area, timeouts in frames,
   slower stop-and-look search (30 deg steps, 2 sharp frames per position); `replay_bridge.py --sim` to test it.
7. **Lost the pouf up close** (run05/06): the detector saw it in ~47% of frames; on each miss the body kept turning
   with the last command at ~1 frame/s and over-turned (camera swung -125 -> +32 deg). Fix (v6): template tracker
   (`tracker.py`), weak detections next to the target count, body stops on a miss, tiles only in search, slower turns.
   Bug found in simulation: the tracker budget reset itself (161 tracker-only frames) -> fixed to "frames since the
   last real detection".
8. **run07 / run08**: below ~1 m the detector hardly fires, and the tracker budget ran out while the tracker still had
   the pouf -> 25-frame budget for close targets. run08: the pouf was found almost only with zoom -> zoom while
   stopped after a miss, ROI for targets up to 130 px. The laptop was on battery (zoom 45% slower) -- plug it in.
9. **Wrong object** (run_10: "a blue chair" -> red pouf). Fix (v7): strict first lock -- score 0.15, CLIP hue check
   (the prompt in 11 colours; black / white / grey were added as alternatives after a dark box passed as "blue" with
   0.23), 3 of 6 frames standing still; relaxed tracking. Checked on the recorded lab2 chairs: "a blue chair" rejects
   all (<= 0.03), "a red chair" accepts only the red ones (0.80-0.93), "a yellow chair" most tan chairs (0.25-0.63).
10. **Switched objects while tracking** (run11 first take, ~39 s): a box 5x smaller than the chair, ~150 px away,
    counted as "near" (centre within 200 px), became the reference, then a white box, later the pouf. Fix: size-aware
    same-object test, reference moved by the gimbal motion (a 35 deg centring move shifts it ~170 px), colour re-check
    every 5 frames. runs 12-14 with this code: 2 arrivals, steady tracking, no wrong locks.
11. Each run now writes `<log>_args.json` with the prompt and all settings.

## 22. Slide outline (5-7 slides, 5 minutes)

1. **Problem + pipeline**: text -> YOLO-World on the laptop -> gimbal + chassis. Why hard: 0.1-1 s per detection,
   0.4 s gimbal delay, 100 deg lens (small far objects), detector flicker and colour blindness (diagram in README).
2. **Model choice**: table 10.2 (YOLO-World s vs m/l/x, 1280, OWLv2) -> s@640.
3. **Baseline + metrics**: table 10.1 (V1 per scenario); metrics = in frame %, centring error, correct / wrong lock.
4. **What failed and why -> fixes**: camera jumps, far objects (table 10.3), stopping too far, losing the target up
   close, wrong object (blue chair -> pouf). One line each with the measured effect.
5. **Robot results + progress vs proposal**: approach runs (10.4: 7/14 arrived at 0.43-0.45 m); proposal checklist
   (stream/control, calibration, V1, recordings done; latency half; Jetson no); who did what (section 23).
6. **Plan to the final**: V2 full (CSRT/KCF tracker + detector thread + PID), V3 Kalman + delay compensation, V4 visual
   template identity; experiments: 4 variants x S1-S6 x 5 runs, detector-rate study 1/2/5/10 fps, approach success
   rate at 2/4/6 m; CVAT annotation (OWLv2 as pre-annotator); plan B per risk; if all works: detector on the robot,
   chained goals ("go to the chair, then the door").

## 23. Who did what (fill in before the talk)

| Person | Proposal role | Done (fill in / correct) |
|---|---|---|
| Arsen Botsko | gimbal control, calibration, tracker + Kalman, latency, robot integration | robot bridge, approach mode, 2026-10-24 robot tests and fixes (with Claude Code), ... |
| Ivan Zarytskyi | detector experiments, prompt study, CLIP identity, re-acquisition | settle gating fix, camera calibration (commits 2026-10-10/11), ... |
| Zakhar Veresniuk | recordings, annotation, evaluation, deployment | `record.py` + `run_seq.sh`, 29/30 sequences S1-S6 (2026-10-18), ... |

## 24. Known issues and next steps

Known issues:
- The detector is the weak point: some objects (the red pouf) are recognised in < 50% of frames, worse up close; the
  score depends a lot on the wording ("pouf" is rare -- "ottoman", "stool" or "cushion" may work better).
- The colour check works for real hues only; "a dark box" vs other dark objects is not solved (needs the V4 template).
- The ground distance assumes the object stands on the floor; for objects on a table the 85% safety stop is used.
- Run 14 still showed camera swings > 20 deg while tracking a big, close object.
- No obstacle avoidance. No separate detector thread: the gimbal updates only as fast as the detector (1-9 Hz).

After the midterm (owners from the proposal):
- **Arsen**: V2 (CSRT/KCF from opencv-contrib or a better tracker + detector thread + PID), V3 Kalman with delay
  compensation, latency numbers from `latency_log.csv`.
- **Ivan**: V4 (visual template of the locked object + CLIP text; re-acquisition among look-alikes), prompt study.
- **Zakhar**: CVAT annotation (OWLv2 pre-annotation), evaluation harness for detector-rate replay, `S6_seq05`, run on
  the Pi itself / Jetson.
- Experiments: 4 variants x S1-S6 x 5 runs; approach success rate and stop distance at 2/4/6 m.
