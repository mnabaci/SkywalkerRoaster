# Skywalker iTop Coffee Roaster Web Application - Complete Project Specification

## Executive Summary

Build a local-first, offline-capable web application for automated control and monitoring of a Skywalker iTop coffee roaster connected via ESP32. The app will handle back-to-back light roasts (8-12 minutes each) with 3-5 minute intervals, providing real-time temperature curve visualization, PID-based automation, audio alerts for DROP/END timing, and ML-driven profile optimization through roast history analysis.

**Tech Stack:**
- **Backend:** Python + FastAPI (real-time PID control, serial communication, ML)
- **Frontend:** React + TypeScript (real-time graphs, controls, history dashboard)
- **Database:** SQLite (local, offline-first, all roast data stored)
- **ML:** scikit-learn or TensorFlow Lite (local temperature offset calibration, bean profile learning)
- **Desktop Wrapper:** Electron (optional Mac app packaging) or run as localhost web app

---

## 1. HARDWARE & COMMUNICATION PROTOCOL

### 1.1 ESP32 Communication Details

**Connection:**
- USB-C cable from ESP32 to MacBook
- Will appear as a virtual COM port (serial device)
- Baud rate: **115200**

**Message Protocol (from provided Arduino code):**

#### Roaster → ESP32 (Incoming Data) - 7 bytes
```
Byte 0-1: Bean temperature (BT) as 16-bit value
Byte 2-3: Environment/Exhaust temperature (ET) as 16-bit value
Byte 4-5: Reserved/unused
Byte 6: Checksum (sum of bytes 0-5)

Calculation:
- Raw value from bytes is divided by 1000 to get actual temperature
- Temperature is converted from F to C using 4th degree polynomial (see SkyCommand.ino calculateTemp())
```

#### ESP32 → Roaster (Outgoing Commands) - 6 bytes
```
Byte 0: Vent (Fan) duty cycle 0-100%
Byte 1: Filter duty cycle 0-100%
Byte 2: Cool duty cycle 0-100% (for post-roast cooling)
Byte 3: Drum motor control (0 = off, 100 = on)
Byte 4: Heat (heater) duty cycle 0-100%
Byte 5: Checksum (sum of bytes 0-4)

Message format:
- Preamble: 7500µs pulse (marks start of message)
- 3800µs delay after preamble
- Each bit: 1500µs pulse = '1', 650µs pulse = '0'
- 750µs delay between bits
- 6 bytes × 8 bits = 48 bits total
```

**Serial Commands (text-based, from Artisan protocol):**
- `READ` - Request temperature/status readback
- `OT1;VALUE` - Set heater duty (0-100)
- `OT2;VALUE` - Set vent/fan duty (0-100)
- `DRUM;VALUE` - Control drum (0 = stop, 1 = on)
- `FILTER;VALUE` - Set filter fan (0-100)
- `COOL;VALUE` - Set cooling fan (0-100)
- `OFF` - Shutdown all systems
- `ESTOP` - Emergency stop (heater 0, vent 100)
- `CHAN` - TC4 initialization
- `UNITS;C` or `UNITS;F` - Set temperature units

**Failsafe Behavior (built into ESP32):**
- If no command received for >1 second: automatic shutdown (all zeros)
- Temperature limit: 300°C triggers emergency stop
- Your backend must send periodic control commands to prevent timeout

### 1.2 Temperature Calibration Challenge

**Known Issue:** Temperature readings are consistently offset (too high or too low) but the offset doesn't drift during a roast.

**Solution Approach:**
1. First 3-5 roasts: User manually marks actual flavor development points (DROP, 1ST CRACK, END)
2. Compare marked temperatures vs. expected temperatures
3. Calculate fixed offset (e.g., "sensor reads +5°C too high")
4. Apply offset to all future roasts automatically
5. Store calibration history and confidence scores

**Important:** The app should display BOTH raw sensor temp and calibrated temp during initial learning phase.

---

## 2. ROASTING PROFILES

### 2.1 Light Roast Profile (Default - YOUR TARGET)

**Temperature Curve:**
```
Phase 1: Drying/Heating (0 - ~1 minute)
  - Start: Room temp (~20°C or ambient)
  - End: 150°C (DROP point)
  - Target ramp: +120°C/min (aggressive heating, no scorching yet)
  - Heater: 100%
  - Fan: 30-40%

Phase 2: Development to 1st Crack (~1 - ~5 minutes)
  - Start: 150°C (DROP)
  - End: 180-185°C (1st CRACK)
  - Target ramp: +8-10°C/min (gentle, controlled development)
  - Heater: 70-80%
  - Fan: 50-60%

Phase 3: Final Development (~5 - ~7 minutes)
  - Start: 180-185°C (1st CRACK)
  - End: 190-195°C (END/DROP)
  - Target ramp: +2-3°C/min (very gentle, short development after crack)
  - Duration: ~1 minute after 1st crack (or user can adjust)
  - Heater: 40-50%
  - Fan: 70-80%
```

**Key Notes for Light Roasts:**
- Tight temperature window (only ~10°C from 1st crack to end)
- User hears 1st crack but sometimes misses exact moment → app should monitor and alert
- Drop temp is critical to prevent scorching (150-160°C is the sweet spot)
- End temp is 190-195°C, roughly 1 minute after 1st crack

### 2.2 Medium & Dark Roast Profiles

**Medium Roast:**
```
Phase 1: Heating
  - Drop: 155-165°C
  - Ramp rate: +100°C/min

Phase 2: Development to 1st Crack
  - 1st Crack: 185-190°C
  - Ramp rate: +6-8°C/min

Phase 3: Development to 2nd Crack (optional)
  - End: 200-205°C (before or at 2nd crack)
  - Ramp rate: +1-2°C/min
  - Duration: ~2 minutes after 1st crack
```

**Dark Roast:**
```
Phase 1: Heating
  - Drop: 160-170°C
  - Ramp rate: +80°C/min

Phase 2: Development to 1st Crack
  - 1st Crack: 190-195°C
  - Ramp rate: +5-7°C/min

Phase 3: Development to 2nd Crack
  - End: 210-215°C (definitely past 2nd crack)
  - Ramp rate: +1-2°C/min
  - Duration: ~3-4 minutes after 1st crack
```

### 2.3 Profile Management (UI)

**Pre-defined profiles:** Light, Medium, Dark (built-in, tweakable)

**Bean-specific profiles:** 
- After 3+ roasts of same bean type → app suggests a custom profile
- User can save as "Ethiopia Yirgacheffe - Best" or similar
- Profiles should be editable: adjust ramp rates, drop temps, end temps, heater/fan percentages

**Profile Switching:** User can switch between profiles from UI before starting a roast

---

## 3. PID CONTROLLER LOGIC

### 3.1 Control Loop Timing

**Loop frequency:** Every 200ms
- Fast enough to respond to temperature changes
- Slow enough to avoid aggressive oscillation in a small roaster
- Matches typical roasting dynamics (6-10 second response time acceptable)

**Update cycle:**
1. Read temperature from ESP32 via serial
2. Calculate target temperature for current time
3. Run PID loop for heater power
4. Run PID loop for fan speed
5. Send new heater/fan values to ESP32
6. Log data (timestamp, BT, ET, target, heater %, fan %)
7. Repeat in 200ms

### 3.2 Dual PID Controllers

**Heater PID (controls Byte 4 - heat duty 0-100%):**
- **Goal:** Follow target temperature curve
- **Error calculation:** target_temp - current_temp
- **Output range:** 0-100% (heater duty)
- **Tuning parameters (starting values, user-adjustable):**
  - Kp (proportional): 2.0
  - Ki (integral): 0.1
  - Kd (derivative): 1.0
  - Max output: 100%
  - Min output: 0%

**Constraints:**
- Prevent overshoot: If temp > target + 2°C, cap heater at 20%
- Prevent oscillation: Limit rate of change (don't jump from 30% to 90% in one cycle)
- Anti-windup: If heater maxed out, stop accumulating integral error

**Fan PID (controls Byte 0 - vent duty 0-100%):**
- **Goal:** Prevent overheating, manage cooling rate
- **Rules:**
  - If current_temp > target_temp + 3°C: increase fan speed
  - If current_temp < target_temp: decrease fan speed
  - Always maintain some minimum fan (20%) to prevent hot spots
  - Can ramp to 100% during cooling phases
- **Tuning parameters (starting values, user-adjustable):**
  - Kp: 1.5
  - Ki: 0.05
  - Kd: 0.5

### 3.3 Manual Override Mode

**What triggers it:**
- User clicks "MANUAL MODE" button in UI
- Switches from automated PID to manual heater/fan sliders

**Manual controls:**
- Heater slider: 0-100% (real-time)
- Fan slider: 0-100% (real-time)
- Drum toggle: on/off
- Cool toggle: on/off

**Return to auto:**
- Click "AUTO MODE" button
- App resumes following target curve from current temperature

**Safety:** Manual mode cannot exceed 100% heater or 100% fan, and still respects the max temp failsafe

### 3.4 Failsafe Mechanisms

**Temperature Overshoot Protection:**
- If temp reaches 220°C (arbitrary safety limit, user-configurable): trigger ESTOP
- Log warning to database
- Alert user immediately

**Command Timeout:**
- Must send a control command to ESP32 every 500ms minimum
- If no command sent in 1 second → ESP32 auto-shuts down (built-in failsafe)
- Backend should maintain a "heartbeat" of control messages even in idle state

**Serial Connection Loss:**
- If serial read fails 5 times in a row → stop sending heater commands
- Switch UI to "CONNECTION LOST" state with big red warning
- User can manually trigger ESTOP if needed

**User Intervention:**
- Emergency STOP button (always available): Heater → 0, Fan → 100, Drum → 0
- Resets PID integral terms
- Logs the stop event with timestamp

---

## 4. ROASTING WORKFLOW & USER EXPERIENCE

### 4.1 Main Roasting Screen

**Layout:**
- **Top-left:** Live temperature graph (scrolling, 12-minute window)
  - X-axis: time (0 to 12 minutes)
  - Y-axis: temperature (100°C to 220°C)
  - Red line: target curve
  - Blue line: actual bean temp
  - Green line: actual env/exhaust temp
  - Shade areas: phase regions (drying, development, final)

- **Top-right:** Status panel
  - Current BT: XXX°C
  - Current ET: XXX°C
  - Target: XXX°C
  - Time elapsed: MM:SS
  - Phase: "Drying" / "Development" / "Final"
  - PID status: Heater 75%, Fan 60%

- **Middle:** Big red buttons
  - **DROP NOW** (when roast started, ready at ~1.5 min)
  - **END NOW** (red, always available during roast)
  - **MANUAL/AUTO toggle** (toggle mode)

- **Bottom-left:** Manual controls (only if in MANUAL mode)
  - Heater slider: 0-100%
  - Fan slider: 0-100%
  - Drum toggle
  - Cool toggle

- **Bottom-right:** Bean/Profile info
  - Selected bean type
  - Selected roast profile (Light/Medium/Dark/Custom)
  - Weight: XXX g
  - Notes input box (optional)

### 4.2 Pre-Roast Setup

**Steps:**
1. Select bean type from dropdown (or create new)
2. Select roast profile (Light/Medium/Dark or saved bean-specific profile)
3. Enter batch weight (for logging)
4. Enter any notes (optional)
5. Click "START ROAST"

**System preparation:**
- Verify serial connection to ESP32 (show status)
- Zero temperature reading from roaster
- Arm all controls (ready to send commands)
- Start PID controllers

### 4.3 During Roast - Event Markers

**User manually marks important events by clicking buttons:**

1. **DROP NOW button** (appears ~90 seconds into roast)
   - Records: timestamp, current temp (should be ~150°C)
   - Stores as "drop_temp" in roast record
   - Transitions to Phase 2 (Development)
   - Reset any integral error in heater PID

2. **1ST CRACK detection** (optional - user clicks when they hear it)
   - Records: timestamp, current temp (should be ~180-185°C)
   - App can suggest: "End roast in ~1 minute for light roast" (sound alert)
   - Transitions to Phase 3 (Final development)
   - Adjust heater/fan aggression for final push

3. **END NOW button** (user clicks or timer triggers)
   - Records: timestamp, current temp (should be ~190-195°C)
   - All controls → 0 (shutdown)
   - Fan → 100% for safety (cool the roaster)
   - Roast complete
   - Transition to POST-ROAST screen

### 4.4 Audio Alerts

**Timing:**
- **5 seconds before DROP:** Beep + text alert "Prepare to DROP"
- **At DROP point:** Loud beep + "DROP NOW"
- **5 seconds before END:** Beep + text alert "Prepare to END"
- **At END:** Loud beep + "END ROAST NOW"

**Customizable:**
- User can disable/enable sounds
- Adjust alert volume
- Adjust alert lead time (5s before, configurable)

**Note:** Sound is critical during roasting because visual cues are hard (roaster is hot, light is in the way, etc.)

### 4.5 Roast Complete / Post-Roast Screen

**After END is clicked:**
1. All heater/drum controls off
2. Fan runs at 100% for 30 seconds (auto-cool)
3. UI shows: "Roast complete! Cool for 1-2 minutes"
4. Display: final stats
   - Total time: MM:SS
   - Drop temp: XXX°C
   - 1st crack temp: XXX°C
   - End temp: XXX°C
   - Average ramp rate: X°C/min

**Post-roast form:**
- Rating: 1-10 (slider or star)
- Defects: checkboxes
  - [ ] Under-developed
  - [ ] Over-developed
  - [ ] Scorched
  - [ ] Uneven
  - [ ] Bitter
  - [ ] Grassy/Vegetal
  - (More as needed)
- Notes: free text
- Buttons: "SAVE ROAST" / "DISCARD"

**After SAVE:**
1. Data stored to SQLite
2. ML engine processes it (if this is bean type seen before)
3. UI shows: "Ready for next roast" or "Preparing next batch?"
4. User can:
   - Start another roast (same bean, same profile)
   - Select different bean
   - View roast history

---

## 5. MACHINE LEARNING & CALIBRATION ENGINE

### 5.1 Temperature Offset Calibration

**Problem:** Sensor reads X°C but actual bean temp is X ± offset°C

**Solution:**
1. First roast of a new bean type: mark DROP and 1ST CRACK manually
2. Backend records: sensor_temp_at_drop, sensor_temp_at_1st_crack
3. User later rates the roast (1-10)
4. After 3-5 roasts of same bean type:
   - Analyze marked temperatures
   - If consistent offset detected (e.g., "always reads +5°C high when rated 8-10"), calculate it
   - Generate calibration curve (linear regression initially)
   - Apply to future roasts of this bean type
   - Display: "Calibration applied: -5°C offset detected"

**Confidence scoring:**
- Each roast contributes a confidence score (0-1)
- Only apply calibration if confidence > 0.7
- Show user: "Calibration 87% confident"

**Refinement:**
- User can manually adjust offset by ±2°C in settings
- Each adjustment is logged with feedback

### 5.2 Bean Profile Optimization

**Goal:** Learn what heater/fan settings produce the best roasts for each bean type

**Data collection per roast:**
- Bean type
- Selected profile (Light/Medium/Dark)
- Heater % over time (logged every 200ms)
- Fan % over time (logged every 200ms)
- Final rating (1-10)
- Marked defects

**After 3+ roasts of same bean:**
- Cluster roasts by rating (high: 7-10, medium: 5-6, low: 1-4)
- Analyze heater/fan patterns in high-rated roasts
- Identify: "High-rated roasts had heater starting at 85%, declining to 45% by end"
- Suggest adjustment: "Try starting heater at 85% for next batch"

**UI Display:**
- "Based on 4 roasts of Ethiopia Yirgacheffe, suggested settings:"
  - Initial heater: 85%
  - Initial fan: 35%
  - Mid-roast adjustment: Reduce heater 2% every 30 seconds
- User can apply or ignore suggestion
- Can tweak manually in profile editor

### 5.3 Predictive Roast Timing

**Goal:** Predict END timing based on historical data

**Logic:**
1. User has roasted Ethiopia Yirgacheffe 5 times
2. Historical average: 1st crack at 4m 30s, end at 6m 15s
3. Current roast: 1st crack detected at 4m 32s
4. App predicts: "End roast at ~6m 20s (estimated based on history)"
5. Show on UI: countdown timer "End in 1m 50s"

**Accuracy improves with roast history**

### 5.4 ML Model Storage

**Where:** SQLite database
**What:** Store trained model data (offsets, coefficients, bean-specific averages)
**Update frequency:** After each rated roast
**Export/Backup:** User can export roast data as CSV/JSON for external analysis

**No internet required** - all processing local to machine

---

## 6. DATA LOGGING & HISTORY DASHBOARD

### 6.1 Real-Time Data Logging

**During each roast, log every 200ms:**
- Timestamp offset (seconds since roast start)
- Bean temperature (BT)
- Environment temperature (ET)
- Heater duty cycle (%)
- Fan duty cycle (%)
- Target temperature
- Current phase (drying/dev/final)
- Manual override flag (true if user in manual mode)

**Event markers:**
- Roast start time
- DROP event (time, temp)
- 1ST CRACK event (time, temp)
- Manual mode toggle (time)
- Emergency stop (if triggered, time, reason)
- Roast end time

### 6.2 Post-Roast Storage

**Roast record includes:**
- Bean type
- Batch weight (g)
- Profile used (Light/Medium/Dark/Custom)
- Start time (datetime)
- End time (datetime)
- Total duration (seconds)
- Drop temp (°C, calibrated)
- 1st crack temp (°C, calibrated)
- End temp (°C, calibrated)
- Peak temp (°C)
- Average ramp rate (°C/min)
- User rating (1-10)
- Defects (list of selected defects)
- Notes (free text)
- All raw time-series data (logged every 200ms)
- Calibration offset applied (if any)

### 6.3 History Dashboard

**Overview tab:**
- Total roasts: N
- Total roasting time: HH:MM
- Favorite bean type: "Ethiopia Yirgacheffe" (most roasted)
- Average rating: 7.2/10
- Most common defect: "Uneven"

**Roast list (searchable/filterable):**
- Sortable table: Date, Bean Type, Profile, Duration, Rating, Defects
- Click row to view detailed graph of that roast
- Filter by: date range, bean type, rating range, profile
- Search: bean type or notes

**Detailed roast view:**
- Graph overlay: target curve vs. actual curve (BT and ET)
- Stats panel: all logged data
- PID data: heater/fan adjustments over time
- User annotations: drop, 1st crack, end points marked on graph
- Edit rating/notes (can revise after-the-fact)
- Export: download as CSV or JSON

**Comparison view:**
- Select 2-3 roasts of same bean type
- Overlay their curves on same graph
- Compare ratings, defects, settings
- Help user identify patterns

### 6.4 Data Export & Backup

**CSV Export:**
- All roast metadata + average temperature per minute
- Columns: date, bean, profile, duration, temps, rating, defects, notes

**JSON Export:**
- Full raw data including every 200ms reading
- For external analysis or backup

**Backup:**
- User can download entire SQLite database file
- Can restore from backup if needed

---

## 7. SETTINGS & CONFIGURATION

### 7.1 Roast Profiles Editor

**User can edit:**
- **Light/Medium/Dark profiles:**
  - Phase 1 target temp
  - Phase 1 ramp rate (°C/min)
  - Phase 1 heater % range
  - Phase 1 fan % range
  - Phase 2 target temp
  - Phase 2 ramp rate
  - Phase 2 heater % range
  - Phase 2 fan % range
  - (same for phase 3)
  - Alert lead time before DROP/END (seconds)
  - Drum on/off
  - Cool settings (post-roast fan duration)

- **PID tuning parameters (advanced):**
  - Heater Kp, Ki, Kd
  - Fan Kp, Ki, Kd
  - Max heater overshoot tolerance
  - Max fan overshoot tolerance

- **Safety limits:**
  - Max temperature allowed (°C)
  - Command timeout (ms)
  - Serial connection retry count

### 7.2 Temperature Calibration Settings

- **Offset adjustment:** ±10°C manual adjustment (overrides learned offset)
- **Auto-calibration:** On/Off toggle
- **Calibration history:** View all learned offsets with timestamps and confidence
- **Reset calibration:** Clear learned offset for specific bean type

### 7.3 Serial Connection Settings

- **COM port:** Auto-detect or manual selection
- **Baud rate:** Fixed at 115200 (read-only, for info)
- **Connection test:** Button to verify connection (read one temp, display raw value)
- **Debug mode:** Show raw serial messages (optional)

### 7.4 Audio & Notifications

- **Alert volume:** 0-100%
- **Enable/disable sounds:** Toggle
- **Alert types:**
  - [ ] Pre-DROP alert
  - [ ] DROP alert
  - [ ] Pre-1st-crack alert (if user enabled)
  - [ ] Pre-END alert
  - [ ] END alert
- **Custom alert lead time:** e.g., "alert 10 seconds before DROP"

### 7.5 Preferences

- **Temperature display:** Celsius (°C) / Fahrenheit (°F)
- **Time format:** 12hr / 24hr
- **Graph theme:** Dark/Light mode
- **Data retention:** Auto-delete roasts older than X days (or keep forever)
- **Default profile:** Light (or user preference)

---

## 8. ERROR HANDLING & RECOVERY

### 8.1 Serial Communication Failures

**Scenario: ESP32 disconnected or USB cable unplugged**

**Behavior:**
1. Serial read fails
2. After 3 consecutive failures: show "CONNECTION LOST" warning
3. Disable heater commands (safety)
4. Show "Reconnect USB and click RECOVER"
5. User clicks RECOVER
6. Backend attempts to reconnect
7. If successful, resume; if not, show manual ESTOP button
8. Log disconnection event with timestamp

**Scenario: Garbled/checksum-failed data**

**Behavior:**
1. ESP32 can send bad checksums sometimes
2. First occurrence: log and ignore (retry next cycle)
3. If 5 consecutive failures: treat as communication error (see above)

### 8.2 Temperature Anomalies

**Scenario: Sudden temp spike (+20°C in one cycle)**

**Behavior:**
1. Detect outlier (> 3 std dev from recent history)
2. Reject reading, use last known good value
3. Log anomaly
4. If 3+ anomalies in 30 seconds: trigger WARNING state (not ESTOP)

**Scenario: Temperature stuck (same value for 30 seconds)**

**Behavior:**
1. Likely sensor failure
2. Show warning: "Temperature sensor may be stuck"
3. Allow user to ESTOP or continue (their choice)
4. Log event

### 8.3 PID Controller Issues

**Scenario: Temperature runaway (exceeds max temp)**

**Behavior:**
1. Trigger ESTOP automatically
2. Log emergency stop with reason
3. Alert user: "EMERGENCY STOP triggered - temperature exceeded limit"
4. Disable roast until user reviews

**Scenario: Heater not responding (temp not rising after 2 min)**

**Behavior:**
1. Check if heater is at 100% for 2+ minutes with no temp rise
2. Warn user: "Heater may not be responding"
3. Suggest: check physical connections, try manual mode
4. Allow user to force override or abort

### 8.4 Database Errors

**Scenario: Failed to save roast data**

**Behavior:**
1. Show warning to user
2. Keep data in memory (don't lose it)
3. Retry save every 5 seconds
4. If still failing after 1 minute, offer to export as JSON file instead
5. User can manually import later

---

## 9. DEVELOPMENT PHASES & TIMELINE

### Phase 1: Foundation & MVP (2 weeks)

**Backend:**
- [ ] FastAPI server setup (localhost:8000)
- [ ] Serial port communication (read/write to ESP32)
- [ ] Parse incoming temperature data
- [ ] Send basic control commands (heater, fan)
- [ ] Implement failsafe timeout logic
- [ ] SQLite database schema and basic queries
- [ ] Simple REST API for temperature readback

**Frontend:**
- [ ] React app setup (localhost:3000)
- [ ] WebSocket connection to backend
- [ ] Live temperature display (numeric)
- [ ] Manual heater/fan sliders
- [ ] Drum & cool toggles
- [ ] Emergency STOP button
- [ ] Basic roast start/end UI

**Testing:**
- [ ] Test serial communication with empty roaster
- [ ] Verify heater/fan commands sent correctly
- [ ] Test failsafe shutdown (unplug USB, verify auto-off)
- [ ] 1 dry run roast (no beans, safety check)

### Phase 2: PID & Real-Time Visualization (2 weeks)

**Backend:**
- [ ] Implement dual PID controllers (heater + fan)
- [ ] Adjust loop to run every 200ms
- [ ] Implement phase transitions (dry → dev → final)
- [ ] Logging of every 200ms data point
- [ ] Target curve generation for Light/Medium/Dark profiles
- [ ] Event marking (DROP, 1ST CRACK, END)

**Frontend:**
- [ ] Real-time temperature graph (Recharts)
- [ ] Overlay target curve on graph
- [ ] Status panel (current temp, target, heater %, fan %)
- [ ] Phase indicator (Drying / Development / Final)
- [ ] DROP/END buttons with styling
- [ ] Manual/Auto mode toggle
- [ ] Pre-roast setup form (bean type, profile, weight)
- [ ] Post-roast rating form

**Testing:**
- [ ] 3-5 light roasts with temperature logging
- [ ] Verify PID keeps temp close to target (±5°C)
- [ ] Check heater/fan response times
- [ ] Tune PID parameters based on behavior
- [ ] Test manual override
- [ ] Test DROP/END buttons work correctly

### Phase 3: Audio Alerts & Profile Management (1 week)

**Backend:**
- [ ] Profile storage/retrieval (Light/Medium/Dark + custom)
- [ ] Profile editor endpoints (update PID params, etc.)
- [ ] Roast history query endpoints
- [ ] Detailed roast retrieval (time series data)

**Frontend:**
- [ ] Profile selector dropdown (pre-roast)
- [ ] Profile editor form (edit Light/Medium/Dark)
- [ ] Audio alert system (Web Audio API)
- [ ] Alert configuration UI
- [ ] History dashboard with roast list
- [ ] Detailed roast view with graph overlay
- [ ] Roast comparison view (2-3 roasts)

**Testing:**
- [ ] Test audio alerts at correct times
- [ ] Create/edit custom profile
- [ ] Switch profiles mid-setup
- [ ] View detailed roast graphs
- [ ] Compare multiple roasts

### Phase 4: ML Calibration & Bean Profiles (2 weeks)

**Backend:**
- [ ] Temperature offset detection algorithm
- [ ] Bean profile learning (high-rated roasts analysis)
- [ ] Predictive roast timing
- [ ] Calibration confidence scoring
- [ ] ML model persistence to SQLite
- [ ] Refinement after each roast

**Frontend:**
- [ ] Calibration status display (offset, confidence)
- [ ] Manual offset adjustment slider
- [ ] Bean profile management (save/load custom profiles)
- [ ] ML suggestions display (heater/fan settings)
- [ ] Predictive END timing countdown
- [ ] Calibration history viewer

**Testing:**
- [ ] 5+ roasts of same bean, verify offset learned
- [ ] Manually adjust offset, verify applied
- [ ] Rotate bean types, confirm separate calibrations
- [ ] Check profile suggestions make sense
- [ ] Verify predictive timing improves with data

### Phase 5: Polish & Deployment (1 week)

**Backend:**
- [ ] Performance optimization (faster queries, smaller logs)
- [ ] Error handling edge cases
- [ ] Data export (CSV/JSON)
- [ ] Backup/restore functionality
- [ ] Comprehensive logging for debugging

**Frontend:**
- [ ] Dark/Light theme
- [ ] Responsive design (if accessing from phone in future)
- [ ] Help/documentation modals
- [ ] Settings panel (all preferences)
- [ ] Data export UI
- [ ] Loading spinners, error messages

**Deployment:**
- [ ] Electron wrapper for Mac app (optional)
- [ ] Or standalone localhost documentation
- [ ] Auto-start instructions
- [ ] Database reset/clear instructions
- [ ] Troubleshooting guide

**Testing:**
- [ ] Full 3-4 back-to-back roast session
- [ ] Test all error scenarios
- [ ] Verify data persists across app restarts
- [ ] Test export/backup/restore
- [ ] Performance under extended use

---

## 10. KEY TECHNICAL DECISIONS & TRADEOFFS

### 10.1 Why Python + FastAPI?

**Pros:**
- Excellent `pyserial` library, mature and stable
- Simple PID controller libraries (`simple-pid`)
- Fast enough for 200ms loop cycles
- ML libraries readily available (scikit-learn, TensorFlow Lite)
- Can run on MacOS/Linux/Windows without issues
- Easy to extend for future features (data analysis, etc.)

**Cons:**
- Requires Python installation (mitigated by bundling in Electron)
- Slightly slower than Go/Rust, but not critical here

### 10.2 Why React + TypeScript?

**Pros:**
- Fast, real-time updates with WebSocket
- Rich charting libraries (Recharts, Chart.js)
- Easy to build responsive UI
- Strong type safety with TypeScript

**Cons:**
- Larger bundle size (mitigated by code splitting)

### 10.3 Why SQLite?

**Pros:**
- Zero setup, file-based database
- Perfect for offline-first apps
- Can backup/restore easily (just copy file)
- Sufficient for your data volume (thousands of roasts)

**Cons:**
- Limited concurrency (not an issue for single user)
- Can migrate to PostgreSQL later if needed

### 10.4 Why Local ML Instead of Cloud?

**Pros:**
- Fully offline, no internet required
- Your roasting data stays private
- Instant feedback (no API latency)
- No cost

**Cons:**
- Limited to scikit-learn/TensorFlow Lite (good enough)
- No advanced NLP/LLM features initially (can add later)

### 10.5 200ms Loop Frequency

**Reasoning:**
- 200ms = 5 updates per second
- ESP32 can handle updates this fast
- Allows PID controller to respond within 1 second to temp changes
- Small roaster has slow thermal response (~10-20 second time constant), so 200ms is sufficient
- Faster (e.g., 100ms) would be overkill and drain CPU
- Slower (e.g., 500ms) might miss rapid changes

---

## 11. ADDITIONAL CONSIDERATIONS & FUTURE FEATURES

### 11.1 Now (MVP)

- Automated light roast with PID
- Manual controls with override
- Real-time graphing
- Roast history & ratings
- Temperature offset calibration

### 11.2 Later (v1.1+)

- **LLM-based roast notes:** "Based on your notes and rating, this roast had X development"
- **Image capture:** Optional USB camera to capture bean color during roast (post-processing)
- **Multi-user profiles:** Save different user preferences, PID settings
- **Cloud sync (optional):** Backup roasts to personal cloud for analysis on other devices
- **Mobile app:** Companion phone app to monitor roasts remotely
- **Batch scheduling:** "Roast every bean type in sequence" automation
- **Integration with recipe apps:** Import recipes from other roasting apps

### 11.3 Technical Debt to Plan For

- **Logging:** Implement proper logging framework (not just print statements)
- **Testing:** Unit tests for PID controller, ML algorithms, serial communication
- **Documentation:** API docs, deployment guide, troubleshooting
- **Refactoring:** Separate concerns (database, serial, PID) into clean modules

---

## 12. DEPLOYMENT & DAILY USE

### 12.1 Setup (One-time)

1. Clone repository
2. Install Python 3.10+
3. `pip install -r requirements.txt` (backend dependencies)
4. Install Node.js 18+
5. `npm install` (frontend dependencies)
6. Start backend: `python main.py` (runs on localhost:8000)
7. Start frontend: `npm start` (runs on localhost:3000)
8. Connect ESP32 via USB-C
9. Open browser to http://localhost:3000

### 12.2 Daily Use

1. Turn on roaster
2. Start app (backend + frontend running)
3. Select bean type and profile
4. Click "START ROAST"
5. Click DROP/END buttons when appropriate
6. Rate roast after completion
7. Repeat for next batch
8. Stop app when done

### 12.3 Ongoing Maintenance

- **Data cleanup:** Optionally delete very old roasts to free up disk space
- **PID tuning:** Adjust parameters if behavior changes (e.g., winter vs. summer ambient temp)
- **Profile refinement:** Save custom profiles for favorite beans
- **Backup:** Periodically download database backup file

---

## 13. SUMMARY OF KEY REQUIREMENTS

| Requirement | Solution |
|---|---|
| **Back-to-back 3-4 light roasts** | Pause/resume between roasts, profile management for quick setup |
| **Accurate automation** | Dual PID (heater + fan), 200ms loop, tight temp control ±3°C |
| **Light roast window (150→195°C)** | Three-phase profile: dry, dev, final; manual event marking |
| **Offline capable** | All processing local, SQLite database, no internet required |
| **Manual override anytime** | Toggle button switches to manual heater/fan sliders instantly |
| **Temperature offset learning** | After 3-5 roasts, ML detects consistent offset and applies auto-correction |
| **Roast history & analysis** | Full time-series logging, dashboard with filtering, comparison view |
| **Audio alerts for DROP/END** | Web Audio API, configurable lead time (5-10 seconds before event) |
| **Bean profile management** | Save/load custom profiles per bean type, editable PID parameters |
| **Profile suggestion** | After 5+ roasts of same bean, suggest heater/fan settings based on high-rated roasts |
| **Real-time graphing** | Target curve overlay with actual data, scrolling 12-minute window |
| **Roast rating & defects** | Post-roast form with 1-10 rating, checkboxes for defects, free notes |
| **Mac compatibility** | Pure Python + React, runs on MacBook, optional Electron wrapper |
| **Responsive failsafes** | Heater off on temp overshoot, auto-shutdown on serial loss, manual ESTOP button |
| **Data export** | CSV/JSON export, SQLite backup/restore |

---

## 14. FINAL NOTES

This project combines real-time control, machine learning, and user experience into a single cohesive application for optimizing your light roasting workflow. The key insight is that your roaster's constraints (small size, 8-12 min roasts, tight temperature window, consistent sensor offset) make it an excellent candidate for automation:

- **Small thermal mass** = fast response to heater/fan changes (good for PID)
- **Repeatable offsets** = easily learnable via ML
- **Consistent batch size** = easy to accumulate roast history
- **Short roasts** = quick iteration, can test changes daily

The app should grow smarter with every roast, learning your preferences and automatically suggesting settings. Start with the MVP (manual PID + graphing), then layer in ML calibration and profile learning.

Good luck with your roasting automation project!

