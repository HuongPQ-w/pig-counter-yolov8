# Pig Counter – Automatic Pig Counting Web App

A local web app that counts pigs passing through a barn gate in video files or live RTSP camera streams, using **YOLOv8 + object tracking** and a **dual-gate line-crossing algorithm** that correctly handles pigs turning back.

## Features

- **Two input sources**: upload a video (drag & drop) or connect an **RTSP camera**.
- **Draw the counting zones by hand** on a real frame: barn area (free-form polygon ROI), entry gate (green line) and exit gate (pink line). No hardcoded coordinates, so it works with any camera position.
- **Live annotated stream** (MJPEG) with per-batch statistics: pigs in the barn, total counted, pigs per batch.
- **Hot re-draw**: pause and redraw the ROI/gates on the latest frame without reloading the video.
- **Export results**: annotated video, per-batch CSV (`_batches.csv`) and per-crossing-event CSV (`_events.csv`).
- **CPU-friendly**: ByteTrack by default, configurable inference size, optional OpenVINO export, thread control for multi-camera setups.

## Counting algorithm

- Counts **physical crossings of a line**, independent of fixed track IDs, so a pig that turns around (and gets a new tracker ID) is still subtracted correctly.
- The confirmation threshold for the forward direction (+1) is stricter than for the reverse direction (-1), because pigs that turn back are often crowded or occluded and have lower confidence.
- The cooldown only suppresses repeated events in the **same direction** (jitter), never opposite-direction events.
- A batch ends automatically when the barn stays empty for 3 consecutive seconds.

## RTSP handling

- Sets `OPENCV_FFMPEG_CAPTURE_OPTIONS` (TCP transport, large buffer, sensible timeout) before importing `cv2`.
- The RTSP stream runs in its own thread and **reconnects automatically** when the connection drops.
- Live view via MJPEG (`multipart/x-mixed-replace`): lightweight, no WebRTC needed, works in any browser.

## Tech stack

Python · Flask · Ultralytics YOLOv8 (custom-trained pig detector) · ByteTrack · OpenCV · PyTorch · HTML/CSS/JavaScript (canvas drawing)

## Project structure

```text
app.py              Flask server (routes, RTSP snapshot, job control)
engine.py           Counting engine (dual-gate logic, tracking, CSV export)
best.pt             Trained YOLOv8 pig detection model
requirements.txt
run.bat / run.sh    One-click launch (Windows / Linux-macOS)
install.bat         First-time environment setup (Windows)
stop.bat            Stop the app
templates/index.html
static/             CSS, JS, icon
uploads/, outputs/  Created at runtime (not committed)
```

## Getting started

**Windows:** double-click `run.bat` (it asks you to run `install.bat` once first; the first install can take 10–20 minutes because PyTorch/Ultralytics are large).

**Linux/macOS:**

```bash
chmod +x run.sh
./run.sh
```

**Manual:**

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python app.py
```

Open <http://127.0.0.1:5000>. Requires Python 3.9+.

### Usage

1. **Choose a source**: upload a video or paste an RTSP URL and click *Connect*.
2. **Draw** the barn ROI polygon, the entry gate (2 points) and the exit gate (2 points) on the frame.
3. Click **Start counting** and watch the live stream and batch statistics.
4. Use the **gear button** to pause and redraw zones at any time.
5. When finished, download the annotated video and the CSV files.

## Running well on CPU-only machines

| Option | How |
|---|---|
| Lighter tracker (default) | ByteTrack is the default. Switch back with `PIG_TRACKER=botsort.yaml`. |
| Smaller inference size | `PIG_IMGSZ=480` or `416` (default 640): faster, slightly less accurate on small/far objects. |
| OpenVINO export (biggest speed-up, typically 2–4×) | `pip install openvino` then `python -c "from ultralytics import YOLO; YOLO('best.pt').export(format='openvino', imgsz=640, half=False)"`. The app detects `best_openvino_model/` automatically on the next run. |
| Multiple cameras on one CPU | Set `PIG_TORCH_THREADS=<cores per job>` (e.g. 4 for 2 cameras on an 8-core machine) so jobs don't starve each other. |

## Limitations & future work

- Counting accuracy depends on camera angle and gate placement; very dense groups crossing together can still cause undercounting.
- Single shared model with a lock: jobs are processed one inference at a time.
- Flask's development server is intended for local use only.

## License

MIT
