# LightTrack

LightTrack is a dual-axis solar tracking system with automated dirt detection, developed as a bachelor's thesis project. It combines embedded firmware, edge computing, machine learning, and a cloud-hosted web dashboard into a single end-to-end IoT system.

The system orients a solar panel toward the sun on two axes (azimuth and elevation) to maximize energy capture, monitors panel and battery power in real time, and uses a camera with an on-device ML model to detect dirt accumulation on the panel surface.

![LightTrack prototype](docs/images/prototype.jpg)

## Repository Structure

This is a parent repository linking four submodules:

| Submodule | Description |
|---|---|
| `solar-firmware` | ESP32 firmware (Arduino/C++): tracking, sensors, display, MQTT |
| `solar-pi` | Raspberry Pi gateway (Python): MQTT/cloud bridge, camera, ML inference |
| `solar-api` | Backend (Node.js/Express): REST API, WebSocket server, database |
| `solar-dashboard` | Web dashboard (Next.js/TypeScript): monitoring and control UI |

## System Architecture

The system has four layers, from hardware to user interface:

```
ESP32 (panel)  <-- MQTT -->  Raspberry Pi (gateway)  <-- WebSocket -->  Backend (cloud)  <-->  Dashboard (browser)
                                                                            |
                                                                            v
                                                                        Supabase
                                                                 (PostgreSQL + Storage + Auth)
```

![Architecture diagram](docs/images/architecture.png)

### 1. ESP32 - Device Layer

The ESP32 controls the physical panel:

- Dual-axis tracking with two servo motors (azimuth and elevation), driven by four LDR light sensors
- Astronomical sunrise/sunset calculation to know when tracking should be active
- Power monitoring with two INA219 sensors: solar panel output and battery (voltage, current, estimated state of charge, charging status)
- Daily energy counters persisted in non-volatile storage, so data survives restarts
- Local OLED display with live status
- Publishes telemetry and receives commands over MQTT

### 2. Raspberry Pi - Gateway Layer

The Raspberry Pi acts as the local hub and the "smart" edge device:

- Hosts the local MQTT broker (Mosquitto) that the ESP32 connects to
- Bridges the local network to the cloud: forwards telemetry upstream over a WebSocket connection and relays commands downstream to the ESP32
- Captures images of the panel with a camera module
- Runs a TensorFlow Lite model locally to classify the panel surface (edge AI: inference happens on the Pi, raw images are never required in the cloud for processing)
- Includes a frame quality gate (rejects images that are too dark, too bright, obstructed, or not showing the panel) so the ML model only evaluates valid frames
- Generates a visual surface analysis overlay highlighting deposit zones on the panel
- Uploads images directly to Supabase Storage over HTTPS

### 3. Backend - Cloud Layer

A Node.js/Express server hosted on Railway:

- Maintains two WebSocket channels: one for the device gateway (`/ws/device`, API key auth) and one for browser clients (`/ws/client`, JWT auth)
- Validates all incoming payloads with schema validation (Zod)
- Is the single writer to the database: persists telemetry, vision results, events, device status, and the full command audit trail
- Exposes a REST API for historical data queries and command creation
- Sends email alerts for important events

### 4. Dashboard - User Layer

A Next.js web application hosted on Vercel:

- Live telemetry: panel power, battery state, servo positions, light sensor readings
- Historical charts for energy production and consumption
- Dirt detection results with the captured image and the surface analysis overlay
- Remote control: manual servo positioning, tracking mode toggle, on-demand camera capture
- Command history with live status (pending, sent, acknowledged)
- Authentication via Supabase Auth; live updates pushed over WebSocket

![Dashboard screenshot](docs/images/dashboard.png)

## Data Storage - Supabase

Supabase provides three services to the system: a PostgreSQL database for structured data, object Storage for images, and Auth for user accounts.

### Database

All persistent data lives in PostgreSQL, organized in six tables:

| Table | Contents |
|---|---|
| `sensor_readings` | Full telemetry snapshots: servo angles, tracking mode, LDR values, battery voltage/percent/status, solar power and daily energy, charging metrics |
| `vision_results` | Dirt detection outcomes: dirt level, cleanliness percent, cleaning decision, model confidence, paths to the raw and processed images |
| `camera_captures` | Manual on-demand captures (no ML), linked to the command that triggered them |
| `device_commands` | Audit trail of every command with its lifecycle: pending, sent, acknowledged, failed |
| `device_status` | Online/offline state and last-seen timestamp for the ESP32 and the Pi |
| `system_events` | Structured event log with severity levels (info, warning, error, critical) |

Database constraints enforce data validity at the lowest level: angle ranges, sensor value bounds, allowed command types and statuses are all checked by PostgreSQL itself, not only by application code.

### Write path - who saves the data

Only the backend writes to the database, using a privileged service-role key that exists exclusively on the server. The flow for telemetry:

1. ESP32 publishes a reading over MQTT
2. The Pi validates it and forwards it through the WebSocket
3. The backend validates the payload again and inserts it into `sensor_readings`
4. The same message is broadcast live to all connected dashboards

The Pi never writes to the database directly. It only talks to the backend over the WebSocket, and the backend persists everything. This keeps a single, validated write path and a clean security model: the database credentials with write access never leave the server.

### Read path - how the dashboard gets data

The dashboard reads in two ways:

- Live data arrives as WebSocket push messages from the backend (no polling)
- Historical data (charts, command history, event logs) is fetched through the REST API, which queries the database

Authenticated users have read-only access enforced by Row Level Security policies in PostgreSQL, so even the public client key cannot modify data.

### Image storage

Images take a separate path from telemetry, because binary files do not belong on a realtime channel:

1. The Pi captures and processes the image locally
2. The Pi uploads the image files (raw and processed overlay) directly to Supabase Storage over HTTPS
3. Only the metadata (storage paths, classification result, confidence) is sent through the WebSocket to the backend, which saves it in `vision_results`
4. When the dashboard needs to display an image, it requests a short-lived signed URL from Storage

This keeps the WebSocket channel lightweight and lets Storage serve images efficiently to the browser.

![Dirt detection result](docs/images/dirt_detection.png)

## Command Flow - controlling the panel from the website

Example: the user moves the panel manually from the dashboard.

1. The dashboard sends the command to the backend via the REST API
2. The backend records it in `device_commands` (status: pending) and pushes it through the device WebSocket to the Pi
3. The Pi publishes it on the local MQTT command topic
4. The ESP32 receives the command, validates the target angles, and moves the servos
5. The ESP32 publishes an acknowledgment on MQTT, which travels back up the same chain
6. The backend updates the command status to acknowledged and notifies the dashboard live

Every command is therefore fully auditable: when it was created, when it reached the device, and whether it was executed.

One exception: the image capture command is handled entirely on the Pi (it owns the camera) and is never forwarded to the ESP32.

## Technologies

| Area | Stack |
|---|---|
| Firmware | C++ (Arduino), MQTT, NTP, NVS persistence |
| Gateway | Python, paho-mqtt, Mosquitto, picamera2, OpenCV, TensorFlow Lite |
| ML | MobileNetV2 transfer learning, trained in Google Colab, TFLite float16 export |
| Backend | Node.js, Express, WebSocket (ws), Zod, Resend (email) |
| Frontend | Next.js 14, TypeScript, Tailwind CSS, shadcn/ui, Recharts, SWR |
| Database & Auth | Supabase (PostgreSQL with RLS, Auth, Storage) |
| Hosting | Railway (backend), Vercel (frontend) |

## Hardware

- ESP32 DevKit-C
- Raspberry Pi 3B with Camera Module 3 Wide
- 2x SG90 servo motors (azimuth, elevation)
- 4x LDR light sensors
- 2x INA219 current/voltage sensors (panel, battery)
- SSD1306 OLED display
- CN3791 MPPT charge controller, 2S Li-ion battery pack, LM2596 buck converter

![Hardware assembly](docs/images/hardware.jpg)
