# Skywalker iTop Coffee Roaster Web Application - Complete Project Specification

## Executive Summary

Build a local-first, offline-capable web application for automated control and monitoring of a Skywalker iTop coffee roaster connected via ESP32. The app will handle back-to-back light roasts (8–12 minutes) with short breaks and provide automation-first PID control, manual overrides as a fallback, full roast history, and on-device learning for calibration and profile optimization.

**Tech Stack:**
- **Backend:** Python + FastAPI (real-time PID control, serial communication, ML)
- **Frontend:** React + TypeScript (real-time graphs, controls, history dashboard) — browser-first
- **Database:** SQLite (local, offline-first, all roast data stored)
- **ML:** scikit-learn or TensorFlow Lite (local temperature offset calibration, bean profile learning)
- **Packaging:** Browser-only app. Run the server locally and open the UI in a browser (no Electron required).

---

## 1. HARDWARE & COMMUNICATION PROTOCOL

This section documents both the low-level binary protocol implemented between the ESP32 and the roaster (for reference) and the confirmed text-based serial protocol used between the host (backend) and the ESP32 firmware.

### 1.0 Serial Communication Protocol (confirmed)

- Baud rate: **115200**
- Commands are newline-terminated ASCII text commands from the host to the ESP32.

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
  - BT and ET are floats produced by the firmware (may be raw or calibrated per firmware code).

Hardware failsafe behavior (firmware):
- If the ESP32 does not receive a valid (parsable) command within **10 seconds** it will autonomously shut down outputs.
- Because of this, the host must send commands periodically during idle. Recommended: send a benign control command (e.g., current heater/vent setpoints) at least every **5 seconds** while idle; the control loop will send updates more frequently during active roasting.

Serial port configuration and defaults:
- Serial port path is configurable (config file or environment variable), not hardcoded.
- Example defaults:
  - macOS: `/dev/tty.usbserial-*` or `/dev/cu.usbserial-*`
  - Linux: `/dev/ttyUSB*` or `/dev/ttyACM*`
  - Windows: `COM3`, `COM4`, etc.
- Baud: 115200 (fixed)

---

### 1.1 ESP32 ↔ Roaster (low-level binary protocol, reference)

(Kept for reference from the repository's SkyCommand firmware.)
- Roaster → ESP32 (binary message): 7 bytes (BT, ET, reserved, checksum). The ESP32/firmware converts this binary message into the textual READ response above.
- ESP32 → Roaster (binary outgoing): 6 bytes with a preamble — the ESP32 sends the low-level pulses to the roaster; the host controls the ESP32 using the text commands documented in the Serial Communication Protocol.

### 1.2 Temperature Calibration Challenge

**Known Issue:** Temperature readings from the sensor chain (firmware → ESP32 → host) can be consistently offset (too high or too low) but are typically stable for the duration of a roast.

**Solution Approach:**
1. During initial roasts of a new bean, the user manually marks key events (Dry End / First Crack / Drop / End).
2. The backend compares the marked temperatures with expected targets and learns a per-bean temperature offset.
3. After 3–5 rated roasts of the same bean type, the system suggests a calibration offset and a confidence score; if accepted, the offset is applied to future roasts.
4. The UI displays both raw sensor temperature and calibrated temperature during the learning phase.

---

## 2. CORE PURPOSE — AUTOMATION FIRST

This application is automation-first: the default operation mode is automated PID control following a user-selected roast profile (waypoint/curve). Manual control is available at any time as a per-channel override; it is a fallback, not the primary mode. The UI and system behaviors are organized around running automated roasts reliably and safely.

---

## 3. ROAST PROFILES — WAYPOINTS & REPLICATION

### 3.1 Profile Definition (waypoint table)

A roast profile is a sequence of waypoints: rows of (elapsed time, target temperature). Example:
- `0:00 → 150°C`
- `4:00 → 180°C`
- `8:00 → 210°C`

- The frontend provides a simple table UI to add/edit waypoints.
- The app linearly interpolates between waypoints to produce a continuous target temperature curve target_temp(t).
- The PID controller chases this interpolated curve in real time during the roast.
- Profiles may optionally include fan waypoints (elapsed time, target fan %). If no fan waypoints are supplied, sensible defaults are used.
- Profiles are named, saved, and reusable.

### 3.2 Profile Replication from Previous Roasts

- Every roast records the actual BT curve at 200ms resolution.
- Any past roast can be promoted to a profile: the recorded BT curve is resampled/normalized into waypoint pairs and stored as a profile. The PID then chases that recorded curve (the PID does not replay raw commands).
- This is the primary "replicate a previous roast" workflow.

---

## 4. RATE OF RISE (RoR)

Priority tiers:
1. Live RoR (must-have): display a rolling RoR (°C/min) on the roast screen, computed from BT using a rolling window (suggested: 30s) and updated in real time.
2. RoR history: show RoR curves from past roasts in the history/detail view.
3. (Deferred) Ability to define an explicit RoR target curve as a profile type and have the automation chase RoR directly.

---

## 5. PID CONTROLLER & AUTOMATION

### 5.1 Control Loop: threading model and timing

- The serial polling + PID control loop must run in a dedicated background thread (Python's threading.Thread), not as an asyncio coroutine.
- Loop frequency: **200ms** (modifiable). This interval provides responsive control without overwhelming the host or the ESP32 and matches the roaster's thermal dynamics.
- The serial/PID thread responsibilities:
  - Read sensor data from ESP32 (READ responses), compute calibrated temps
  - Compute target_temp(t) from the selected profile (interpolated waypoints)
  - Run dual PID controllers (heater and fan) and compute outputs
  - Enforce safety checks and failsafes
  - Send control commands to the ESP32 frequently during active roasts and maintain an idle heartbeat (send current setpoints at least every 5 seconds) to avoid the ESP32 10s failsafe
  - Push state updates into a thread-safe queue or snapshot object for the FastAPI layer to read

Thread-safe communication with FastAPI:
- Use thread-safe primitives (queue.Queue, threading.Lock) to share state and events between the hardware thread and FastAPI's async handlers.
- The FastAPI SSE endpoint consumes snapshots or event queues and pushes updates to the browser.

### 5.2 Dual PID Controllers (heater + fan)

Heater PID (controls heater %):
- Goal: Follow target_temp(t)
- Output range: 0–100%
- Example starting tuning (user-adjustable): Kp=2.0, Ki=0.1, Kd=1.0
- Safety constraints: cap heater if temp > target + 2°C; anti-windup and rate-limited changes

Fan PID / Fan control:
- The fan can be controlled either via fan waypoints or via an active controller that helps shape RoR and prevent overshoot.
- If no fan waypoints are defined in profile, the system applies sensible defaults (e.g., 40–60%) and uses fan PID rules to protect against overshoot or to cool after END.
- Example starting tuning (user-adjustable): Kp=1.5, Ki=0.05, Kd=0.5

### 5.3 Per-channel manual override (automation-first behavior)

- Heater and fan are driven by automation by default.
- The UI shows sliders for both channels at all times. Touching a slider switches only that channel to manual override. The other channel remains automated.
- The automation thread continues to compute PID outputs for overridden channels but does not send them; this allows a smooth Resume Auto behavior.
- A per-channel "Resume Auto" button returns the channel to automation. The PID should smoothly blend the output to avoid jumps.
- ESTOP always takes priority and will immediately set heater=0 and vent=100.

### 5.4 Failsafes & Safety

- Emergency stop (ESTOP) immediately sets heater=0 and vent=100 and logs the event.
- Host must maintain a heartbeat (send current setpoints at least every 5s while idle) to avoid the ESP32's 10s autonomous shutdown.
- Backend must detect serial disconnects and immediately stop sending heater commands and display a prominent connection warning.
- If the temperature exceeds configured maximums (user-configurable, default e.g., 220°C), ESTOP is triggered automatically.

---

## 6. ROAST EVENTS — DRY END & FIRST CRACK

- The roast screen contains two prominent one-click event buttons: **Dry End** (aka Drop point for some workflows) and **First Crack**.
- Clicking either stamps the current timestamp and temperature into the roast log. These are manual events — there is no acoustic/vibration auto-detection.
- Once First Crack is marked, the drop countdown (per-roast configured delay) begins (see Drop Suggestion & Countdown logic below).
- Events are recorded with the roast data and displayed as markers on the post-roast chart.

---

## 7. DROP SUGGESTION & COUNTDOWN LOGIC

- Each roast has a per-roast configured target drop delay (set on the pre-roast form). Example: "drop 60 seconds after First Crack."
- When the user marks First Crack, the UI starts a visible countdown to the configured drop time.
- When the countdown reaches zero the backend/UI:
  - Fires audio + visual alert
  - Sends `COOL;100` to activate the cooling fan automatically
  - Presents a prompt: "Drop now — or defer?" with defer options (+15s, +30s)
- The user physically performs the drop and clicks **Beans Dropped** (or similar) to log the drop and transition to post-roast cooling.

---

## 8. OVERRIDE DURING AUTOMATION (details)

- While automation controls the roast, the heater/fan sliders remain active.
- Adjusting a slider places only that channel into manual override; the other channel remains automated.
- An indicator displays which channels are in auto vs manual mode.
- The Resume Auto button per channel snaps that channel back to PID control.
- ESTOP is always visible and overrides everything.

---

## 9. FAN AUTOMATION

- Automation controls both heater (OT1) and fan (OT2). Fan waypoints are optional in profiles.
- If fan waypoints are present, the PID/automation follows those fan targets.
- If fan waypoints are absent, a reasonable default is used (e.g., 50%) and the fan PID runs to manage temperature and RoR.
- During post-roast cooling the fan is driven to 100% automatically when appropriate (e.g., on END or at drop countdown expiry).

---

## 10. PRE-ROAST / PER-ROAST CONFIGURATION SCREEN

Before starting a roast the user fills a simple form that is stored with the roast record:
- Roast name / batch label
- Green coffee weight (g) and variety (free-text)
- Profile to run (select saved profile or choose "manual" )
- Target drop delay after First Crack (seconds)
- Fan profile selection or default fan %

This per-roast configuration is intentionally simple — not a complex settings page.

---

## 11. DATA LOGGING & HISTORY

- Log resolution: 200ms (timestamp offset, BT, ET, heater%, fan%, target_temp, phase, manual flags).
- Event markers saved: start, Dry End, First Crack, Drop, End, ESTOP, manual toggle events.
- Any recorded roast can be promoted to a profile; the recorded BT curve is resampled into waypoint pairs.
- Export: CSV/JSON of roast metadata and full time-series data; SQLite DB file backup/restore.

---

## 12. FRONTEND & REAL-TIME STREAMING — SSE

- Use Server‑Sent Events (SSE) for one-directional real-time streaming from the server to the browser. SSE will carry:
  - BT and ET values
  - PID outputs (heater%, fan%)
  - RoR values
  - Roast state updates and event markers
- Control actions (set heater/fan, start/stop roast, ESTOP, mark events) are REST POST endpoints.
- SSE is simpler than WebSockets for this single-user local app and integrates well with the backend thread model.

---

## 13. BACKEND: THREADING & ARCHITECTURE (summary)

- The background hardware thread handles serial I/O and the PID loop.
- FastAPI handles HTTP REST endpoints and serves an SSE stream by reading from a thread-safe event queue or snapshot.
- Use queue.Queue for event streaming (producer: hardware thread; consumer: SSE handler) and threading.Lock for snapshot reads.
- The hardware thread must maintain the recommended heartbeat cadence (send setpoints at least every 5s while idle).

---

## 14. DEVELOPMENT PHASES & TIMELINE (reordered)

1) Serial communication + background polling thread
   - Validate hardware comms from Python (pyserial) independent of Artisan
   - Confirm READ responses and command sending; implement heartbeat

2) FastAPI backend with SSE endpoint streaming live temperature and state
   - Minimal React page that subscribes to SSE and displays a live temperature value and status

3) PID control loop + per-channel manual override UI
   - Implement automation-first behavior, ESTOP, safety checks

4) Roast profile recording and playback
   - Waypoint editor, promote past roasts to profiles, interpolation logic

5) Roast history, full charting, RoR visualization, UI polish

6) Smart suggestions / ML (calibration & bean profile optimization)

Suggested durations are the same as previously listed (MVP ~2 weeks phases etc.) but reordered to validate hardware and SSE early.

---

## 15. TARGET PLATFORM

- Primary development/runtime target: **macOS** (USB serial is straightforward on Mac and development will be centered here).
- Windows and Linux are supported; only the serial port path format differs (COM* on Windows).

---

## 16. ERROR HANDLING, TESTING & SAFETY

- ESTOP always available and takes immediate precedence.
- The firmware enforces a 10s failsafe; the backend must send periodic commands (5s idle heartbeat recommended).
- On serial disconnect or repeated checksum failures, the backend should show a prominent connection lost state and disable automated heating.
- Do dry, empty-drum runs first and verify PID behavior before roasting beans.
- Implement logging and a small test checklist for first runs (verify READ parsing, setpoint echoing, ESTOP behavior, cooldown actions).

---

## 17. WHERE CHANGES WERE APPLIED

I integrated the 16 suggestions you provided directly into the original spec sections above. Key insertions and changes include:
- Explicit Serial Communication Protocol (section 1.0)
- SSE for real-time streaming (section 12)
- Removed Electron packaging and made the app browser-first (Executive Summary + packaging)
- Background threading model for serial/PID loop (sections 5 & 13)
- Automation-first emphasis (section 2)
- Waypoint-based profile definition and promotion of recorded roasts to profiles (section 3)
- Live RoR requirement and history (section 4)
- Dry End / First Crack event handling and drop countdown (sections 6 & 7)
- Per-roast pre-start configuration screen (section 10)
- Fan automation clarification (section 9)
- Reordered development phases to validate serial comms & SSE early (section 14)
- macOS as primary target (section 15)

---

## 18. DEPLOYMENT & STARTUP (updated)

- Start backend with uvicorn on localhost:8080 (example):
  `uvicorn backend.main:app --host 127.0.0.1 --port 8080 --reload`
- The frontend can be served as a static build by the backend or run via `npm start` during development; open the UI in your browser at http://127.0.0.1:8080 (or the port you choose).
- Serial port path is configured via a local config file or environment variable; the app will attempt to auto-detect if unspecified.

---

## 19. NEXT STEPS / SUGGESTED FIRST IMPLEMENTATION TASKS

- Implement Python serial thread skeleton (pyserial read/write, heartbeat, checksum handling). Use queue.Queue to pass events to FastAPI.
- Implement FastAPI SSE endpoint and minimal React SSE client showing live BT/ET and controls.
- Implement basic PID loop skeleton in the serial thread with safe defaults and emergency stop handling.
- Create a simple pre-roast form and wire START / END / event buttons to REST endpoints.

---

## 20. FINAL NOTES

This merged document preserves the original spec's content and structure while integrating your 16 suggestions inline where they belong. If you want, I can now:
- Run a follow-up commit to add a short example Python module that demonstrates the threading model, SSE endpoint, and minimal PID loop, or
- Re-run tests to validate READ parsing against your ESP32 using a serial loopback simulator if you prefer.

Which would you like me to do next?
