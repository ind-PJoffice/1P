# Master README — Complete Project Guide

> **Version**: 1.1.0  
> **Hardware**: NVIDIA Jetson Orin Nano  
> **Purpose**: AI-powered video analytics for retail pharmacy stores

---

## Table of Contents

- [1. High-Level Overview](#1-high-level-overview)
- [2. Explain the Code](#2-explain-the-code)
- [3. Relationship Between Files](#3-relationship-between-files)
- [4. Running the Project](#4-running-the-project)
- [FAQ](#faq)
- [Appendix](#appendix)

---

# 1. High-Level Overview

## What Problem Does This Project Solve?

This project turns a security camera into a smart analytics system for pharmacy retail stores. It runs on an NVIDIA Jetson Orin Nano — a small, GPU-equipped computer — and does the following:

1. **Counts people** entering and leaving the store in real-time.
2. **Detects exchange events** (hand-to-hand interactions between staff and customers).
3. **Records video continuously** in 1-minute segments for later review.
4. **Streams live video** to any web browser over WebRTC (a protocol that lets browsers receive video directly from a camera, with no third-party server).
5. **Analyzes demographics** (age and gender estimation from faces).
6. **Reports health data** (CPU, GPU, RAM, temperature) about the Jetson device.
7. **Sends alerts** to Telegram when events happen.
8. **Self-updates** from GitHub using over-the-air (OTA) updates.

> **OTA (Over-The-Air)**: A way to push software updates to a device remotely, without physically touching it.

## What Is the Expected Input?

- One or more **RTSP camera streams** — RTSP (Real-Time Streaming Protocol) is the standard way IP cameras send video over a network.
- A **JSON configuration file** that tells the system the camera URL, store name, detection settings, API endpoints, and more.

## What Is the Expected Output?

| Output | Where It Goes |
|---|---|
| Live video stream | Browser via WebRTC (port 8000) |
| Person count updates | RabbitMQ → local database → cloud API |
| Exchange events | RabbitMQ → local database → Telegram |
| 1-minute video segments | `upload_dir/` → local archive → remote SSD |
| Demographics data | Local database → cloud API |
| Device health metrics | Local database → cloud API |
| Log messages | Local database → Azure Blob Storage |
| Telegram alerts | Telegram bot API |

## Major Components

```
┌─────────────────────────────────────────────────────────┐
│                   RTSP Camera Feed                      │
└──────────┬──────────────────┬───────────────┬───────────┘
           │                  │               │
           ▼                  ▼               ▼
   ┌───────────────┐  ┌──────────────┐ ┌─────────────────┐
   │   ffmpeg.py   │  │ continuous   │ │   exchange      │
   │  (Detection   │  │ _recorder.py │ │ _detection/     │
   │  + WebRTC)    │  │ (1-min clips)│ │  producer.py    │
   └───────┬───────┘  └──────┬───────┘ └────────┬────────┘
           │                 │                  │
           ▼                 ▼                  ▼
   ┌───────────────┐  ┌──────────────┐  ┌───────────────┐
   │  RabbitMQ     │  │  upload_dir/ │  │   RabbitMQ    │
   │  Queues       │  │  (staging)   │  │   Queue       │
   └───────┬───────┘  └──────┬───────┘  └───────┬───────┘
           │                 │                  │
           ▼                 ▼                  ▼
   ┌───────────────┐  ┌──────────────┐  ┌───────────────┐
   │  footfall/    │  │ transfer_to  │  │  exchange/    │
   │  consumer.py  │  │ _local.py    │  │  consumer.py  │
   └───────┬───────┘  └──────┬───────┘  └───────┬───────┘
           │                 │                  │
           ▼                 ▼                  ▼
   ┌──────────────────────────────────────────────────────┐
   │       backend-feature/ (FastAPI on port 9000)        │
   │              PostgreSQL Database                     │
   └──────────────────────┬───────────────────────────────┘
                          │
                          ▼
                ┌──────────────────┐
                │  upload_to_db.py │
                │  (Cloud Sync)    │
                └──────────────────┘
```

## Technologies Used

| Technology | Purpose |
|---|---|
| **Python 3.10+** | Primary language |
| **YOLO (Ultralytics)** | Object detection (people) and pose estimation (exchange) |
| **PyTorch** | Deep learning framework (age/gender models) |
| **OpenCV** | Image processing, video reading, ROI masking |
| **FFmpeg** | Video decoding (RTSP → raw frames) and segment recording |
| **aiortc + aiohttp** | WebRTC signaling server for browser live streaming |
| **RabbitMQ + pika** | Message queue between detection producers and consumers |
| **FastAPI + SQLAlchemy** | Local REST API and PostgreSQL database |
| **PostgreSQL** | Persistent storage for all analytics data |
| **SQLite** | Fallback queue when the API is unreachable |
| **systemd** | Linux service manager — keeps processes alive across reboots |
| **Telegram Bot API** | Sends alerts and reports to operators |
| **GitHub API** | OTA (over-the-air) updates |

> **RabbitMQ**: A message broker — think of it as a post office for data. One process puts a message in a queue, and another process takes it out and processes it. This lets producers and consumers work at different speeds without losing data.

> **systemd**: The Linux startup system. It starts services automatically when the device boots and restarts them if they crash.

## Execution Flow from Start to Finish

When the Jetson powers on:

```mermaid
flowchart TD
    A[Jetson boots] --> B[systemd starts all .service files]
    B --> C1[backend.service → FastAPI on :9000]
    B --> C2[ffmpeg.service → RTSP capture + YOLO + WebRTC on :8000]
    B --> C3[continuous_recorder.service → 1-min segment recording]
    B --> C4[footfall_count.service → RabbitMQ consumer]
    B --> C5[exchange.service → exchange RabbitMQ consumer]
    B --> C6[jetson_resources.service → resource monitoring]
    B --> C7[upload_to_db.service → cloud sync]
    B --> C8[config.service → config polling every 15 min]
    B --> C9[ota-updater.service → checks GitHub for updates]
    
    C2 -->|person count messages| D[RabbitMQ]
    C2 -->|exchange messages| D
    D --> C4
    D --> C5
    C4 -->|POST to API| C1
    C5 -->|POST to API| C1
    C3 -->|metadata| C1
    C6 -->|resource data| C1
    C7 -->|reads unsent data| C1
    C7 -->|pushes to cloud| E[Cloud API]
```

## Application Lifecycle

1. **Boot**: systemd starts all services.
2. **Config fetch**: `configuration.py` loads the JSON config (or fetches it from the cloud API).
3. **Active hours check**: Many services only run between configured hours (default 8 AM–10 PM). Outside this window they sleep.
4. **Detection loop**: `ffmpeg.py` captures camera frames, runs YOLO person detection, publishes counts to RabbitMQ, and serves live WebRTC video.
5. **Recording**: `continuous_recorder.py` writes 1-minute MP4 segments independently.
6. **Consuming**: `footfall_count/consumer.py` and `exchange_detection/consumer.py` read RabbitMQ queues, compute statistics, send Telegram alerts, and store data in the local database.
7. **Syncing**: `upload_to_db.py` periodically reads unsent rows from the local database and pushes them to the cloud.
8. **File transfer**: `transfer_to_local.py` moves segments from `upload_dir/` to structured local archive. `transfer_to_ssd.py` offloads old files to a remote SSD when disk is low.
9. **Self-update**: `ota_updater.py` polls GitHub for version changes, pulls new code, and reboots.

---

# 2. Explain the Code

This section walks through every important source file. For each file we explain why it exists, when it runs, its key functions, and common pitfalls.

---

## 2.1 `ffmpeg.py` — The Central Live Camera Service

**Why it exists**: This is the single most important file. It is the "brain" that connects the camera feed to everything else — live streaming, person detection, and exchange detection.

**When it runs**: Started by `ffmpeg.service` (or `webrtc_streaming.service`) at boot. Runs as a long-lived daemon.

**What it does (four jobs in one)**:

1. **RTSP Capture**: Opens the camera stream once using FFmpeg in a subprocess, decodes frames to raw numpy arrays.
2. **YOLO Person Detection**: Runs YOLO inference on frames at a low rate (e.g., 2 fps) to save GPU. Publishes person-count messages to RabbitMQ.
3. **Exchange Detection**: Optionally runs YOLO Pose alongside person detection to detect hand-to-hand exchanges.
4. **WebRTC Streaming**: Hosts an embedded HTTP server (aiohttp on port 8000) that serves a viewer page and handles WebRTC signaling so any browser can watch live video.

### Key Classes

#### `RTSPCapture` (line 564)

```python
class RTSPCapture:
    def __init__(self, rtsp_url, buffer_size=1):
        # Spawns an FFmpeg subprocess that decodes RTSP to raw BGR frames
        # Frames are stored in a thread-safe deque (a double-ended queue)
```

**Purpose**: Wraps an FFmpeg subprocess that continuously decodes the RTSP stream into raw video frames. It runs in its own thread. Two consumers read from it:

- **YOLO detection** calls `capture.read()` — this **pops** a frame from the queue (consuming it).
- **WebRTC streaming** calls `capture.get_webrtc_frame()` — this reads the **latest** frame without removing it, so multiple viewers can watch without starving the detector.

> **deque (double-ended queue)**: A data structure that lets you add and remove items from both ends efficiently. Here it's used as a fixed-size buffer that automatically drops old frames when new ones arrive.

**Tricky logic**: The `_read_exact()` method uses `select()` with a 15-second timeout. If FFmpeg hangs (network drop, stalled camera), the capture loop detects it and restarts the subprocess automatically.

**Common beginner mistake**: Don't create multiple `RTSPCapture` instances for the same camera. The whole point of this design is that there is **one shared capture** that feeds both YOLO and WebRTC. Creating two would double the CPU usage.

#### `SharedFrameTrack` (line 799)

```python
class SharedFrameTrack(VideoStreamTrack):
    # Reads from RTSPCapture's non-consuming buffer for WebRTC
    async def recv(self):
        frame = self._capture.get_webrtc_frame()
        # Falls back to a black frame if no data for 5 seconds
```

**Purpose**: A WebRTC video track that reads from the shared camera buffer. This eliminates a second FFmpeg decode process that was causing 400%+ CPU spikes in older versions.

#### `SharedAudioTrack` (line 879)

Captures audio from a USB microphone via FFmpeg + PulseAudio and sends it over WebRTC. Uses async I/O to avoid blocking the event loop.

#### `RecordingManager` (line 739)

```python
class RecordingManager:
    def start(self, output_path, mic_device=None):
        # Does NOT spawn a new FFmpeg process!
        # Just marks a time range and posts a "bookmark" to the API
```

**Purpose**: In older versions, the system spawned a separate FFmpeg process to record event clips. This caused massive CPU spikes. The current `RecordingManager` is lightweight — it only records timestamps and posts an "event bookmark" to the backend API, relying on `continuous_recorder.py` for actual video storage.

### Key Functions

| Function | Purpose |
|---|---|
| `consume_people_frames()` | Main detection loop — reads frames, runs YOLO, publishes to RabbitMQ |
| `start_person_count()` | Starts the detection thread |
| `stop_person_count()` | Stops the detection thread (but does NOT release the camera) |
| `handle_offer()` | WebRTC signaling — receives SDP offer, creates answer |
| `handle_video_get()` | Serves recorded video files with HTTP Range support |
| `start_webrtc_server()` | Launches the aiohttp server on a background thread |

### Important Variables

| Variable | Purpose |
|---|---|
| `RELAY_CAPTURE_FPS` | The FPS at which FFmpeg decodes the stream — set to `max(CAPTURE_FPS, WEBRTC_STREAM_FPS)` |
| `INFERENCE_SKIP_FRAMES` | How many captured frames to skip between YOLO inferences — saves GPU |
| `capture_instance` | Global shared `RTSPCapture` — started once, never destroyed during runtime |
| `PCS` | Set of active WebRTC peer connections |

### Main Entry Point (line 1150)

```python
if __name__ == '__main__':
    capture_instance = RTSPCapture(rtsp)  # Start shared capture 24/7
    _init_audio()                          # Detect USB mic
    start_webrtc_server()                  # WebRTC runs 24/7
    
    while True:
        if not within_active_hours():
            stop_person_count()
            wait_until_start()  # Sleep until 8 AM
            continue
        
        # Load YOLO model, start detection, monitor threads
        start_person_count()
        while within_active_hours():
            time.sleep(5)
            # Restart dead threads
```

**Key point**: WebRTC runs 24/7 (so you can always watch the camera). Person detection only runs during active hours (default 8 AM–10 PM).

---

## 2.2 `continuous_recorder.py` — Segment Video Recording

**Why it exists**: Records the camera feed as 1-minute MP4 segments, independently from the detection loop. This ensures footage is always saved, even if detection crashes.

**When it runs**: Started by `continuous_recorder.service`. Runs during active hours.

### How It Works

```
RTSP Camera → FFmpeg segment muxer → record_dir/ → process_completed_segments() → upload_dir/
                                                              ↓
                                                     metadata → Backend API
                                                              ↓ (on failure)
                                                     local_footage_queue.db (SQLite retry queue)
```

1. `start_ffmpeg_segment_recording()` spawns an FFmpeg process using the `-f segment` muxer that automatically splits the video into 1-minute files named like `20260708_121500.mp4`.
2. Every 5 seconds, `process_completed_segments()` checks `record_dir/` for segments older than 65 seconds (meaning FFmpeg has moved on to the next file).
3. Completed segments are moved to `upload_dir/`, and metadata (filename, start time, end time) is POSTed to the backend API.
4. If the API call fails, the metadata is saved to a local SQLite queue (`local_footage_queue.db`) and retried every 5 minutes.

### Key Functions

| Function | Purpose |
|---|---|
| `start_ffmpeg_segment_recording()` | Spawns FFmpeg with segment muxer, optional audio |
| `stop_ffmpeg_segment_recording()` | Sends `q` to FFmpeg for graceful shutdown |
| `process_completed_segments()` | Moves finished segments, posts metadata |
| `check_ffmpeg_health()` | Detects crashes and stalls, disables audio if mic is problematic |
| `_make_room()` | Deletes oldest clips when disk is full |

### Tricky Logic — Audio Crash Recovery

If FFmpeg crashes 3 times within 60 seconds, the recorder assumes the USB microphone is the cause. It disables audio and retries. After 30 minutes, it tries re-enabling audio automatically.

```python
if _crash_count >= 3 and not DISABLE_AUDIO:
    _disable_audio_with_timer()  # Sets a 30-minute cooldown
```

**Common mistake**: Don't run `continuous_recorder.py` and an old `ffmpeg.py` event-recording at the same time. They would fight over the RTSP stream.

---

## 2.3 `configuration.py` — Configuration Management

**Why it exists**: Centralizes how the system loads its settings. Every other file imports `load_or_create_config()` from here.

**When it runs**: Imported by almost every file. Also runs standalone as `config.service` (polls every 15 minutes for config changes from the cloud).

### How It Works

1. On import, `load_or_create_config()` looks for `*_config.json` files in the project directory.
2. If one exists, it loads it. If not, it fetches the device serial number, calls the cloud API to get the config, and saves it as `{store_name}-{device_name}_config.json`.
3. When running standalone (`__main__`), it polls the cloud API every 15 minutes, compares the response with the saved file, and if the config changed, saves it and **restarts all services**.

### Key Functions

| Function | Purpose |
|---|---|
| `load_or_create_config()` | Load or fetch config — called by every file |
| `get_device_serial()` | Reads Jetson serial from `/sys/firmware/devicetree/` |
| `fetch_and_save_config()` | Fetches from API, compares, saves if changed |
| `restart_system_services()` | Restarts backend, footfall, jetson_resources, upload_to_db |
| `connect_to_wifi()` | Runs `update_wifi.sh` to connect to configured WiFi |

### Important Config Fields

| Field | Type | Purpose |
|---|---|---|
| `rtsp_url` | string | Camera RTSP stream URL |
| `store.name` | string | Store identifier (used in all API calls) |
| `person_detection.fps` | int | Target detection FPS |
| `person_detection.ROI` | object | Region of interest polygon (normalized 0–1 coordinates) |
| `person_detection.inference_resolution` | array | Resolution for YOLO input `[width, height]` |
| `exchange_detection.enabled` | bool | Turn exchange detection on/off |
| `exchange_detection.polygon_zone` | array | Polygon separating staff zone from customer zone |
| `webrtc.port` | int | Port for WebRTC server (default 8000) |
| `service_start_time` / `service_stop_time` | int | Active hours (default 8–22) |
| `DATABASE_URL` | string | PostgreSQL connection string |
| `BASE_URL` | string | Local backend API URL |
| `github` | object | OTA update GitHub repo info |

---

## 2.4 `database_apis/api_calls.py` — Shared HTTP Client Layer

**Why it exists**: Every part of the system needs to POST data to the local backend API. This file centralizes all those HTTP calls so they're consistent and easy to maintain.

**When it runs**: Imported by `ffmpeg.py`, `continuous_recorder.py`, `footfall_count/consumer.py`, `exchange_detection/consumer.py`, `upload_to_db.py`, `jetson_resources.py`, and `transfer_to_local.py`.

### Key Functions

| Function | Endpoint | Called By |
|---|---|---|
| `call_demographics_apis()` | `POST /demographics/` | Demographics detection |
| `call_footfall_apis()` | `POST /footfall/` | Footfall consumer (hourly) |
| `call_realtime_counts_api()` | `POST /realtime_count/` | Footfall consumer (real-time) |
| `call_exchange_api()` | `POST /exchange/` | Exchange consumer |
| `call_jetson_resources_api()` | `POST /jetson_resources/` | Jetson resource monitor |
| `call_continuous_footage_api()` | `POST /continuous_footage/` | Continuous recorder |
| `flush_pending_footage()` | retries pending rows | Continuous recorder startup |
| `update_continuous_footage_path()` | `PATCH /continuous_footage/{filename}/` | transfer_to_local |
| `get_unsent_data()` | `GET /{type}/?status=false` | upload_to_db |
| `update_status()` | `PATCH /{type}/` | upload_to_db |

### The Retry Queue Pattern

The most important function here is `call_continuous_footage_api()`. If the backend is unreachable:

```python
except Exception as e:
    # Network error → save to local SQLite queue for retry
    from database_apis.local_footage_store import save as _local_save
    _local_save(footage_data)
```

Later, `flush_pending_footage()` reads all queued rows and retries them. On a 4xx error (server rejected it), the row is deleted permanently. On a 5xx error or network error, the row stays in the queue.

---

## 2.5 `database_apis/local_footage_store.py` — SQLite Retry Queue

**Why it exists**: When the backend API is down, footage metadata would be lost forever. This file provides a local SQLite database (`local_footage_queue.db`) that buffers failed API calls until the server comes back.

### Key Functions

| Function | Purpose |
|---|---|
| `save(footage_data)` | Insert a row into `pending_footage` table |
| `get_all()` | Read all pending rows (oldest first) |
| `delete(row_id)` | Remove a successfully synced row |
| `increment_attempts(row_id)` | Track retry count |
| `count()` | Return number of pending rows |

> The video file itself is always safe in `upload_dir/` — this queue only protects the **metadata record** (filename, timestamps, etc.) that lets the dashboard find and index the video.

---

## 2.6 `database_apis/database_logger.py` — Custom Log Handler

**Why it exists**: Sends Python log messages to the backend API so they can be viewed remotely. It's a custom `logging.Handler` that intercepts log calls and POSTs them.

```python
class DatabaseLogHandler(Handler):
    def emit(self, record):
        # Formats the log, deduplicates it, then calls call_logs_api()
```

**Side effect**: Every `logger.info()` / `logger.error()` call in the codebase generates an HTTP POST to `POST /logs/`. This can be a lot of traffic if the system is chatty. The handler uses a `recent_logs` set to prevent duplicates.

---

## 2.7 `footfall_count/producer.py` — Person Detection Producer

**Why it exists**: The actual YOLO inference function for person detection. It's called by `ffmpeg.py`'s detection loop.

> **Note**: Despite being in `footfall_count/`, this file's `person_detection()` function is **imported and called directly** by `ffmpeg.py`. It does not run as its own process.

### Key Function: `person_detection()`

```python
def person_detection(model, frame, channel):
    # 1. Apply ROI polygon mask to frame
    # 2. Optionally resize to inference_resolution
    # 3. Run YOLO with classes=[0] (person only), conf=0.6
    # 4. Count detected people
    # 5. If people found: encode frame as JPEG+base64 (for demographics)
    # 6. Publish message to RabbitMQ "person_detection_queue"
    # 7. Return True if people were found
```

**Optimization**: Frame encoding (JPEG → base64) only happens when `person_count > 0`. This saves ~20% CPU by skipping encoding on empty frames.

---

## 2.8 `footfall_count/consumer.py` — Person Count Consumer

**Why it exists**: Listens to the RabbitMQ `person_detection_queue`, computes stable person counts using the statistical mode, sends Telegram alerts, and stores data in the database.

**When it runs**: Started by `footfall_count.service` at boot.

### How It Works

1. Collects 10 messages in a batch.
2. Computes the **mode** (most common value) of the person counts — this eliminates detection jitter.
3. If the mode changed from the last sent value, sends a Telegram notification and stores the count in the database.
4. If nobody has been seen for `no_person_timeout` seconds (default 30), publishes a "stop person count" message.

### `PersistentCounter` Class

Saves hourly footfall counts to `counter_state.json` so they survive restarts. Provides `get_hourly_count()` and `get_daily_count()` for reporting.

### Background Thread: `perform_analytics()`

Runs every hour, generates hourly footfall reports, sends them to Telegram, and posts them to the database.

---

## 2.9 `exchange_detection/producer.py` — Exchange Detection Producer

**Why it exists**: Detects physical exchanges (hand-to-hand interactions like handing over a medicine bag) between staff and customers using pose estimation.

**When it runs**: Two modes:
1. **Embedded**: Called directly by `ffmpeg.py`'s detection loop when `exchange_detection.enabled` is true.
2. **Standalone**: Can run independently with `python exchange_detection/producer.py`.

### Detection Logic

```
Frame → YOLO Pose → Keypoints (17 body points per person) → Zone Classification → Distance Measurement
```

1. **Zone classification**: A polygon zone is defined in the config. People inside the polygon are "staff", people outside are "customers". Uses OpenCV's `pointPolygonTest()`.
2. **Wrist distance**: Measures distances between all wrist pairs (left↔left, left↔right, right↔left, right↔right) of people in different zones.
3. **Dynamic threshold**: Instead of a fixed distance, uses a trained linear equation (`threshold = m * Y + c`) that accounts for perspective — objects closer to the camera appear larger.
4. **Tracking window**: Instead of triggering on a single frame, tracks the interaction over 1.25 seconds and captures the frame with the **minimum wrist distance**. This makes detections much more stable.

### `ExchangeTracker` Class

```python
class ExchangeTracker:
    def start_tracking(self, distance, frame, ...):
        # Begin tracking session — record first distance
    def update(self, distance, frame, ...):
        # If new distance < recorded minimum, update best frame
    def should_trigger(self):
        # Return True when tracking window has elapsed
    def get_result(self):
        # Return the best frame/distance from this session
```

---

## 2.10 `exchange_detection/consumer.py` — Exchange Event Consumer

**Why it exists**: Reads exchange events from RabbitMQ, sends Telegram alerts, stores events in the database, and generates hourly and daily reports.

**When it runs**: Started by `exchange.service`.

### Background Threads

- **`perform_analytics()`**: Sends hourly exchange count reports to Telegram.
- **`perform_daily_report()`**: At midnight, generates a daily Excel report of exchanges.

---

## 2.11 `exchange_detection/threshold_training/` — Threshold Training Pipeline

**Why it exists**: The wrist distance threshold for exchange detection depends on camera perspective. Objects far from the camera appear smaller, so the threshold should vary with the Y-position in the frame.

### Three-Step Training Process

| File | Purpose |
|---|---|
| `step1_labeling_tool.py` | GUI tool to label real exchanges with distance measurements |
| `step2_train_equation.py` | Fits a linear equation to labeled data: `threshold = m * Y + c` |
| `step3_test_threshold.py` | Validates the trained threshold on test video |

### `dynamic_threshold.py`

Loads the trained equation from `threshold_equation.json` and provides:

```python
threshold = DynamicThreshold(fallback_threshold=60)
t = threshold.get_threshold(y_position)  # Returns distance threshold for this Y
```

If no trained equation exists, it falls back to the static threshold from config.

---

## 2.12 `continuous_recorder.py` (Revisited — see section 2.2)

Already covered. See section 2.2.

---

## 2.13 `upload_to_db.py` — Cloud Synchronization

**Why it exists**: The local database on the Jetson is not accessible to the cloud dashboard. This script periodically reads unsent data and pushes it to the cloud PostgreSQL server.

**When it runs**: Started by `upload_to_db.service`. Runs during active hours with a configurable interval (default 300 seconds).

### What It Syncs

1. **Footfall data**: Reads `status=false` rows from `realtime_perosn_count` table, converts to cloud format, POSTs to `{postgres_url}/customer-footfall/`.
2. **Demographics data**: Same pattern, POSTs to `{postgres_url}/demographics/`.
3. **Continuous footage**: Reads unsynced segments (only those moved out of `upload_dir`), POSTs to `{postgres_url}/store-footage/bulk`.

### Data Format Conversion

The local database stores data in one format, but the cloud API expects a different format. Functions like `convert_footfall_for_postgres()` handle this translation, including adding the `admin_id` field required by the cloud.

---

## 2.14 `transfer_to_local.py` — Move Segments to Archive

**Why it exists**: Video segments in `upload_dir/` are in a flat directory. This script moves them into a structured local archive organized by date: `{local_base_path}/{admin_id}/event_videos/{YYYY-MM-DD}/{filename}.mp4`.

**When it runs**: Triggered by `transfer_to_local.timer` (systemd timer, runs periodically).

### Process

1. List all `.mp4` files in `upload_dir/`.
2. Group by date (extracted from filename).
3. Validate each video with OpenCV — delete corrupted files (0 frames), keep short clips (< 30s).
4. Copy to local archive path.
5. Call cloud API to register the footage record.
6. If API succeeds: delete from `upload_dir/`, update the `continuous_footage` table's `file_path`.
7. Bulk-insert to `StoreFootage` API for Day View in the dashboard.

---

## 2.15 `transfer_to_ssd.py` — Offload to Remote SSD

**Why it exists**: The Jetson has limited storage. When disk usage exceeds 75%, this script transfers files from the local archive to a remote SSD via SSH/SCP.

**When it runs**: Triggered by `transfer_to_ssd.service` / `transfer_to_ssd.timer`.

### Process

1. Check disk usage. If below 75%, exit.
2. Process date folders oldest-first.
3. For each video: `scp` to remote SSD → call API to update `device_id` → delete local copy.
4. After each date folder, re-check disk usage. Stop if below threshold.

---

## 2.16 `jetson_resources.py` — Device Health Monitor

**Why it exists**: Reports Jetson hardware metrics (CPU, GPU, RAM, temperature, power, fan speed) to the database at regular intervals.

**When it runs**: Started by `jetson_resources.service`. Runs in a loop with configurable interval (default 40 seconds).

### How It Works

Uses `tegrastats` (NVIDIA's Jetson system monitor) to get detailed hardware metrics, and `psutil` for standard CPU/disk info. Parses the `tegrastats` output line-by-line with regex to extract RAM, CPU cores, GPU load, temperatures, and power consumption.

---

## 2.17 `ota_updater.py` — Over-The-Air Updates

**Why it exists**: Allows the development team to push code updates to deployed Jetson devices remotely via GitHub.

**When it runs**: Started by `ota-updater.service`. Polls every hour (configurable).

### Update Flow

```
1. Check GitHub version.txt vs local version.txt
2. If different → git clone (shallow, depth=1) to /tmp/ota_update_tmp/
3. Compare every file: if changed → copy to application folder
4. If requirements.txt changed → pip install
5. If system_requirements.txt changed → apt-get install
6. If .service files changed → run service_script.sh to reload systemd
7. Save new version → Reboot device
```

### Security

- The GitHub Personal Access Token (PAT) is stored **base64-encoded** in the config to avoid accidental exposure. It's decoded at runtime.
- Git clone commands log a censored URL (with `***` instead of the PAT).

---

## 2.18 `master_automation_final.py` — Deployment Diagnostics

**Why it exists**: A comprehensive diagnostic tool that checks every aspect of the deployment — network, camera, device status, database, services — and sends a formatted Telegram report with a test video.

**When it runs**: On demand (not a background service). Two modes:
- `--mode deployment`: Full check + Telegram report + test video.
- `--mode device_status`: Quick status check, JSON output only.

### What It Checks

| Check | Method |
|---|---|
| Network interfaces | `ip -4 addr show` |
| Camera ping | `ping -c 1` |
| RTSP stream test | FFmpeg 3-second capture |
| CPU / RAM / Disk | `psutil` |
| WiFi signal strength | `iwconfig` |
| Internet connectivity | Socket to `8.8.8.8:53` |
| PostgreSQL connection | `psycopg2.connect()` |
| Service status | `systemctl is-active` / `pgrep -f` |
| Tailscale VPN | Ping to `ssd_device_ip` |

---

## 2.19 `demographics_detection/` — Age and Gender Analysis

**Why it exists**: Analyzes detected faces to estimate age and gender.

### Files

| File | Purpose |
|---|---|
| `frame_processing.py` | Main processor — detects faces with YOLO, runs age/gender inference, stores results |
| `script.py` | Standalone demo — captures from RTSP and runs demographics |
| `models/age_model.pth` | Pre-trained VGG16 age regression model (~537 MB) |
| `models/gender_model.pth` | Pre-trained ResNet152 gender classifier (~233 MB) |
| `models/yolov11s-face.pt` | YOLO face detector (~19 MB) |

### Inference Pipeline

```
Frame → YOLO Face Detection → Face Crops → ResNet152 (gender) + VGG16 (age) → JSON result → API
```

**Important**: This module is a **separate analysis path**, not part of the main detection loop. It processes frames saved by the footfall consumer, not live camera frames.

---

## 2.20 `age_gender_test.py` — Age/Gender Inference Functions

**Why it exists**: Provides the actual model loading and inference functions used by `demographics_detection/frame_processing.py`.

### Key Functions

| Function | Purpose |
|---|---|
| `load_gender_model()` | Loads ResNet152 gender classifier |
| `load_age_model()` | Loads VGG16 age regressor |
| `detect_faces()` | Face detection using MTCNN |
| `detect_face_yolo()` | Face detection using YOLO (primary method) |
| `predict_gender_and_age()` | Runs age + gender inference on face crops |
| `analyze_image()` | End-to-end: detect faces → predict → annotate |

---

## 2.21 `backend-feature/` — Local FastAPI Backend

**Why it exists**: Provides a REST API that all other processes use to store and query data. It runs on port 9000 and uses a local PostgreSQL database.

### Files

| File | Purpose |
|---|---|
| `main.py` | FastAPI app setup, route registration, server startup |
| `database.py` | SQLAlchemy engine/session setup |
| `models.py` | SQLAlchemy ORM models (database table definitions) |
| `schemas.py` | Pydantic schemas (request/response validation) |
| `routes/*.py` | API endpoint definitions (16 route files) |
| `crud_operations/*.py` | Database CRUD operations (15 files) |
| `utils/*.py` | Helpers (column management, CSV export, token verification) |

### Database Tables

| Table | Purpose |
|---|---|
| `faces` | Demographics data (age, gender, bbox) |
| `footfall` | Hourly footfall reports |
| `realtime_perosn_count` | Real-time person count changes (note: typo in table name is intentional — it's baked into the database) |
| `exchange_events` | Exchange detection events |
| `continuous_footage` | Metadata for recorded video segments |
| `customer_store_footage` | Cloud-synced footage records |
| `logs` | Application log messages |
| `jetson_status` | Jetson active timestamps |
| `customer_active_duration` | Customer presence periods |
| `conversation` | Audio conversation data |
| `session_token` | Session tokens |

---

## 2.22 `service/` and `timer/` — systemd Unit Files

**Why it exists**: systemd `.service` files tell Linux how to start each process. `.timer` files schedule periodic tasks.

### Services

| Service File | What It Starts | User |
|---|---|---|
| `ffmpeg.service` | `ffmpeg.py` (detection + WebRTC) | jetson |
| `webrtc_streaming.service` | `ffmpeg.py` (identical — likely a duplicate) | jetson |
| `continuous_recorder.service` | `continuous_recorder.py` | jetson |
| `backend.service` | `backend-feature/main.py` (FastAPI) | jetson |
| `footfall_count.service` | `footfall_count/consumer.py` | jetson |
| `exchange.service` | `exchange_detection/consumer.py` | jetson |
| `jetson_resources.service` | `jetson_resources.py` | root |
| `upload_to_db.service` | `upload_to_db.py` | root |
| `transfer_to_local.service` | `transfer_to_local.py` | root |
| `transfer_to_ssd.service` | `transfer_to_ssd.py` | root |
| `config.service` | `configuration.py` (config polling) | root |
| `demographic.service` | demographics pipeline | root |

### Timers

| Timer File | Purpose |
|---|---|
| `demographic.timer` | Schedules demographics processing |
| `transfer_to_local.timer` | Schedules file transfer to local archive |
| `transfer_to_ssd.timer` | Schedules file transfer to remote SSD |

> **Note**: `ffmpeg.service` and `webrtc_streaming.service` are nearly identical — both start `ffmpeg.py`. Only one should be enabled. This appears to be a legacy duplication.

---

## 2.23 Shell Scripts

| Script | Purpose |
|---|---|
| `service_script.sh` | Copies `.service` and `.timer` files to `/etc/systemd/system/`, enables and starts them |
| `setup_ffmpeg_service.sh` | Installs the ffmpeg service as a user service |
| `setup_ota_service.sh` | Installs the OTA updater service |
| `update_wifi.sh` | Connects the Jetson to a WiFi network |

---

## 2.24 Other Files

| File | Purpose | Notes |
|---|---|---|
| `relu_fuction.py` | Contains only `print("hello")` | **Unused junk file — can be safely deleted** |
| `ffmpeg_old1.py` | Old version of ffmpeg.py | **Legacy backup — can be safely deleted** |
| `ffmpeg.py.bak-audio-2` | Another backup | **Legacy backup — can be safely deleted** |
| `test-benchmark.py` | Performance benchmarking tool | Development tool, not used in production |
| `check_now.jpg`, `current_view.jpg`, etc. | Sample/test images | Not used by code — can be deleted |
| `node-v18.20.5-linux-arm64.tar.xz` | Node.js binary archive | Not used by Python code — leftover from setup |
| `tegrastats_*.log` | Captured tegrastats output | Debug data — can be deleted |
| `local_footage_queue.db` | SQLite retry queue database | **Do not delete** — contains pending metadata |
| `yolo11n.pt` | YOLO11 Nano model weights | **Required** — used for person detection |
| `version.txt` | Current code version (e.g., `1.1.0`) | Used by OTA updater |
| `requirements.txt` | Python package dependencies | Used by OTA updater and manual setup |
| `system_requirements.txt` | System-level apt packages | Used by OTA updater |
| `requirements_backup_20251230.txt` | Old requirements backup | Can be deleted |

---

# 3. Relationship Between Files

## 3.1 Dependency Graph

```mermaid
graph TD
    CONFIG[configuration.py] --> |"load_or_create_config()"| FFMPEG[ffmpeg.py]
    CONFIG --> RECORDER[continuous_recorder.py]
    CONFIG --> UPLOAD[upload_to_db.py]
    CONFIG --> FOOTPROD[footfall_count/producer.py]
    CONFIG --> FOOTCONS[footfall_count/consumer.py]
    CONFIG --> EXCHPROD[exchange_detection/producer.py]
    CONFIG --> EXCHCONS[exchange_detection/consumer.py]
    CONFIG --> JETRES[jetson_resources.py]
    CONFIG --> OTA[ota_updater.py]
    CONFIG --> MASTER[master_automation_final.py]
    CONFIG --> DEMOG[demographics_detection/frame_processing.py]
    CONFIG --> BACKEND_DB[backend-feature/database.py]
    CONFIG --> LOGS_API[database_apis/logs_api.py]
    CONFIG --> API_CALLS[database_apis/api_calls.py]
    
    API_CALLS --> FFMPEG
    API_CALLS --> RECORDER
    API_CALLS --> UPLOAD
    API_CALLS --> FOOTCONS
    API_CALLS --> EXCHCONS
    API_CALLS --> JETRES
    API_CALLS --> DEMOG
    API_CALLS --> TRANSFER_LOCAL[transfer_to_local.py]

    DB_LOGGER[database_apis/database_logger.py] --> FFMPEG
    DB_LOGGER --> RECORDER
    DB_LOGGER --> FOOTPROD
    DB_LOGGER --> FOOTCONS
    DB_LOGGER --> EXCHPROD
    DB_LOGGER --> EXCHCONS
    DB_LOGGER --> JETRES
    DB_LOGGER --> DEMOG

    FOOTPROD --> |"person_detection()"| FFMPEG
    EXCHPROD --> |"detect_exchanges()"| FFMPEG
    
    LOCAL_STORE[database_apis/local_footage_store.py] --> API_CALLS
    
    DYN_THRESH[threshold_training/dynamic_threshold.py] --> EXCHPROD
    
    AGE_GENDER[age_gender_test.py] --> DEMOG
    
    BACKEND_DB --> BACKEND_MAIN[backend-feature/main.py]
    BACKEND_MODELS[backend-feature/models.py] --> BACKEND_MAIN
```

## 3.2 Execution Flow Step-by-Step

### Flow 1: Person Detection → Telegram Alert

```
1. ffmpeg.py reads a frame from RTSPCapture
2. ffmpeg.py calls footfall_count/producer.py → person_detection()
3. person_detection() runs YOLO on the frame
4. person_detection() publishes count to RabbitMQ "person_detection_queue"
5. footfall_count/consumer.py receives the message
6. consumer.py batches 10 messages, computes mode
7. If count changed: consumer.py calls database_apis/api_calls.py → call_realtime_counts_api()
8. api_calls.py POSTs to backend-feature/main.py → /realtime_count/
9. backend routes → crud_operations → PostgreSQL
10. consumer.py sends Telegram alert
```

### Flow 2: Exchange Detection → Database

```
1. ffmpeg.py reads a frame, downscales it
2. ffmpeg.py calls exchange_detection/producer.py → detect_exchanges()
3. detect_exchanges() runs YOLO Pose, finds keypoints
4. Classifies each person as inside/outside polygon zone
5. Measures wrist distances between inside↔outside pairs
6. Uses dynamic_threshold.py to get threshold for this Y position
7. If below threshold → ExchangeTracker starts/updates tracking
8. After tracking window → publishes event to RabbitMQ "exchange_detection_queue"
9. exchange_detection/consumer.py receives event
10. consumer.py → Telegram notification + database_apis/api_calls.py → call_exchange_api()
11. backend stores in exchange_events table
```

### Flow 3: Continuous Recording → Cloud Dashboard

```
1. continuous_recorder.py starts FFmpeg with segment muxer
2. FFmpeg writes 1-min segments to record_dir/
3. continuous_recorder.py detects completed segment (>65s old)
4. Moves segment to upload_dir/
5. POSTs metadata to backend API → continuous_footage table
    ↓ (if API fails)
    Saves to local_footage_queue.db via local_footage_store.py
    ↓ (periodically)
    flush_pending_footage() retries failed rows
6. transfer_to_local.py (timer) picks up files from upload_dir/
7. Validates with OpenCV, moves to local archive
8. POSTs to cloud API
9. Deletes from upload_dir/
10. Updates continuous_footage.file_path in database
11. If disk >75%: transfer_to_ssd.py SCPs to remote SSD
```

## 3.3 File Categories

### Entry Points (files that run as processes)

| File | How It Starts |
|---|---|
| `ffmpeg.py` | `ffmpeg.service` |
| `continuous_recorder.py` | `continuous_recorder.service` |
| `footfall_count/consumer.py` | `footfall_count.service` |
| `exchange_detection/consumer.py` | `exchange.service` |
| `backend-feature/main.py` | `backend.service` |
| `upload_to_db.py` | `upload_to_db.service` |
| `jetson_resources.py` | `jetson_resources.service` |
| `transfer_to_local.py` | `transfer_to_local.timer` |
| `transfer_to_ssd.py` | `transfer_to_ssd.timer` |
| `configuration.py` | `config.service` |
| `ota_updater.py` | `ota-updater.service` |
| `master_automation_final.py` | Manual / on-demand |

### Shared Libraries (imported but never run directly)

| File | Used By |
|---|---|
| `configuration.py` (as module) | Everything |
| `database_apis/api_calls.py` | Most processes |
| `database_apis/database_logger.py` | Most processes |
| `database_apis/local_footage_store.py` | `api_calls.py` only |
| `database_apis/logs_api.py` | `database_logger.py` only |
| `footfall_count/producer.py` | `ffmpeg.py` (imported, not standalone) |
| `exchange_detection/producer.py` | `ffmpeg.py` (imported for detect_exchanges) |
| `exchange_detection/threshold_training/dynamic_threshold.py` | `exchange_detection/producer.py` |
| `age_gender_test.py` | `demographics_detection/frame_processing.py` |
| `backend-feature/database.py` | `backend-feature/main.py` |
| `backend-feature/models.py` | Backend routes and CRUD files |
| `backend-feature/schemas.py` | Backend routes |

### Files Safe to Modify

| File | What You Can Change |
|---|---|
| `*_config.json` | Camera URLs, detection settings, API endpoints |
| `footfall_count/consumer.py` | Telegram message format, batch size, analytics logic |
| `exchange_detection/consumer.py` | Alert format, report generation |
| `jetson_resources.py` | Which metrics to collect, push interval |
| `master_automation_final.py` | Diagnostic checks, Telegram format |
| `upload_to_db.py` | Data conversion format, sync interval |
| Backend route files | API endpoint behavior |

### Files to Modify with Caution

| File | Risk |
|---|---|
| `ffmpeg.py` | Heart of the system — bugs crash live streaming + detection |
| `continuous_recorder.py` | Bugs cause video loss |
| `configuration.py` | Breaking this breaks everything |
| `database_apis/api_calls.py` | Breaking this breaks all data storage |

### Files to Rarely Modify

| File | Why |
|---|---|
| `backend-feature/models.py` | Changing table schemas can corrupt existing data |
| `backend-feature/database.py` | Core infrastructure |
| `ota_updater.py` | A bug here can brick remote devices |
| systemd `.service` files | Incorrect settings can prevent boot recovery |

---

# 4. Running the Project

## 4.1 Prerequisites

### Hardware

- **NVIDIA Jetson Orin Nano** (8GB RAM recommended)
- **IP Camera** with RTSP stream support
- **USB Microphone** (optional, for audio recording/streaming)
- **External SSD** (optional, for video archival over SSH)
- **Network connection** (Ethernet preferred, WiFi supported)

### Software

| Requirement | Version | Notes |
|---|---|---|
| **JetPack / L4T** | 5.x+ | NVIDIA's Linux for Jetson |
| **Python** | 3.10+ | Must support `match` statements and `str \| None` syntax |
| **PostgreSQL** | 14+ | Local database |
| **RabbitMQ** | 3.x | Message queue server |
| **FFmpeg** | 4.x+ | Video processing |
| **CUDA** | Included in JetPack | GPU acceleration for YOLO |
| **PulseAudio** | System default | For USB microphone access |

### Environment

- The project expects to be installed at `/home/jetson/AI-ML/` (configurable in service files).
- A Python virtual environment at `/home/jetson/AI-ML/venv/`.
- PostgreSQL running locally with a database named `fastapi`.

---

## 4.2 Installation

### Step 1: System Packages

```bash
sudo apt-get update
sudo apt-get install -y \
    postgresql postgresql-contrib \
    rabbitmq-server \
    ffmpeg \
    sshpass \
    libavformat-dev libavcodec-dev libavdevice-dev \
    libavutil-dev libswscale-dev libavresample-dev \
    libopus-dev libvpx-dev libssl-dev libffi-dev \
    pkg-config python3-dev
```

### Step 2: PostgreSQL Setup

```bash
# Start PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Create user and database
sudo -u postgres psql -c "CREATE USER pharmacy1 WITH PASSWORD '1p';"
sudo -u postgres psql -c "CREATE DATABASE fastapi OWNER pharmacy1;"
```

### Step 3: RabbitMQ

```bash
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
```

### Step 4: Clone Repository

```bash
cd /home/jetson
git clone <repository-url> AI-ML
cd AI-ML
```

### Step 5: Python Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

> **Note**: On Jetson, `torch` and `torchvision` must be installed from NVIDIA's wheels, not PyPI. See [NVIDIA's PyTorch installation guide for Jetson](https://forums.developer.nvidia.com/t/pytorch-for-jetson/).

### Step 6: Download AI Models

The models should be in these locations:

```
AI-ML/yolo11n.pt                                    # Person detection (~5.6 MB)
AI-ML/demographics_detection/models/age_model.pth    # Age estimation (~537 MB)
AI-ML/demographics_detection/models/gender_model.pth # Gender classification (~233 MB)
AI-ML/demographics_detection/models/yolov11s-face.pt # Face detection (~19 MB)
```

If using exchange detection, also need:

```
AI-ML/yolo11n-pose.pt    # Pose estimation model
```

### Step 7: Configuration

Create or fetch the config file. The system will try to fetch it from the cloud API using the device's serial number. Alternatively, create it manually:

```bash
cp UrMedz-Eldorado-rpi-1_config.json mystore-jetson_config.json
# Edit with your camera URL, store name, API endpoints, etc.
```

Minimum required fields:

```json
{
    "device_id": "your_device_serial",
    "device_name": "jetson",
    "store": { "name": "your-store-name", "admin_id": "your_admin_id" },
    "rtsp_url": "rtsp://user:pass@camera-ip:554/stream",
    "BASE_URL": "http://localhost:9000",
    "DATABASE_URL": "postgresql://pharmacy1:1p@localhost:5432/fastapi",
    "person_detection": {
        "fps": 5,
        "model": "yolo11n.pt",
        "ROI": { "p1": [1,0], "p2": [1,1], "p3": [0,1], "p4": [0,0] }
    }
}
```

### Step 8: Install systemd Services

```bash
sudo bash service_script.sh
```

This copies all `.service` and `.timer` files to `/etc/systemd/system/`, enables them, and starts them.

---

## 4.3 How to Run

### Run Everything (Production)

After installation, all services start automatically on boot. To manually start them:

```bash
# Start the backend first (other services depend on it)
sudo systemctl start backend.service

# Start detection and streaming
sudo systemctl start ffmpeg.service

# Start recording
sudo systemctl start continuous_recorder.service

# Start consumers
sudo systemctl start footfall_count.service
sudo systemctl start exchange.service

# Start monitoring
sudo systemctl start jetson_resources.service
sudo systemctl start upload_to_db.service
```

### Run Individual Components (Development)

```bash
# Activate virtual environment
source venv/bin/activate

# Run the backend
cd backend-feature && python main.py

# Run the main detection + streaming (in another terminal)
python ffmpeg.py

# Run the continuous recorder (in another terminal)
python continuous_recorder.py

# Run the footfall consumer (in another terminal)
python footfall_count/consumer.py
```

### Run Diagnostics

```bash
python master_automation_final.py --mode deployment
# Or for quick status:
python master_automation_final.py --mode device_status --no-video
```

---

## 4.4 Expected Output

### When Everything Works

```
$ sudo systemctl status ffmpeg.service
● ffmpeg.service - ffmpeg — RTSP capture, YOLO detection, WebRTC streaming
   Active: active (running)

$ sudo systemctl status backend.service
● backend.service - FastAPI Backend Service
   Active: active (running)
```

Opening `http://<jetson-ip>:8000` in a browser shows the live camera viewer.

Opening `http://<jetson-ip>:9000/docs` shows the FastAPI swagger documentation.

Telegram receives messages like:

```
🚀 People have entered MyStore!
🕒 2026-07-08 14:30:00
🌟 Current footfall : 2 people
📈 Total footfall (Today) : 15 people
```

### Checking Logs

```bash
# Live logs for ffmpeg service
sudo journalctl -u ffmpeg.service -f

# Live logs for backend
sudo journalctl -u backend.service -f

# Last 100 lines of footfall consumer
sudo journalctl -u footfall_count.service -n 100
```

---

## 4.5 Troubleshooting

### ❌ "ModuleNotFoundError: No module named 'ultralytics'"

```bash
source venv/bin/activate
pip install ultralytics
```

### ❌ "Failed to connect to RabbitMQ"

```bash
sudo systemctl start rabbitmq-server
sudo systemctl enable rabbitmq-server
```

### ❌ "DATABASE_URL config variable is not set!"

Your config JSON file is missing or doesn't have `DATABASE_URL`. Check that a `*_config.json` file exists in the project root.

### ❌ "Cannot open video stream" / "RTSPCapture: no data from FFmpeg"

- Check that the camera is reachable: `ping <camera-ip>`
- Test the RTSP URL directly: `ffmpeg -i "rtsp://..." -t 3 -y test.mp4`
- Check credentials in the RTSP URL.

### ❌ CUDA out of memory

The Jetson Orin Nano has limited GPU memory. Solutions:
- Reduce `inference_resolution` in config (e.g., `[320, 240]`).
- Disable exchange detection if not needed.
- Lower `capture_fps`.

### ❌ "Error posting footage bookmark, saving locally for retry"

The backend API is unreachable. The data is being saved to `local_footage_queue.db` and will be retried automatically. Check:

```bash
sudo systemctl status backend.service
```

### ❌ FFmpeg crashes repeatedly

Check the audio. If the USB microphone is causing crashes, the system will auto-disable audio after 3 rapid crashes. You can also manually set `"disable_audio": true` in the config.

### ❌ Disk full

Run the SSD transfer manually:

```bash
python transfer_to_ssd.py
```

Or increase `DISK_USAGE_THRESHOLD` in `transfer_to_ssd.py`.

### ❌ OTA updater not working

Check the GitHub PAT hasn't expired:

```bash
sudo journalctl -u ota-updater.service -n 50
```

Generate a new PAT, base64-encode it, and update the config.

---

# FAQ

## General

**Q: Which file starts the application?**  
A: There is no single entry point. The system is a collection of independent services. `ffmpeg.py` is the most important one (detection + streaming). `backend-feature/main.py` is the database API. Each runs as a separate process managed by systemd.

**Q: Where does the data come from?**  
A: From the RTSP camera stream. The config JSON file (`*_config.json`) tells the system which camera to connect to.

**Q: Where is the output saved?**  
A: Multiple places:
- **Video segments**: `record_dir/` → `upload_dir/` → `{local_base_path}/{admin_id}/event_videos/`
- **Analytics data**: PostgreSQL database (local, port 5432)
- **Alerts**: Telegram
- **Logs**: PostgreSQL `logs` table → Azure Blob Storage

**Q: How are errors handled?**  
A: Three levels:
1. **Process-level**: systemd restarts crashed services automatically (`Restart=always`).
2. **API-level**: Failed API calls to the backend are queued locally and retried.
3. **Hardware-level**: FFmpeg crashes are detected and the subprocess is restarted. Audio is auto-disabled if it causes repeated crashes.

**Q: How can I add a new feature?**  
A: Follow the existing pattern:
1. Create a new file or module for your feature.
2. Import `load_or_create_config()` from `configuration.py`.
3. Use `database_apis/api_calls.py` to talk to the backend.
4. If you need a new database table, add a model in `backend-feature/models.py`, a schema in `schemas.py`, CRUD operations, and routes.
5. Create a `.service` file and add it to `service/`.

## Specific Files

**Q: Can I delete `relu_fuction.py`?**  
A: Yes. It only contains `print("hello")` and is not imported anywhere.

**Q: Can I delete `ffmpeg_old1.py` and `ffmpeg.py.bak-audio-2`?**  
A: Yes. These are old backups. Only `ffmpeg.py` is used.

**Q: What happens if `person_detection()` in `footfall_count/producer.py` is removed?**  
A: Person detection stops working entirely. `ffmpeg.py` imports and calls this function directly.

**Q: Which file should I modify to change the detection sensitivity?**  
A: Edit the `*_config.json` file:
- `person_detection.fps`: How many frames per second to process
- `person_detection.ROI`: Which area of the frame to analyze
- `person_detection.inference_resolution`: Resolution for YOLO input

**Q: Why are `ffmpeg.service` and `webrtc_streaming.service` identical?**  
A: This is a legacy duplication. Both start `ffmpeg.py`. Only enable **one** of them.

---

# Appendix

## A. Project Directory Tree

```
AI-ML/
├── ffmpeg.py                          # Central detection + WebRTC service (1230 lines)
├── continuous_recorder.py             # 1-minute segment recorder (453 lines)
├── configuration.py                   # Config loading and management (217 lines)
├── upload_to_db.py                    # Cloud sync service (197 lines)
├── transfer_to_local.py               # Move segments to local archive (239 lines)
├── transfer_to_ssd.py                 # Offload to remote SSD (212 lines)
├── jetson_resources.py                # Device health monitor (280 lines)
├── ota_updater.py                     # Over-the-air updater (461 lines)
├── master_automation_final.py         # Deployment diagnostics (1204 lines)
├── age_gender_test.py                 # Age/gender inference functions (148 lines)
├── *_config.json                      # Device configuration (machine-specific)
├── version.txt                        # Current code version
├── requirements.txt                   # Python dependencies
├── system_requirements.txt            # System-level apt packages
├── yolo11n.pt                         # YOLO person detection model
├── local_footage_queue.db             # SQLite retry queue (auto-created)
│
├── database_apis/
│   ├── __init__.py
│   ├── api_calls.py                   # Shared HTTP client for all API calls
│   ├── database_logger.py             # Custom log handler → sends logs to API
│   ├── local_footage_store.py         # SQLite fallback queue for footage metadata
│   └── logs_api.py                    # POST /logs/ endpoint caller
│
├── footfall_count/
│   ├── producer.py                    # person_detection() function (imported by ffmpeg.py)
│   ├── consumer.py                    # RabbitMQ consumer + Telegram alerts
│   └── counter_state.json             # Persistent hourly counts
│
├── exchange_detection/
│   ├── producer.py                    # Exchange detection using YOLO Pose
│   ├── consumer.py                    # RabbitMQ consumer + Telegram alerts
│   ├── daily_report.py                # Daily Excel report generation
│   ├── set_polygon_zone.py            # GUI tool to define staff/customer zones
│   ├── set_dividing_line.py           # GUI tool for line-based zone (legacy)
│   ├── test_exchange.py               # Test exchange detection on video file
│   ├── snapshots/                     # Saved exchange event images
│   └── threshold_training/
│       ├── __init__.py
│       ├── dynamic_threshold.py       # Threshold calculator (equation-based)
│       ├── step1_labeling_tool.py     # GUI tool: label exchange distances
│       ├── step2_train_equation.py    # Train threshold equation from labels
│       └── step3_test_threshold.py    # Validate trained threshold
│
├── demographics_detection/
│   ├── frame_processing.py            # Demographics pipeline
│   ├── script.py                      # Standalone RTSP demographics demo
│   └── models/
│       ├── age_model.pth              # VGG16 age regression (~537 MB)
│       ├── gender_model.pth           # ResNet152 gender classifier (~233 MB)
│       └── yolov11s-face.pt           # YOLO face detector (~19 MB)
│
├── backend-feature/
│   ├── main.py                        # FastAPI app (port 9000)
│   ├── database.py                    # SQLAlchemy engine setup
│   ├── models.py                      # ORM table definitions
│   ├── schemas.py                     # Pydantic request/response schemas
│   ├── routes/                        # 16 route files (one per feature)
│   ├── crud_operations/               # 15 CRUD files (one per table)
│   └── utils/                         # Helpers (column mgmt, CSV, auth)
│
├── service/                           # systemd service unit files
│   ├── ffmpeg.service
│   ├── webrtc_streaming.service       # Duplicate of ffmpeg.service
│   ├── continuous_recorder.service
│   ├── backend.service
│   ├── footfall_count.service
│   ├── exchange.service
│   ├── jetson_resources.service
│   ├── upload_to_db.service
│   ├── transfer_to_local.service
│   ├── transfer_to_ssd.service
│   ├── config.service
│   └── demographic.service
│
├── timer/                             # systemd timer unit files
│   ├── demographic.timer
│   ├── transfer_to_local.timer
│   └── transfer_to_ssd.timer
│
├── logs/
│   ├── __init__.py
│   └── get_logs.py                    # Upload logs to Azure Blob Storage
│
├── service_script.sh                  # Install/restart systemd services
├── setup_ffmpeg_service.sh            # Install ffmpeg user service
├── setup_ota_service.sh               # Install OTA updater service
├── update_wifi.sh                     # WiFi connection script
│
├── record_dir/                        # FFmpeg writes segments here (temporary)
├── upload_dir/                        # Completed segments staging area
└── demo_output/                       # Demographics output images
```

## B. Full Execution Flow Diagram

```mermaid
flowchart TD
    subgraph BOOT["System Boot"]
        SYS[systemd] --> BE[backend.service<br>FastAPI :9000]
        SYS --> FF[ffmpeg.service<br>Detection+WebRTC :8000]
        SYS --> CR[continuous_recorder.service]
        SYS --> FC[footfall_count.service]
        SYS --> EX[exchange.service]
        SYS --> JR[jetson_resources.service]
        SYS --> UP[upload_to_db.service]
        SYS --> OT[ota-updater.service]
        SYS --> CF[config.service]
    end

    subgraph CAMERA["Camera"]
        CAM[RTSP Camera] --> |RTSP Stream| FF
        CAM --> |RTSP Stream| CR
    end

    subgraph DETECTION["Detection Pipeline"]
        FF --> |shared capture| YOLO[YOLO Person Detection]
        FF --> |shared capture| POSE[YOLO Pose Detection]
        FF --> |shared capture| WRTC[WebRTC Stream]
        YOLO --> |count message| RMQ[RabbitMQ]
        POSE --> |exchange event| RMQ
    end

    subgraph CONSUMING["Event Processing"]
        RMQ --> FC
        RMQ --> EX
        FC --> |store count| BE
        FC --> |alert| TG[Telegram]
        EX --> |store event| BE
        EX --> |alert| TG
    end

    subgraph RECORDING["Video Pipeline"]
        CR --> |1-min segments| RD[record_dir/]
        RD --> |move completed| UD[upload_dir/]
        UD --> |metadata| BE
        UD --> |move to archive| TL[transfer_to_local.py]
        TL --> LA[local archive]
        LA --> |if disk > 75%| TS[transfer_to_ssd.py]
        TS --> SSD[Remote SSD]
    end

    subgraph SYNC["Cloud Sync"]
        UP --> |read unsent| BE
        UP --> |push| CLOUD[Cloud API]
    end

    WRTC --> BROWSER[Web Browser]
    JR --> |hardware metrics| BE
```

## C. Suggested Reading Order for New Developers

1. **`configuration.py`** — Understand how settings are loaded
2. **`*_config.json`** — See what settings exist
3. **`ffmpeg.py`** — The central service (read the `__main__` block first, then classes)
4. **`footfall_count/producer.py`** — How detection works
5. **`footfall_count/consumer.py`** — How detection results are processed
6. **`continuous_recorder.py`** — How video is recorded
7. **`database_apis/api_calls.py`** — How data reaches the database
8. **`backend-feature/main.py`** — The API that stores everything
9. **`backend-feature/models.py`** — The database tables
10. **Service files in `service/`** — How everything starts automatically

## D. Suggestions for Future Improvements

1. **Remove duplicate service files**: `ffmpeg.service` and `webrtc_streaming.service` are identical. Keep one, delete the other.
2. **Delete unused files**: `relu_fuction.py`, `ffmpeg_old1.py`, `ffmpeg.py.bak-audio-2`, `node-v18.20.5-linux-arm64.tar.xz`, `requirements_backup_20251230.txt`.
3. **Fix the `utils/service_start_stop.py` import**: This module is imported by several files but is not in the repository. It must be present on the Jetson or the services will fail to start.
4. **Add the `utils/` directory to the repo**: It contains `service_start_stop.py` (active hours logic) but the directory is missing from the repository root.
5. **Fix the typo in `realtime_perosn_count` table name**: The table is misspelled (`perosn` instead of `person`). This requires a database migration.
6. **Add health check endpoints**: The backend has no health check route. Add `GET /health` for monitoring.
7. **Centralize duplicate code**: `get_usb_microphone_name()` is duplicated in `ffmpeg.py` and `continuous_recorder.py`. Extract to a shared utility.
8. **Add `logger` variable in `logs_api.py`**: Line 28–30 reference `logger` but it's never defined in that file. The `except` blocks will crash silently.
9. **Add automated tests**: There are no unit tests. The `test_exchange.py` file is a manual test tool, not an automated test suite.
10. **Use environment variables or `.env` for secrets**: The config JSON contains WiFi passwords, database credentials, Telegram tokens, and GitHub PATs in plaintext. Consider using environment variables or a secrets manager.
11. **Consider containerization**: Running all services natively makes deployment fragile. Docker containers would improve reproducibility.