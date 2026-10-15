# ANVESHAK UGV

This package runs terrain segmentation, local A* path planning, ESP32 camera input, motor control, and the web dashboard. Visual odometry is no longer calculated inside the main perception pipeline or rendered inside the web dashboard.

The visual odometry application supplied separately is included unchanged in purpose under `standalone_visual_odometry/`. Pressing **Open Visual Odometry** on the web dashboard starts that application as a separate Python process and opens its native OpenCV window on the computer running `run.py`.

## Install

Use Python 3.10 or newer in a virtual environment:

```bash
python -m pip install -r requirements.txt
```

The package uses `opencv-python`, not `opencv-python-headless`, because the standalone window requires OpenCV GUI support.

## Run

From the extracted `anveshak-ugv` directory:

```bash
python run.py
```

Open the URL printed in the terminal, normally:

```text
http://localhost:8080
```

The default camera source is:

```text
http://192.168.4.1:81/stream
```

Use another camera or video with:

```bash
python run.py --source example.mp4
python run.py --source 0
python run.py --source http://192.168.4.1:81/stream
```

Use `--dry-run` to prevent physical motor requests during testing.

## Dashboard views

- **All views** shows camera, segmentation, and A* together.
- **Only Segmentation** shows the segmentation overlay.
- **Only A*** shows the traversability mask and planned route.
- **Open Visual Odometry** launches the separate OpenCV application.
- **Upload test video** switches the main pipeline to the uploaded video and disables motors.
- **Return to camera** restores the source passed through `--source` and keeps motors disabled for safety.

For a local or uploaded video, the standalone application opens that video independently from frame zero. For a live source, it reads the dashboard's raw-frame mirror, so the ESP32 stream is opened only once. Only one standalone window is launched at a time. Close it with `Q` or `Esc`; pressing the button again after it closes starts a new one.

The OpenCV window appears on the host computer, not on another phone or computer that merely opened the web page.

## Main options

```text
--source SOURCE
--device auto|cpu|cuda
--weights PATH
--calibration PATH
--width 640
--height 480
--seg-every 4
--seg-scale 1.0
--seg-confidence 0.65
--floor-fraction 0.45
--seg-tta
--fp16
--threads N
--clearance 16
--perception-timeout 2.5
--show-local-window
--no-dashboard
--host 0.0.0.0
--port 8080
--dry-run
--max-frames N
--metrics-json PATH
```

## Standalone visual odometry

The dashboard starts it automatically. It can also be run directly:

```bash
python standalone_visual_odometry/main.py --video example.mp4
```

It supports local video files, webcam indices, RTSP sources, MJPEG HTTP streams, and KITTI sequences. Its `run(video_path)` function can also be called from another Python module.

## Safety

The dashboard has no authentication. Use it only on a trusted local network. Test with `--dry-run` before enabling motors. The floor override is a manual rule and can incorrectly mark obstacles as traversable; inspect the mask before driving.


## Human Detection

The combined perception pipeline includes YOLO person detection via `semantic segmentation/human_detection.py`. It detects COCO `person` class objects, annotates the raw/segmentation/A* views, and publishes `human_detected`, `human_count`, and `human_max_confidence` telemetry to the dashboard.

Install `ultralytics` from `requirements.txt`. If `yolov8n.pt` is not present beside `combined_run.py`, Ultralytics will use/download the configured model name on first run. Configure detection with `--human-model`, `--human-confidence`, `--human-imgsz`, and `--human-every`.


## Geographic Local Map

The **Local Map** dashboard now supports geographic anchoring:

1. Open **Local Map**.
2. Use the OpenStreetMap view to search/pan/zoom to the operating area.
3. Click **Select area**, then click two opposite corners.
4. Enable **Satellite imagery** to switch the background to ArcGIS World Imagery.
5. Set **Local-frame heading offset** if the VO +X axis is not aligned with the map.
6. Click **Apply overlay**.

The live metric local map is then projected onto the selected real-world area. The overlay includes the VO trail, planned waypoints/route, current rover position, and confirmed survivor detections.

VO does not provide absolute latitude/longitude or compass heading by itself. Therefore, the selected rectangle supplies the geographic origin/scale and the heading offset supplies the orientation. A compass/GNSS heading can later be used to automate that offset.

The OSM base layer and attribution follow Leaflet/OpenStreetMap usage, while the satellite layer uses Esri World Imagery tiles. Internet access is required in the browser for these basemap tiles.


## Local Map v2 fixes

- Fixed Leaflet stacking so the map cannot cover the fixed ANVESHAK header/sidebar.
- Added explicit OpenStreetMap / Satellite basemap buttons.
- Selecting two corners now automatically anchors the area and switches to satellite imagery.
- Exposes the actual standalone visual-odometry trajectory to the browser as `vo_trail`.
- Cyan line = standalone VO trajectory; green dashed line = LocalNav trajectory; magenta dashed line = A* route; green marker = rover.
- Map is invalidated after page activation to prevent blank/incorrect tile layout.

## v3 VO + satellite reliability fixes

- Uses the documented `services.arcgisonline.com` World Imagery tile service.
- Shows a satellite tile error state if the browser cannot reach the imagery service.
- Adds live VO diagnostics to the Local Map.
- Prevents one bad VO frame from killing the VO worker.
- Handles stacked essential matrices returned by RANSAC.
- Keeps the standalone VO point ordering consistent with previous -> current motion.
