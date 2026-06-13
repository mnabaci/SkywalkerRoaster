# Skywalker iTop Coffee Roaster Web Application - Complete Project Specification

## Executive Summary

Build a local-first, offline-capable web application that is automation-first for controlling a Skywalker iTop coffee roaster via an ESP32. The app's primary purpose is automated roast control (PID-driven) with manual control as a fallback. It supports back-to-back light roasts (8–12 minutes) with short breaks, stores full roast history, and learns calibration/profile adjustments over time.

Tech stack (recommended)
- Backend: Python + FastAPI (serial communication, PID control, ML)
- Frontend: React + TypeScript (browser UI using SSE for real-time streaming)
- Database: SQLite (local, offline-first)
- ML: scikit-learn or TensorFlow Lite (local, on-device learning)
- Packaging: Browser-only app (no Electron). Run the server locally and open the app in a browser.

---

## New: Serial Communication Protocol (explicit)

This is the confirmed text-based serial protocol to/from the ESP32 running SkyCommand-compatible firmware.

- Baud rate: 115200
- Commands are newline-terminated text commands from the host to ESP32.

Host → ESP32 (commands):
- `READ` → request a status/temperature line
- `OT1;<0-100>` → set heater duty (0–100%)
- `OT2;<0-100>` → set vent/fan duty (0–100%)
- `DRUM;<0|1>` → drum motor off/on
- `COOL;<0-100>` → cooling fan (post-roast) duty
- `FILTER;<0-100>` → filter fan duty
- `OFF` → full shutdown (all zeros)
- `ESTOP` → emergency stop (heat=0, vent=100)
- `UNITS;C` or `UNITS;F` → set units

ESP32 → Host (READ response):
- Response format (newline-terminated):
  `0.0,<BT>,<ET>,<heat%>,<vent%>,0\n`
  - Example: `0.0,185.32,210.12,85,40,0`
  - BT and ET are floats from the firmware (raw or calibrated per firmware code).

Hardware failsafe behavior (firmware):
- If the ESP32 does not receive a valid (parsable) command within 10 seconds it will autonomously shut down outputs.
- Because of this, the host must send commands periodically during idle. Recommended: send a benign control command (e.g., current heater/vent setpoints) at least every 5 seconds while idle; the control loop will send updates far more frequently during active roasting.

Serial port configuration and defaults:
- Serial port path is configurable (config file or environment variable), not hardcoded.
- Example defaults:
  - macOS: /dev/tty.usbserial-* or /dev/cu.usbserial-*
  - Linux: /dev/ttyUSB* or /dev/ttyACM*
  - Windows: COM3, COM4, etc.
- Baud: 115200 (fixed)

---

## 1. HARDWARE & COMMUNICATION (refined)

### 1.1 ESP32 & Roaster message formats

(Kept for reference from firmware code)
- Roaster → ESP32 (binary message): 7 bytes (BT, ET, reserved, checksum) — the ESP32/firmware converts binary sensor data into the textual READ response above. The project backend communicates to the ESP32 using the text commands documented in the Serial Communication Protocol.
- ESP32 → Roaster (binary outgoing): 6 bytes with preamble — the ESP32 sends the low-level pulses to the roaster; the host only controls the ESP32 using text commands.

### 1.2 Temperature calibration note
- Sensors may be consistently offset by a fixed amount between firmware reading and reality. The app will learn per-bean offsets and present both raw and calibrated temperatures during the learning period.

---

## 2. CORE PURPOSE — Automation-First

This application is an automation-first roast controller. The default operation mode is automated PID control following a user-selected roast profile (waypoint/curve). Manual control is available at any time as a per-channel override. The UI and system behaviors are organized around running automated roasts reliably and safely.

---

## 3. ROAST PROFILE DEFINITION

A roast profile is a waypoint table of (elapsed time, target temperature). Example rows: `0:00 → 150°C`, `4:00 → 180°C`, `8:00 → 210°C`.

- The frontend provides a simple table UI to add/edit waypoints.
- The app linearly interpolates between waypoints to produce a continuous target temperature curve (target_temp(t)).
- The PID controller chases this interpolated curve in real time.
- Profiles are named, saved, and reusable. Profiles may optionally include fan waypoints (see Fan Automation).

Profile replication from previous roasts:
- Every roast records the actual BT curve at 200ms resolution.
- A past roast can be promoted to a profile. When promoted, the recorded BT curve becomes the profile's waypoint list (the curve is resampled/normalized as waypoints) so the PID controller will chase that curve in a future roast.

---

## 4. RATE OF RISE (RoR)

Priority tiers:
1. Live RoR (must-have): display a rolling RoR (°C/min) on the roast screen, computed from BT using a rolling window (e.g., 30 seconds) and updated in real time.
2. RoR history: show RoR curves from past roasts in the history/detail view.
3. (Deferred) Explicit RoR target profiles: ability to define a RoR curve as a profile and have automation chase it.

---

## 5. PID CONTROL & AUTOMATION

Overview remains: dual PID controllers (heater + fan) running the control loop. Key adjustments with the user's suggestions:

- Control loop runs in a dedicated background thread (Python threading.Thread). See Backend: Python serial loop section below.
- Loop frequency: 200ms (modifiable).
- PID follows the interpolated waypoint curve (target_temp(t)). Fan channel may follow separate fan waypoints or run by its PID rules when no fan waypoints are present.
- Automation-first: controller drives outputs; manual slider interaction switches per-channel to manual override.

Per-channel override behavior:
- Heater and fan sliders are always visible.
- Touching a slider immediately sets that channel into manual override only; the other channel stays on automation.
- A visual indicator shows auto/manual state per channel and a per-channel "Resume Auto" button snaps the channel back to automation.
- ESTOP always takes precedence: it immediately sets heater=0 and vent=100 and logs the event.

Fan automation:
- Profiles optionally include fan waypoints. If undefined, a sensible default (e.g., 50%) is used.
- Fan PID can be driven by a target fan curve or by a temperature-based controller to help control RoR.

---

## 6. ROAST EVENTS & DROP LOGIC

Event buttons (manual):
- Dry End (aka DROP point for some workflows) and First Crack.
- Clicking either stamps current timestamp and temperature into the roast log.
- These events are manual; the app does not auto-detect acoustic or vibration events.

Drop suggestion/countdown logic:
- Each roast record contains a configured target drop delay (seconds after First Crack). This is set in the per-roast pre-start form.
- When the user marks First Crack, a countdown starts on the roast screen toward the target drop time.
- At countdown zero, the app:
  - Fires visual + audio alert
  - Sends `COOL;100` to activate the cooling fan automatically
  - Prompts the user: "Drop now — or defer?" with defer options (+15s, +30s)
- The user physically drops the beans and clicks "Beans Dropped" to log the drop event.

---

## 7. OVERRIDE DURING AUTOMATION

- The UI shows heater and fan sliders even when automation is active.
- Touching a slider places only that channel into manual override. The other channel remains automated.
- The automation thread will continue computing PID outputs for overridden channels but will not send them while the channel is overridden (so they can be resumed cleanly).
- A per-channel Resume Auto button returns the channel to automation and the PID controller will smoothly blend back to the controlled value.

---

## 8. DATA LOGGING & HISTORY (summary)

- Log resolution: 200ms (timestamp offset, BT, ET, heater%, fan%, target_temp, phase, manual flags).
- Event markers: start, Dry End, First Crack, Drop, End, ESTOP, manual toggles.
- Roast promotion: recorded BT curve can be promoted to a profile.

---

## 9. BACKEND: PYTHON SERIAL LOOP & THREAD MODEL (new details)

- The serial polling + PID loop MUST run in a dedicated background thread (threading.Thread) separate from FastAPI's asyncio event loop.
- Thread responsibilities:
  - Maintain the 200ms control loop timing reliably.
  - Read/write serial to ESP32 using pyserial.
  - Maintain heartbeat command cadence (send current setpoints at least every 5 seconds when idle to avoid the ESP32 10s failsafe).
  - Push events/state updates into a thread-safe queue (queue.Queue) for the API thread to read.
- FastAPI handlers and SSE endpoints read state from a thread-safe snapshot (protected by threading.Lock or using queue.Queue to consume events).
- Do not run the hardware loop as an asyncio coroutine: use threads to avoid timing jitter from the event loop and GIL interactions.

Serial error handling:
- If serial errors or checksum failures occur repeatedly, implement backoff and surface a "CONNECTION LOST" state to the UI.

---

## 10. UI & REAL-TIME STREAMING — Use SSE (Server-Sent Events)

- Replace WebSockets with Server-Sent Events (one-directional push) for real-time streaming of:
  - Temperature data (BT/ET)
  - Roast state updates (phase changes, event markers)
  - PID outputs (heater%, fan%) and RoR
- Control actions remain REST POST endpoints (e.g., /api/control/heater, /api/control/fan, /api/roast/start, /api/roast/estop).
- SSE is simpler for single-user local deployments, requires no extra connection management, and integrates cleanly with the backend thread model.

---

## 11. PRE-ROAST / PER-ROAST CONFIGURATION SCREEN (new)

Before starting each roast, the user sets:
- Roast name / batch label
- Green coffee weight (g) and variety (text)
- Profile to run (saved profile or "manual")
- Target drop delay after First Crack (seconds)
- Fan profile selection or default fan %

This is a simple form shown prior to `START ROAST` and stored with the roast record.

---

## 12. DEVELOPMENT PHASES & REORDERED TIMELINE (updated)

1) Serial communication + background polling thread
   - Validate hardware comms from Python independent of Artisan
   - Confirm READ/command exchange and heartbeat cadence

2) FastAPI backend with SSE endpoint streaming live temperature and status
   - Minimal React page that subscribes to SSE and displays a live number

3) PID control loop + manual override UI
   - Implement per-channel override, ESTOP, and safety checks

4) Roast profile recording and playback
   - Profile waypoint editor, promote past roast to profile, interpolation

5) Roast history, full charting, RoR, UI polish
   - Detailed roast views, RoR curves, comparison

6) Smart suggestions / ML (calibration & profile optimization)

---

## 13. TARGET PLATFORM

- Primary development and runtime target: macOS (USB serial, reliable development flow).
- Windows is supported as a secondary target (serial paths differ: COM*). Linux also supported.

---

## 14. SAFETY, FAILSAFES & OPERATION (highlights)

- ESTOP always available and takes precedence.
- Firmware enforces a 10-second failsafe; host must send periodic commands during idle (recommended every 5s).
- Backend must watch for temperature runaway and trigger ESTOP if configured limits are exceeded.
- On serial disconnect: backend should immediately prevent sending heater commands and surface the issue to the UI.

---

## 15. OTHER FEATURES (summary of earlier items)

- RoR live display, RoR history, event stamping (Dry End / First Crack), audio alerts + countdown to drop, per-channel manual override, fan waypoints optional, roast promotion to profile, per-roast pre-start form, full logging and export.

---

## 16. DEPLOYMENT & STARTUP (updated)

- Start backend with uvicorn on localhost:8080 (example):
  `uvicorn backend.main:app --host 127.0.0.1 --port 8080 --reload`
- Frontend served as static build or via `npm start` during development; open http://localhost:8080 (or http://127.0.0.1:8080) in browser.
- Serial port is configured in a local config file or environment variable; the app will attempt to auto-detect if unspecified.

---

## What's changed in the spec (summary)

1. Added an explicit Serial Communication Protocol section with command/response formats, and emphasized the 10s firmware failsafe and 5s idle heartbeat requirement.
2. Replaced WebSockets with SSE for real-time streaming.
3. Removed Electron packaging; app is a browser-first local web server.
4. Introduced a mandatory background thread model for the serial/PID loop and detailed thread-safe communication.
5. Made macOS the primary target platform.
6. Reordered development phases to validate serial comms and SSE early.
7. Reframed the product as automation-first.
8. Defined profiles as waypoint tables, with linear interpolation for PID target.
9. Added profile replication from past roasts (promote roast to profile).
10. Added RoR requirements and prioritized live RoR display.
11. Clarified Dry End / First Crack event handling and countdown/drop workflow.
12. Added per-roast pre-start configuration form.
13. Detailed per-channel override behavior and Resume Auto.
14. Clarified fan automation and optional fan waypoints.

---

If you want, I will commit this updated spec to the repository now. The target file path is `COFFEE_ROASTER_APP_SPEC.md` in the `mnabaci/SkywalkerRoaster` repository.
