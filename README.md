# Foot Traffic Monitor to Evaluate Storefront Activity in Plaza Colón, Mayagüez

<img width="1222" height="767" alt="Screenshot 2026-03-31 at 6 14 29 PM" src="https://github.com/user-attachments/assets/295d28a2-2a53-4702-bff2-1891c6538453" />

This community project is done in collaboration with Perspectivas Globales, a program by OverComing Adversities Inc. (https://ocapr.org). We've deployed on Raspberry Pi 4B a storefront people and car counter using a Pi Camera, quantized EfficientNet architecture, a lightweight centroid tracker, and virtual line-crossing logic. We also showcase the use of CEMI (https://cemi.capicu.ai) to monitor and evaluate deployment workflows for our compressed model.

## What is included

- `main.py`: runnable entrypoint
- `foot_traffic/`: detector, tracker, counter, camera, and event output modules
- `config.example.yaml`: editable runtime config
- `scripts/download_model.sh`: helper to fetch a COCO TFLite model and labels
- `tests/`: focused unit tests for line-crossing logic

## Pipeline

`Picamera2 -> TFLite detector -> centroid tracker -> line crossing -> CSV/MQTT`

The default config filters detections down to `person` and `car`, which are standard COCO classes.

## Quick start

1. Create a virtual environment:

   ```bash
   sudo apt install -y python3-picamera2
   python3 -m venv .venv
   source .venv/bin/activate
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

   On newer Raspberry Pi Python builds, this project uses `ai-edge-litert` as the TFLite runtime inside the venv.
   If your venv cannot see `Picamera2`, the app also falls back to the Raspberry Pi OS system package at `/usr/lib/python3/dist-packages`.

2. Copy the sample config:

   ```bash
   cp config.example.yaml config.yaml
   ```

3. Download a model and labels:

   ```bash
   chmod +x scripts/download_model.sh
   ./scripts/download_model.sh
   ```

4. Run the monitor:

   ```bash
   python3 main.py --config config.yaml
   ```

   If you launch it over SSH without a desktop session, the app now auto-disables the OpenCV preview and continues in headless mode.

## Camera sources

- `picamera2`: default for a Raspberry Pi camera module
- integer like `0`: USB webcam
- file path: recorded video for testing

Set `camera.source` in `config.yaml`.

You can also adjust orientation with:

```yaml
camera:
  rotation: 0
  flip_vertical: true
```

Pi Camera frames are captured as `RGB888` for the current Pi camera path. On this setup we avoid an extra manual channel swap because it can invert reds and blues in the preview.

For better Pi 4B performance, a good starting point is:

```yaml
camera:
  width: 640
  height: 480
  fps: 30

detector:
  max_results: 5
  num_threads: 4
  detect_every_n_frames: 1
  detect_interval_ms: 150
```

## Choosing the counting line

Edit:

```yaml
counter:
  line:
    start: [320, 80]
    end: [320, 400]
```

This line is drawn on the preview. Each tracked object is counted once when its centroid crosses from one side of the line to the other.

Direction labels are inferred from the line orientation:

- mostly vertical line: `left_to_right` and `right_to_left`
- mostly horizontal line: `top_to_bottom` and `bottom_to_top`

## Output

- CSV rows with timestamp, class, direction, confidence, track id, centroid, and bounding box
- Optional MQTT publishing for downstream dashboards or automations
- Optional annotated video recording

## CEMI Logging

If `cemi-cli` is installed, the monitor can log real runtime runs into a local CEMI workspace:

```yaml
cemi:
  enabled: true
  project: rpi-foot-traffic-monitor
  log_dir: .cemi-runtime
  run_name: storefront-live-monitor
  emit_interval_ms: 1000
```

The live integration logs:

- configuration parameters for the camera, detector, tracker, and counting line
- preview FPS and detector inference time
- rolling detector latency summaries including recent `p50` and `p95`
- active tracks, detections per pass, and total counted events
- running directional counts in summary metrics
- final CSV and annotated video as artifacts when enabled

To open the local workspace for real runs:

```bash
./.venv/bin/cemi view --save-dir .cemi-runtime
```

You can also launch the monitor under CEMI:

```bash
./.venv/bin/cemi start -- python3 main.py --config config.yaml
```

## Raspberry Pi notes

- Start with `640x480`, camera `fps: 30`, `max_results: 5`, `num_threads: 4`, and `detect_interval_ms: 150` on a Pi 4B.
- Camera capture now runs in a background thread and preview rendering is decoupled from detection, which reduces visible lag in the feed.
- Use `detect_interval_ms: 100-200` to tune how often boxes refresh without forcing the preview loop to wait for every inference.
- Start with `display: true` while tuning the line position and camera angle.
- When running headless over SSH, leave `display: true` if you want; the app will automatically disable preview when no GUI display is available.
- If frame rate is too low on a Pi 4B, lower camera resolution or FPS first.
- Restricting detection to `person` and `car` and lowering `max_results` reduces CPU load.
- Move to a Coral USB accelerator only if the unaccelerated TFLite pipeline is too slow for your scene.

## Tests

```bash
python3 -m unittest discover -s tests
```

## Known assumptions

- The detector is expected to use a COCO-style label file and a common TFLite detection output signature.
- Picamera2 must already be installed and working on the Pi if you use `camera.source: picamera2`.
# rpi-foot-traffic-monitor
# rpi-foot-traffic-monitor
