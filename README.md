# LightTrack

LightTrack is a dual-axis solar tracking system with automated dirt detection, developed as a bachelor's thesis project. It combines embedded firmware, edge computing, machine learning, and a cloud-hosted web dashboard into a single end-to-end IoT system.

The system orients a solar panel toward the sun on two axes (azimuth and elevation) to maximize energy capture, monitors panel and battery power in real time, and uses a camera with an on-device ML model to detect dirt accumulation on the panel surface.

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
```

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
- Runs a TensorFlow Lite model locally to classify the panel surface as clean, slightly dirty, or dirty
- Includes a frame quality gate (rejects images that are too dark, too bright, obstructed, or not showing the panel) so the ML model only evaluates valid frames
- Generates a visual surface analysis overlay highlighting deposit zones on the panel

### 3. Backend - Cloud Layer

A Node.js/Express server hosted on Railway:

- Maintains two WebSocket channels: one for the device gateway (`/ws/device`) and one for browser clients (`/ws/client`)
- Validates all incoming payloads with schema validation (Zod)
- Stores telemetry, vision results, and events in a PostgreSQL database (Supabase)
- Exposes a REST API for historical data queries
- Sends email alerts for important events

### 4. Dashboard - User Layer

A Next.js web application hosted on Vercel:

- Live telemetry: panel power, battery state, servo positions, light sensor readings
- Historical charts for energy production and consumption
- Dirt detection results with the captured image and the surface analysis overlay
- Remote control: manual servo positioning, tracking mode toggle, on-demand camera capture
- Authentication and realtime updates via Supabase

## Data Flow

**Telemetry (device to user):**

ESP32 reads sensors and publishes JSON telemetry over MQTT to the Pi. The Pi forwards it through the WebSocket to the backend, which validates it, stores it in the database, and broadcasts it to all connected dashboard clients. The dashboard updates in real time without page refresh.

**Commands (user to device):**

A user action on the dashboard (for example, moving the panel manually) is sent to the backend via WebSocket. The backend routes the command through the device WebSocket to the Pi, which publishes it on the local MQTT command topic. The ESP32 receives it, executes it, and the resulting state change flows back up through the normal telemetry path, confirming the action on the dashboard.

**Dirt detection:**

On a schedule or on demand from the dashboard, the Pi captures an image, runs the quality gate, performs ML inference, generates the surface analysis overlay, uploads the images to cloud storage, and sends the classification result upstream. The dashboard displays the verdict, confidence scores, and images.

## Technologies

| Area | Stack |
|---|---|
| Firmware | C++ (Arduino), MQTT, NTP, NVS persistence |
| Gateway | Python, paho-mqtt, Mosquitto, picamera2, OpenCV, TensorFlow Lite |
| ML | MobileNetV2 transfer learning, trained in Google Colab, TFLite float16 export |
| Backend | Node.js, Express, WebSocket (ws), Zod, Resend (email) |
| Frontend | Next.js 14, TypeScript, Tailwind CSS, shadcn/ui, Recharts, SWR |
| Database & Auth | Supabase (PostgreSQL, Auth, Realtime, Storage) |
| Hosting | Railway (backend), Vercel (frontend) |

## Hardware

- ESP32 DevKit-C
- Raspberry Pi 3B with Camera Module 3 Wide
- 2x SG90 servo motors (azimuth, elevation)
- 4x LDR light sensors
- 2x INA219 current/voltage sensors (panel, battery)
- SSD1306 OLED display
- CN3791 MPPT charge controller, 2S Li-ion battery pack, LM2596 buck converter
