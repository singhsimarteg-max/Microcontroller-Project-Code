# Single-scan automatic layout selection

Open **AutonomousKobuki Auto Layout 3** as the complete VS Code project.
Build the Debug preset, then launch **Debug NUCLEO-G474RE**. The output remains
`build/Debug/AutonomousKobuki.elf`; the included launch.json is unchanged.
Continue if the debugger pauses at main. The robot waits for B0.

## Buttons and sequence

- **B0:** start a new run. **B1:** unused. **B2:** stop from every state.
- Scan once through 360 degrees counterclockwise. Find the nearest object's
  range and the midpoint of its two detected range edges.
- Select the layout using the nearest valid target distance:

| Nearest distance | Result |
| --- | --- |
| 500 mm or above | Layout A |
| 450 mm or below | Layout B |
| 451–499 mm | Stop in FAULT; no layout selected |

Invalid readings or a scan without a bounded target also stop. The sensor
cannot distinguish distances below its 300 mm minimum; a maximum-range
reading by itself is not treated as a detected target.

- Face the nearest object's midpoint, then execute the selected route below.
- At the end, perform one more 360-degree scan, face the new nearest object's
  midpoint, advance `FINAL_APPROACH_MM` (default 10 mm), and stop in IDLE.
  This final scan does not choose another layout.
- Release buttons before pressing B0 for another run.

| Route | Movements after facing the starting obstacle |
| --- | --- |
| A | CW 90 degrees; drive 1195 mm; CCW 90 degrees; drive 1395 mm; CCW 139.4 degrees; drive 918 mm |
| B | CCW 90 degrees; drive 1395 mm; CW 90 degrees; drive 1195 mm; CW 139.5 degrees; drive 918 mm |

**All distances are millimetres.** Each route's three planned straight legs
sum to 3508 mm. Controller stopping tolerance and recovery affect actual travel.
The speed cap remains 150 mm/s; the existing distance controller peaks at
120 mm/s. Longer legs use short distance setpoints to handle encoder rollover.

## Change settings in one place

Edit `Inc/nav_config.h`, then rebuild and upload:

| Setting | Meaning |
| --- | --- |
| `LAYOUT_A_MIN_MM` | This nearest distance or above selects A; default 500 mm |
| `LAYOUT_B_MAX_MM` | This nearest distance or below selects B; default 450 mm |
| `ROUTE_SHORT_LEG_MM` | A first leg / B second leg; default 1195 mm |
| `ROUTE_LONG_LEG_MM` | A second leg / B first leg; default 1395 mm |
| `ROUTE_FINAL_LEG_MM` | Both layouts' last leg; default 918 mm |
| `FINAL_APPROACH_MM` | Extra forward move after final alignment; default 10 mm; 0 disables it |
| `ROUTE_A_FIRST_TURN_DEG` | A departure turn; default -90 degrees |
| `ROUTE_A_SECOND_TURN_DEG` | A turn after first leg; default +90 degrees |
| `ROUTE_A_FINAL_TURN_DEG` | A turn after second leg; default +139.4 degrees |
| `ROUTE_B_FIRST_TURN_DEG` | B departure turn; default +90 degrees |
| `ROUTE_B_SECOND_TURN_DEG` | B turn after first leg; default -90 degrees |
| `ROUTE_B_FINAL_TURN_DEG` | B turn after second leg; default -139.5 degrees |
| `ROUTE_DISTANCE_SCALE`, `ROUTE_ANGLE_SCALE` | Scale all route distances or all route angles with one edit |
| `BACKOFF_MM`, `TURN_AWAY_DEG` | Recovery: 100 mm backwards, 10 degrees away |
| `ALIGN_EDGE_RISE_MM` | End the target span when range exceeds its minimum by 150 mm |

**Positive route angle = counterclockwise/left. Negative = clockwise/right.**
For example, change `ROUTE_A_FIRST_TURN_DEG` from `(-90.0f)` to `(+60.0f)`
to turn left 60 degrees after alignment instead of right 90 degrees.
Use turns smaller than 180 degrees in magnitude to give an unambiguous shortest
turn direction. The initial/final scans remain full counterclockwise scans;
alignment automatically turns to the measured midpoint.

Keep `LAYOUT_B_MAX_MM` below `LAYOUT_A_MIN_MM`. A compile-time check catches
accidentally overlapping thresholds. The three-sample median remains in the
scan to reject isolated bad readings. Other labelled scan/sensor/timeouts are
in the same configuration file.

The final advance uses the existing encoder-distance controller and its few-mm
stopping tolerance, so 10 mm is a nominal target. Its setting is independent of
the route distance scale and accepts 0–2000 mm. The existing drive timeout and
global safety checks apply. A side-hazard recovery during this final move uses
the existing alignment recovery: back off, turn away, scan and align again,
then attempt the short final move from the new position.

## Safety and recovery

- B2, wheel drop or **centre cliff** immediately stop to IDLE.
- A side cliff or bumper stops the robot, reverses about 100 mm, turns about
  10 degrees away and resumes. Left triggers a right turn; right triggers left.
  Centre/both bumpers default to a left turn.
- Recovery during scanning or alignment restarts one scan because the robot
  has moved. Once a layout is selected it stays selected for that run; the
  changed distance after backing up does not reclassify it.
- Recovery during driving retains progress and adds the reversed distance
  back to the remaining distance. Turning does not count as straight travel.
- Missing feedback, persistent hazards, scan gaps, an unbounded target,
  a distance in the initial selection gap or a timeout stop in FAULT (both LEDs red).

Press B2 to acknowledge a fault, clear its cause, then release and press B0.
Before deliberately pausing the debugger, press B2 and confirm the robot has
stopped: pausing the MCU does not transmit a stop command to the Kobuki.

## Code and debugger

The application has two C files and two headers:

| File | Purpose |
| --- | --- |
| `Src/main.c` | Initialisation, read/decide/act loop, FSM, routes and recovery |
| `Inc/nav_config.h` | Adjustable measurements, thresholds and timeouts |
| `Src/scan.c` | Median, scan bins, edge midpoint and distance-based layout choice |
| `Inc/scan.h` | Scan helper declarations and structures |

The UART, ADC, ultrasonic and Kobuki drivers are unchanged. CMSIS, startup,
linker and build files are required for the STM32 project. Optional PC tests
in `tests/` are not compiled into the robot firmware.

Normal sequence: IDLE -> SEARCH -> ORIENT -> PATHFIND -> SEARCH -> ORIENT -> IDLE.
The final ORIENT includes `ORIENT_FINAL_DRIVE` after alignment for the short advance.
RECOVER handles side hazards; FAULT holds stopped until B2.

Watch these fields under `nav_debug`:

- `layout_distance_mm`: initial nearest measurement used for layout choice.
- `layout`: 0 = none, 1 = A, 2 = B.
- `alignment.target`: nearest range, both edges and midpoint.
- `state`, `substate`, `route.remaining_mm`, `fault`, `stop_reason`.
- Fault 17 means the initial range fell in the gap; fault 12 means no bounded
  nearest object was found. The old `identification` fields have been removed.

## Checks and physical limits

Run PC checks with `sh tests/run_tests.sh`. These exercise distance thresholds,
both complete routes, single scans, midpoint calculation, encoder wraparound,
safety priority, recovery and timeouts. Debug and Release STM32 builds and the
PC checks pass. This version has not been tested on the robot here.

Layout selection assumes the actual nearest ultrasonic return distinguishes
the layouts. Check the measured values on the course and tune the two
thresholds if needed. Any bounded nearest target at 500 mm or more now selects
A, even if much farther away than 550 mm. The sensor's wide beam, filtering
delay and 300 mm minimum reading still limit scan and edge accuracy.

Recovery can change the endpoint. This version retains bumper/cliff recovery;
the added final forward move is measured by the encoder. The source before
adding that move is saved in
`../checkpoints/AutonomousKobuki-auto-3-before-final-approach.zip`.
