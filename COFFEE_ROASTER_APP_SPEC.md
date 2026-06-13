# Skywalker iTop Coffee Roaster — Web Application Specification

## Executive Summary

Build a **local-first, automation-first** web application for automated control and monitoring of a Skywalker iTop coffee roaster connected via ESP32 over USB serial. The app runs entirely on the user's machine and is accessed through a browser. Manual control is always available as a per-channel override but is not the primary workflow.

**Tech Stack:**
- **Backend:** Python + FastAPI — serial communication, background PID control loop, SSE streaming
- **Frontend:** React + TypeScript — real-time charts, controls, roast history dashboard
- **Database:** SQLite — local, offline-first, all roast data stored on device
- **Packaging:** Browser-only. Start the backend with `uvicorn` and open `http://localhost:8080` in any browser. No Electron required.

**Primary development platform:** macOS (USB serial is straightforward; safe for the laptop). Windows is a supported secondary target — the only difference is the serial port path format (`COM*`).

---

## 1. Hardware & Serial Communication Protocol

### 1.1 Confirmed Serial Protocol

- **Baud rate:** 115200
- **Encoding:** Newline-terminated ASCII text in both directions
- **Port:** User-configurable via config file or environment variable (not hardcoded)
  - macOS default: `/dev/tty.usbserial-*` or `/dev/cu.usbserial-*`
  - Linux default: `/dev/ttyUSB*` or `/dev/ttyACM*`
  - Windows default: `COM3`, `COM4`, etc.

**Host → ESP32 commands:**

| Command | Effect |
|---|---|
| `READ` | Request a status/temperature reading |
| `OT1;<0-100>` | Set heater duty (0–100%) |
| `OT2;<0-100>` | Set vent/fan duty (0–100%) |
| `DRUM;<0\|1>` | Drum motor off (0) / on (1) |
| `COOL;<0-100>` | Cooling fan duty (post-roast) |
| `FILTER;<0-100>` | Filter fan duty |
| `OFF` | Full shutdown (all outputs to zero) |
| `ESTOP` | Emergency stop — heater=0, vent=100 immediately |
| `UNITS;C` / `UNITS;F` | Set temperature units |

**ESP32 → Host (response to READ):**
```
0.0,<BT>,<ET>,<heat%>,<vent%>,0\n
```
Example: `0.0,185.32,210.12,85,40,0`

BT = Bean Temperature, ET = Environment Temperature (floats, already computed by firmware).

### 1.2 Hardware Failsafe — Critical

The ESP32 firmware has a built-in 10-second autonomous shutdown: if it does not receive a valid parsable command within 10 seconds it cuts all outputs. The backend **must** send a heartbeat command (current setpoints or a benign `READ`) at least every **5 seconds** during idle. The active PID control loop satisfies this naturally at 200ms.

### 1.3 Low-Level Binary Protocol (Reference Only)

The ESP32 handles all binary communication with the roaster internally. The host never sees or sends binary frames — only the text commands above. This section is retained for reference only.

---

## 2. Core Purpose — Automation First

This is an **automation-first roast controller.** The default operation is automated PID control following a user-selected roast profile. Manual control (via sliders) is always available as a per-channel override but is a fallback, not the primary mode. All UI design decisions should reflect this priority.

---

## 3. Roast Profiles

### 3.1 Waypoint-Based Profiles

A roast profile is a named sequence of waypoints, each containing elapsed time and target temperature:

| Time | Target BT |
|---|---|
| 0:00 | 150°C |
| 4:00 | 180°C |
| 8:00 | 210°C |

- The frontend provides a simple table UI to add, edit, and reorder waypoints.
- The backend linearly interpolates between waypoints to produce a continuous `target_temp(t)` curve.
- The PID controller chases this interpolated curve in real time throughout the roast.
- Profiles may optionally include **fan waypoints** (elapsed time → target fan %). If absent, a sensible default fan % is used and the user can override manually.
- Profiles are named, saved, and reusable across roasts.

### 3.2 Replicate a Previous Roast

- Every roast records the actual BT curve at 200ms resolution.
- Any past roast can be **promoted to a profile**: the recorded BT curve is resampled into waypoint pairs and saved as a named profile.
- On subsequent roasts, the PID chases that recorded curve — it does **not** replay raw heat/fan commands.
- This is the primary "replicate a previous roast" workflow.

---

## 4. Rate of Rise (RoR)

Three tiers in priority order:

1. **Live RoR (must-have, Phase 1):** A rolling RoR value in °C/min is computed from BT using a rolling 30-second window and displayed live on the roast screen, updated in real time via SSE.
2. **RoR History (Phase 5):** RoR curves from past roasts are shown in the history and roast detail views so the user can study and compare profiles.
3. **RoR Profile Target (Deferred / Phase 6+):** The ability to define an explicit target RoR curve as a profile type, with the automation system chasing RoR directly rather than absolute temperature.

---

## 5. Backend Architecture — Threading Model

### 5.1 Two-Layer Architecture

The backend has two strictly separated concerns:

**Layer 1 — Hardware Thread (`threading.Thread`):**
- Owns the serial port exclusively
- Runs the 200ms polling + PID control loop
- Sends commands to the ESP32 and maintains the heartbeat
- Computes calibrated temperatures, `target_temp(t)`, PID outputs
- Enforces all safety checks and failsafes
- Pushes state snapshots and events into a thread-safe `queue.Queue`

**Layer 2 — FastAPI (async):**
- Handles all HTTP REST endpoints (control commands, pre-roast form submission, history queries)
- Reads from the hardware thread's queue to serve the SSE stream
- Never touches the serial port directly
- Uses `threading.Lock` for any shared mutable state reads

### 5.2 Thread-Safe Communication

```
Hardware Thread  →  queue.Queue (events/snapshots)  →  FastAPI SSE handler  →  Browser
Browser  →  HTTP POST (control commands)  →  FastAPI  →  shared command object (Lock-protected)  →  Hardware Thread reads on next loop tick
```

This separation keeps the 200ms hardware loop reliable regardless of HTTP request load.

---

## 6. PID Controller

### 6.1 Control Loop

- Loop frequency: **200ms** (configurable)
- On each tick the hardware thread:
  1. Sends `READ` and parses the BT/ET response
  2. Applies calibration offset (if active) to get calibrated BT
  3. Computes `target_temp(t)` from the active profile's interpolated waypoints
  4. Runs the heater PID and fan PID
  5. Applies safety checks and clamps outputs
  6. Sends `OT1` and `OT2` commands (or maintains heartbeat if idle)
  7. Pushes a state snapshot to the event queue

### 6.2 Dual PID Controllers

**Heater PID (controls OT1):**
- Goal: track `target_temp(t)`
- Output range: 0–100%
- Starting tuning (user-adjustable in settings): Kp=2.0, Ki=0.1, Kd=1.0
- Safety: cap output if BT > target + 2°C; anti-windup enabled; rate-limit changes per tick

**Fan PID (controls OT2):**
- When fan waypoints are present: chases the fan waypoint curve
- When absent: runs at a configurable default (e.g. 50%) with PID rules to prevent overshoot
- Post-roast cooling: automatically drives OT2 to 100% on drop or END
- Starting tuning (user-adjustable): Kp=1.5, Ki=0.05, Kd=0.5

### 6.3 Per-Channel Manual Override

- Both heater and fan sliders are visible and interactive at all times during a roast.
- Adjusting a slider switches **only that channel** to manual override; the other channel remains on PID automation.
- The PID continues computing outputs for overridden channels internally (so Resume Auto is smooth and jump-free).
- A per-channel **Resume Auto** button returns the channel to PID control, blending smoothly from the current manual value.
- Override state is logged per event (timestamp + channel + value).

### 6.4 Failsafes

- **ESTOP:** Always visible. Immediately sends `ESTOP` (heater=0, vent=100), logs the event, and locks out automation until manually cleared.
- **Over-temperature:** If BT exceeds a user-configurable maximum (default 230°C), ESTOP triggers automatically.
- **Serial disconnect:** If the serial port is lost or READ responses stop, the backend stops sending heater commands, displays a prominent connection warning, and attempts reconnection.
- **Heartbeat:** The hardware thread sends current setpoints at least every 5 seconds while idle to prevent the ESP32's 10-second autonomous shutdown.

---

## 7. Real-Time Streaming — Server-Sent Events (SSE)

SSE is used for all server-to-browser real-time data. It is one-directional (server → browser), which is all that is needed. SSE is simpler than WebSockets, has no connection management overhead, and is fully sufficient for a single-user local app.

**SSE stream carries (200ms cadence):**
- BT and ET (raw and calibrated)
- Live RoR (°C/min, rolling 30s window)
- `target_temp(t)` (current profile target)
- Heater % and fan % (actual sent values)
- Automation state per channel (auto / manual override)
- Roast phase (idle / preheat / roast / cooling)
- Active event markers (Dry End, First Crack, Drop timestamps)
- Drop countdown value (seconds remaining, once First Crack is marked)
- Connection status

**Control commands (browser → server) are plain REST POST endpoints:**
- Set heater % (manual override)
- Set fan % (manual override)
- Resume auto (per channel)
- Start / pause / end roast
- Mark Dry End event
- Mark First Crack event
- Mark Beans Dropped event
- ESTOP

---

## 8. Pre-Roast Configuration

Before starting each roast the user completes a simple form. All fields are stored with the roast record.

| Field | Type | Notes |
|---|---|---|
| Roast name / batch label | Text | Free text |
| Green coffee weight | Number (g) | For logging |
| Coffee variety / origin | Text | Free text |
| Profile | Select | Choose saved profile or "Manual" |
| Target drop delay after First Crack | Number (seconds) | Per-roast, e.g. 60s for light roast |
| Fan profile | Select or default % | Optional |

This is a simple form, not a complex settings system.

---

## 9. Roast Events

The roast screen has two prominent one-click event buttons:

- **Dry End** — stamps the current timestamp + BT into the roast log
- **First Crack** — stamps the current timestamp + BT into the roast log, and starts the drop countdown

Events are manual — there is no auto-detection. They are stored with the roast record and displayed as markers on both the live chart and the post-roast chart.

---

## 10. Drop Suggestion & Countdown

1. The per-roast target drop delay is set on the pre-roast form (e.g. 60 seconds after First Crack for a light roast).
2. When the user clicks **First Crack**, a visible countdown timer starts on the roast screen.
3. When the countdown reaches zero:
   - Audio + visual alert fires
   - `COOL;100` is sent automatically (cooling fan activates)
   - A prompt appears: **"Drop now — or defer?"** with defer options (+15s, +30s)
4. The user physically drops the beans (there is no automated drop mechanism on this roaster).
5. The user clicks **Beans Dropped** to log the drop event, end the automated roast phase, and begin the post-roast cooling phase.

---

## 11. Data Logging & History

**Log resolution:** 200ms — each record contains: timestamp offset, BT, ET, calibrated BT, heater%, fan%, `target_temp`, phase, manual override flags.

**Event markers stored:** Start, Dry End, First Crack, Drop (Beans Dropped), End, ESTOP, manual override toggles.

**Profile promotion:** Any past roast can be promoted to a reusable profile from the history view. The BT curve is resampled into waypoint pairs.

**Export:** Full roast data exportable as CSV or JSON. SQLite DB file can be backed up or restored.

---

## 12. Temperature Calibration (Phase 6)

Temperature readings are typically offset (consistently high or low) but stable within a roast session. Calibration is deferred to Phase 6:

1. During early roasts of a new bean, the user manually marks key events (Dry End, First Crack, Drop).
2. The backend compares marked temperatures against expected targets and computes a per-bean offset.
3. After 3–5 rated roasts of the same bean type, the system suggests a calibration offset with a confidence score. The user accepts or rejects it.
4. During the learning phase the UI shows both raw and calibrated temperatures.

---

## 13. Smart Suggestions / ML (Deferred — Phase 6)

Machine learning features are explicitly deferred until the core app is complete and real roast data exists. This is a low-priority nice-to-have, not a project requirement. No ML code should be written until Phase 5 is complete.

---

## 14. Development Phases

### Phase 1 — Serial Communication & Hardware Thread
- Implement `pyserial` read/write in a dedicated `threading.Thread`
- Parse READ responses; send OT1/OT2/DRUM commands
- Implement 5-second idle heartbeat to satisfy ESP32 failsafe
- Validate independently from Artisan (use a simple Python script first)
- Confirm serial port detection on macOS

### Phase 2 — FastAPI + SSE + Minimal React UI
- FastAPI app with SSE endpoint consuming the hardware thread's `queue.Queue`
- Minimal React page showing live BT/ET numbers and connection status
- REST endpoints for ESTOP and basic manual control
- Confirm end-to-end: hardware → Python → SSE → browser

### Phase 3 — PID Control Loop & Automation UI
- Implement dual PID (heater + fan) in the hardware thread
- Per-channel manual override and Resume Auto
- ESTOP always accessible
- Safety checks: over-temperature, serial disconnect
- Basic roast start/stop flow

### Phase 4 — Roast Profiles & Playback
- Waypoint table UI (add/edit/delete waypoints)
- Linear interpolation of `target_temp(t)` from waypoints
- Save/load named profiles
- Record full BT curve at 200ms resolution
- Promote past roast to profile

### Phase 5 — Roast History, Charts, RoR History & UI Polish
- Full roast history dashboard
- Live and post-roast charts (BT, ET, target curve, RoR, event markers)
- RoR history curves in past-roast detail view
- Pre-roast configuration form
- Drop countdown + Dry End / First Crack / Beans Dropped event buttons
- Export (CSV/JSON)

### Phase 6 — Calibration, Smart Suggestions & RoR Profile Targeting (Deferred)
- Temperature offset calibration and per-bean learning
- Smart drop suggestions based on past roast patterns
- RoR target curve as a profile type (PID chases RoR directly)
- ML-based profile optimization

---

## 15. Startup & Deployment

```bash
# Start the backend
uvicorn backend.main:app --host 127.0.0.1 --port 8080 --reload

# Start the frontend (development)
npm start
```

- The frontend can also be served as a static build by the backend for a single-command startup.
- Open `http://localhost:8080` in any browser.
- Serial port is configured via `config.yaml` or environment variable. The app will attempt auto-detection if unspecified.
- No installation, packaging, or code signing required. This is a developer-run local tool.

---

## 16. Safety Checklist (First Runs)

Before roasting beans:
- [ ] Confirm READ responses parse correctly (BT/ET values plausible)
- [ ] Confirm OT1 and OT2 commands echo back in READ response
- [ ] Confirm ESTOP fires immediately and heater drops to 0
- [ ] Confirm heartbeat prevents ESP32 shutdown during idle
- [ ] Confirm serial disconnect warning appears when USB is unplugged
- [ ] Run a dry empty-drum test at low heat (20–30%) and verify PID stability
- [ ] Verify over-temperature cutoff fires correctly at configured threshold
