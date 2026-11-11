# Sequence of Operation: Robotic Label Applicator for Steel Coils

Vendor-neutral reference for any programmer, in any language. Terms: **Robot**, **PLC**, **HMI**, **Web Page**, **Printer**, **SCADA/DCS**.

## 1. Purpose and Scope

A coil stops at the labelling station. The system identifies the coil, detects its orientation, and the Robot picks an adhesive label from the Printer and applies it to the requested surface(s): **Outer (OD)**, **Side Face**, **Inner (ID/bore)**, in any combination or all three.

**Orientation definitions**

- **Eye-to-Wall:** coil axis horizontal. Side faces are vertical, bore faces sideways.
- **Eye-to-Sky:** coil axis vertical. Side faces are horizontal, bore faces upward.

**Architecture:** The PLC is the master. It owns the state machine, interlocks, request queue and fault handling. The Robot executes motion and tool actions on PLC command. The Printer prints and presents labels on PLC command. HMI, Web Page, Pendant and SCADA/DCS are command sources that only request actions through the PLC (except Pendant in Manual mode).

## 2. Equipment and Signals (Inventory)

| Item | Purpose |
| --- | --- |
| Coil presence sensor | Confirms coil is stopped in position |
| Coil position/orientation detection (vision or laser) | Detects Eye-to-Wall / Eye-to-Sky, coil ID/OD/width, bore centre |
| Printer | Prints label, presents it at a fixed pick point, reports status (ready, printing, label present, out of labels/ribbon, error) |
| Label presence sensor at pick point | Confirms label is available |
| Robot with vacuum/applicator tool | Picks and presses the label |
| Vacuum sensor | Confirms label held, confirms release |
| Label verification (optional camera/scanner) | Reads applied label, checks position/content |
| Safety devices | E-stop, light curtain or area scanner, guard door interlock |
| Beacon and buzzer | Status annunciation |

## 3. Control Authority

**3.1 Selector.** A single **Local/Remote** selector (hardwired or PLC-held) and a **Mode** selector decide who may command.

| Source | Allowed when | Can do |
| --- | --- | --- |
| Robot Pendant | Mode = Manual/Maintenance, Pendant enabled, deadman held | Jog, teach, test motions |
| HMI | Local | Full operation: start, stop, reset, mode, manual functions, requests |
| Web Page | Remote-Web enabled by HMI | Monitor, submit label requests, start/stop, reset (no manual jog) |
| SCADA/DCS | Remote-Plant enabled | Submit requests, start/stop, read status, remote reset (no manual jog) |

**3.2 Priority (high to low):** Safety > Pendant (Manual only) > HMI (Local) > SCADA/DCS > Web Page.

**3.3 Rules**

1. Only one source holds **Command Authority** at a time.
2. Authority changes only in **Idle** (no cycle running) or after a controlled stop.
3. Pendant in Manual mode removes authority from all other sources. Auto is blocked until the Pendant is returned to Auto and acknowledged on the HMI.
4. Stop and Abort are accepted from any source at any time (safe-side commands).
5. Every command is logged with source, user, time.
6. If the active remote source loses heartbeat (timeout 5 s), the PLC finishes or safely holds the current step, raises "Remote Comms Lost", and reverts authority to HMI.

## 4. Operating Modes

| Mode | Description |
| --- | --- |
| **Auto** | Full sequence runs on label requests |
| **Semi-Auto** | Runs one request, then waits for operator confirmation |
| **Manual** | Individual functions (move to pick, vacuum on/off, print test label, jog) from HMI or Pendant |
| **Maintenance** | Manual plus bypass of selected non-safety sensors, password protected, logged |
| **Dry Run** | Full motion with no label picked, for commissioning |

## 5. Label Request Data

Supplied by SCADA/DCS, Web Page or HMI:

| Field | Example |
| --- | --- |
| Request ID | 100245 |
| Coil ID / Heat no. | C-778123 |
| Surfaces required (bit mask) | Outer=1, Side=1, Inner=0 |
| Side face(s) | Face A, Face B, or both (Eye-to-Wall) / Top only (Eye-to-Sky) |
| Label template ID and variable data | Template 3, weight, grade, dimensions |
| Copies per surface | 1 |
| Priority | Normal |

The PLC validates the request. Rejects: no surface selected, unknown template, Coil ID missing, duplicate ID.

## 6. Readiness Conditions (Permissive to Start)

All must be TRUE: no active faults; E-stop healthy; guards closed and area clear; Robot in Auto, servo ready, at Home; Printer online with labels and ribbon; vacuum pressure OK; Auto mode selected; command authority assigned; no pendant in Manual.

## 7. Startup and Homing

1. Power-up: PLC checks I/O health and comms (Robot, Printer, vision, SCADA/DCS).
2. Operator or SCADA/DCS presses **Reset**. PLC clears resettable faults if the cause is gone.
3. **Homing:** Robot moves to Home through a safe path (PLC sends "Go Home"). Timeout 60 s.
4. Printer self-check, label feed calibration if requested.
5. Status becomes **Ready/Idle**.

## 8. Automatic Sequence (Main Flow)

**Step 0 – Idle.** Wait for a valid request and coil present.

**Step 1 – Request accepted.** PLC latches request, confirms Ready conditions. Beacon: steady green.

**Step 2 – Coil verification.** Coil presence sensor ON and stable for 2 s. Coil stopped signal from line received. Else wait; timeout 120 s → Fault F01.

**Step 3 – Coil identification and measurement.** Vision/laser measures coil position, OD, ID, width. Result compared with request data (tolerance configurable). Mismatch → Fault F05 (operator decides: continue with measured data, correct request, or abort).

**Step 4 – Orientation detection.** Vision/laser reports Eye-to-Wall or Eye-to-Sky. Not determined within 10 s → retry once → Fault F04.

**Step 5 – Build job list.** For each requested surface, create a job with orientation-specific Robot program number and target offsets. Order: **Inner → Side Face(s) → Outer** (reduces risk of tool collision with applied labels; configurable).

**Step 6 – Send coil geometry and job list to Robot.** Robot acknowledges data valid (timeout 5 s).

**Step 7 – Per-job loop** (repeat for each surface):

7.1 **Print label:** PLC sends template ID and data to Printer; Printer prints and presents label. Wait "Label at pick point". Timeout 20 s. Failure → retry up to 2 times → Fault F07.

7.2 **Check Robot clear:** Robot at Home or safe waypoint, safety area clear.

7.3 **Robot picks label:** Move to pick approach → pick position → vacuum ON → wait vacuum OK (timeout 2 s) → lift and retreat. If vacuum not OK: re-try pick up to 2 times, then discard label and reprint (counted); then Fault F08.

7.4 **Robot moves to surface** using the program for that surface and orientation (see Section 9). Speed reduced near coil.

7.5 **Apply label:** Contact detection (force/proximity or taught depth), press for dwell time (default 1 s), vacuum OFF, blow-off pulse, retract. Vacuum released not confirmed within 2 s → Fault F09.

7.6 **Verify (if fitted):** Scanner/camera reads label presence and content. Fail → one re-apply attempt allowed only after operator confirmation (the label cannot be re-picked), otherwise print a new label and apply beside it per operator choice. Persistent failure → Fault F10.

7.7 **Job done:** Mark surface complete, log result. Robot returns to safe waypoint. Next job, or go to Step 8.

**Step 8 – Cycle complete.** Robot returns Home. PLC reports result (Success / Partial / Failed with surface-wise status) to HMI, Web Page and SCADA/DCS. Handshake: "Labelling Done" to line. Wait for coil-released signal.

**Step 9 – Reset.** Clear request, wait coil leaves (timeout 300 s → warning). Return to Step 0.

## 9. Surface-Wise Application Details

| Surface | Eye-to-Wall (axis horizontal) | Eye-to-Sky (axis vertical) |
| --- | --- | --- |
| **Outer (OD)** | Label on the curved surface, tool normal to OD, placed on top or side per taught position. Robot compensates curvature using OD. | Label on vertical curved wall, tool horizontal, normal to OD. |
| **Side Face** | Face is vertical. Tool horizontal, approaches along the coil axis. Face A and Face B need separate approaches (Robot reach/space decides). | Face is horizontal. Tool vertical, approaches from above (Top face only). |
| **Inner (ID/bore)** | Tool enters the bore horizontally, labels the inner wall at a taught clock position. Requires bore diameter ≥ tool clearance, else Fault F11. | Tool enters bore from above, labels inner vertical wall using angled or side-reaching tool pose. |

Common rules: positions are calculated from measured geometry plus taught offsets; every approach uses a clear approach point; label is never placed over a weld or strap zone defined in the recipe; the Robot slows down within 150 mm of the surface.

## 10. Timeouts and Retries

| Event | Timeout | Retry | On failure |
| --- | --- | --- | --- |
| Coil arrival | 120 s | 0 | F01 |
| Vision result | 10 s | 1 | F04 |
| Robot data acknowledge | 5 s | 1 | F03 |
| Print and present label | 20 s | 2 | F07 |
| Vacuum on pick | 2 s | 2 | F08 |
| Robot move (each) | Based on program, max 60 s | 0 | F02 |
| Vacuum release | 2 s | 1 | F09 |
| Label verification | 5 s | 1 | F10 |
| Remote heartbeat | 5 s | 0 | F12 |
| Homing | 60 s | 1 | F13 |

All values are configurable from the HMI (engineer level).

## 11. Fault List and Recovery

| Code | Fault | Cause | Recovery |
| --- | --- | --- | --- |
| F01 | Coil not arrived | Line delay | Wait / Retry / Abort |
| F02 | Robot motion timeout or error | Collision, servo fault | Clear obstruction, Robot reset, Resume from last safe point or Home |
| F03 | Robot comms/data fault | Network, program mismatch | Check link, resend data, Retry |
| F04 | Orientation not detected | Vision issue, dirty lens, coil shifted | Clean, re-detect, or operator selects orientation manually (authorised) |
| F05 | Coil data mismatch | Wrong coil or request | Correct data, Continue, or Abort |
| F06 | Safety trip (E-stop/guard/scanner) | Person in zone | Clear zone, reset safety, Reset on HMI, Robot goes Home before Auto resumes |
| F07 | Printer fault | Out of labels/ribbon, jam, offline | Reload, clear jam, Retry print |
| F08 | Label pick failure | Vacuum low, label curled | Check tool and label, Retry |
| F09 | Label not released | Adhesive, vacuum valve | Manual release, inspect tool |
| F10 | Label verification failed | Skew, missing, unreadable | Operator inspects, Accept / Reapply / Skip |
| F11 | Bore too small / surface unreachable | Coil dimension out of range | Skip surface with operator OK, or Abort |
| F12 | Remote comms lost | SCADA/DCS or Web Page heartbeat | Authority returns to HMI, re-establish link |
| F13 | Homing failed | Robot blocked | Manual jog to Home from Pendant |
| F14 | Air/vacuum supply low | Compressor | Restore supply |
| F15 | Request invalid | Bad data | Reject and report reason to the source |

**Recovery principles**

1. Faults latch until the cause is cleared **and** Reset is pressed.
2. After any stop, the PLC offers: **Resume** (only if Robot position and label state are safe), **Restart Job**, **Skip Surface**, **Abort Cycle**.
3. A label held by the tool at the time of a stop is flagged "label in hand". Recovery must discard it to the reject bin or apply it, never ignore it.
4. On Abort: Robot goes to Home via safe path, tool vacuum off, request marked Failed or Partial.
5. Alarm history stores time, code, step, source, and operator action.

## 12. Operator Functions

**HMI screens:** Overview (status, step, coil, orientation, surfaces done); Request entry; Manual control; Alarms and recovery; Settings (timeouts, offsets, label templates); Diagnostics (I/O, comms); Logs and counters.

**Operator actions available**

- Select mode and control source; start, pause, stop, abort, reset
- Enter or edit label request; choose surfaces by tap (Outer / Side A / Side B / Inner / All)
- Re-print last label; print test label
- Skip surface; reapply; accept label manually
- Manual vision orientation override (with authority)
- Robot Home, move to pick point, move to service position (for label roll change)
- Acknowledge alarms with guided recovery text (what happened, what to check, what to press)
- Label and ribbon low warnings before empty

**Web Page:** read-only live status plus request submission and start/stop if enabled; shows the same alarm text as HMI.

**Beacon:** Green steady = ready, green flashing = running, amber = warning, red = fault, blue = waiting for operator.

## 13. Handshake Signals (PLC ↔ Devices)

| Direction | Signal | Meaning |
| --- | --- | --- |
| PLC → Robot | Start, Program number, Surface ID, Orientation, Coil geometry | Job command |
| PLC → Robot | Stop, Hold, Go Home, Fault reset | Control |
| Robot → PLC | Ready, Running, At Home, Job Done, Fault, Label in hand, Step number | Status |
| PLC → Printer | Print request, Template ID, Data, Quantity | Print job |
| Printer → PLC | Ready, Printing, Label at pick point, Out of labels/ribbon, Error | Status |
| PLC → SCADA/DCS | Ready, Busy, Done, Result, Alarm, Heartbeat | Reporting |
| SCADA/DCS → PLC | Request data, Start, Abort, Reset, Heartbeat | Commands |

Handshake rule: every command is held until the receiver's acknowledge is seen, then dropped; the receiver drops its acknowledge after the command drops (4-way handshake). Each wait has a timeout.

## 14. Safety and Interlocks

- Safety circuits are hardware-based and independent of PLC logic; PLC only reads status.
- Robot cannot move in Auto if guard open, area scanner tripped, or E-stop active.
- Pendant deadman and reduced speed apply in Manual.
- Printer cutter/mechanism guarded.
- No remote commands that cause motion are accepted unless area-clear is confirmed.
- Passwords and user levels: Operator, Maintenance, Engineer.

## 15. Main State List (for programming)

| State | Entry | Exit |
| --- | --- | --- |
| INIT | Power on | Checks pass |
| HOMING | Reset / start | Robot at Home |
| IDLE | Ready | Request valid |
| WAIT\_COIL | Request accepted | Coil stable |
| MEASURE | Coil stable | Data and orientation OK |
| PRINT | Next job | Label at pick point |
| PICK | Label ready | Vacuum OK |
| APPLY | Label held | Label released |
| VERIFY | Label applied | Pass, or operator decision |
| NEXT\_JOB / COMPLETE | Job done | Jobs remain or finished |
| PAUSED | Stop/Hold | Resume |
| FAULT | Any fault | Reset and cause cleared |
| ABORT | Abort | Robot Home |

Any state can go to FAULT, PAUSED or ABORT.

## 16. Assumptions

1. The PLC is the single master and holds the state machine.
2. The line sends a "coil stopped" signal and receives "labelling done".
3. The Printer presents labels at a fixed pick point.
4. Label size and tool design suit all three surfaces; one tool is used.
5. Surface order is Inner, then Side, then Outer (configurable).
6. Timeout values are starting points to be tuned at commissioning.
7. A reject bin is provided for discarded labels.