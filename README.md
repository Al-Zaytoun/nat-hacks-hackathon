# FireBusters

### Thermal vision meets autonomous fire suppression.

FireBusters is a hackathon robotics prototype that detects heat sources, decides which potential fire to address first, and positions a spray nozzle using an XY gantry. An overhead thermal camera provides a view of the workspace, while a carriage-mounted flame sensor helps distinguish flames from objects that are simply hot.

The project connects **computer vision, embedded firmware, motion control, and feedback-driven decision making** in one system. A Raspberry Pi handles perception and planning; an ESP32 controls the motors and spray trigger. A built-in simulator lets you explore the same control loop without assembling the hardware.

**Stack:** Python · OpenCV · NumPy · Raspberry Pi · ESP32 · Arduino/C++ · PlatformIO · I²C · USB serial

## At a glance

- **Detect and track:** Process 32 × 24 thermal frames, identify hot regions, and track their size and growth across frames.
- **Prioritize:** Rank targets by area and growth, with a travel-distance penalty to reduce unnecessary movement.
- **Aim:** Convert image coordinates to floor coordinates with a four-point perspective calibration, then translate them into gantry positions.
- **Confirm:** Check a separate flame sensor before spraying, and temporarily ignore rejected heat sources.
- **Respond and verify:** Sweep the nozzle over a target, monitor progress, reposition when needed, and check for remaining flame after spraying stops.
- **Observe:** View a live heat map, target overlays, nozzle position, and controller status in a browser or desktop window.

## Try it without hardware

The simulator replaces both the thermal camera and the ESP32 connection while using the application's detection, targeting, and state-machine code.

Install Python 3 with `pip` and virtual-environment support. Download or clone this repository, open a terminal in its root directory, and create a fresh environment.

**Windows PowerShell**

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
cd raspberry_pi
..\.venv\Scripts\python.exe main.py --sim --web
```

**macOS / Linux**

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
cd raspberry_pi
python main.py --sim --web
```

Open [localhost:8080](http://localhost:8080) to view the simulation. Stop it with `Ctrl+C`.

The scenario includes two initial fires with different growth rates, a hot mug, a person-temperature object, and another fire that appears after roughly 25 seconds. Extinguished simulated fires leave warm patches, allowing the controller to exercise its leftover-heat handling. These are synthetic scenarios, not measurements of real-world performance.

For a desktop preview, replace `--web` with `--show`. For a timed run without a viewer, use `--sim --duration 60`. The simulator does not require a calibration file; its initial rough-mapping notice is expected.

> `requirements.txt` includes NumPy, OpenCV, and pyserial. Adafruit hardware packages are conditionally included on `aarch64` and `armv7l` systems. A non-Pi ARM computer can run the simulator with just `python -m pip install numpy opencv-python pyserial`.

## System architecture

```text
MLX90640 thermal camera
         │ I²C
         ▼
Raspberry Pi 4 ───────────────────► Browser dashboard
  detection · tracking              heat map · status
  target selection · calibration
  suppression state machine
         │ USB serial: commands, heartbeat, telemetry
         ▼
       ESP32
         ├── A4988 drivers ──► X and Y stepper motors
         ├── Servo ──────────► Spray trigger
         └── Inputs ◄──────── Flame sensor, limit switches, e-stop
```

The Pi sends high-level commands rather than individual motor steps. The ESP32 handles stepper updates, homing, servo actuation, sensor sampling, and a command-timeout watchdog independently of the Python controller.

### Control loop

```text
SCAN → MOVE → CONFIRM → SUPPRESS → VERIFY → SCAN
```

| Stage | Behavior |
| --- | --- |
| **Scan** | Detect regions above both an absolute temperature threshold and an ambient-temperature margin. Require three observations before treating a track as a candidate. |
| **Move** | Select a target and map its image position to the gantry's millimeter coordinates. |
| **Confirm** | Read the downward-facing flame sensor. Reject unconfirmed heat sources for a configurable cooldown. |
| **Suppress** | Spray with a small circular sweep. Monitor heat and flame readings, reposition if progress stalls, and consider switching if another target becomes substantially more urgent. |
| **Verify** | Turn spraying off, wait, and check for remaining flame. Retry within configured limits or record the outcome and continue scanning. |

Initial target selection uses:

```text
priority = area × (1 + max(growth_rate, 0))
           ───────────────────────────────
           1 + distance_penalty × travel_cm
```

This is a threshold-and-tracking vision pipeline; it does not require a trained machine-learning model.

## Hardware

The prototype build described in the project notes uses the following components. Exact motor, servo, and mechanical dimensions must match the assembled gantry.

| Component | Purpose |
| --- | --- |
| Raspberry Pi 4, microSD card, and suitable power supply | Runs the Python controller and dashboard |
| MLX90640 thermal camera | Fixed thermal view of the workspace; the notes suggest the 110° wide-angle version |
| ESP32 development board and USB data cable | Runs firmware and exchanges commands and telemetry with the Pi |
| 2 stepper motors and 2 A4988 drivers | Move the X carriage and Y beam |
| Gantry frame, carriage, belts, pulleys, and Y drive shaft | Mechanical positioning system; firmware assumes GT2 belts and 20-tooth pulleys |
| Servo-actuated spray trigger, nozzle, and spray supply | Applies suppressant at the selected position |
| Analog flame-sensor module | Confirms flame below the carriage |
| 2 limit switches | Establish the X and Y home positions |
| Emergency-stop button | Provides an input to the firmware's stop logic |
| Motor power supply, wiring, and driver capacitors | Original notes specify a 3S 18650 pack and 100 µF across each driver's VMOT/GND |

Mount the nozzle and flame sensor on the carriage, pointing down. Mount the thermal camera on the frame with a clear view of the floor and minimal obstruction from the moving beam. A short shroud around the flame sensor narrows its field of view.

### Connections and board configuration

**Pi to MLX90640**, using physical header pin numbers:

| Camera | Raspberry Pi |
| --- | --- |
| VIN | Pin 1 — 3.3 V |
| GND | Pin 6 — ground |
| SDA | Pin 3 — I²C data |
| SCL | Pin 5 — I²C clock |

Connect the Pi to the ESP32 over USB. The firmware uses **115200 baud**.

The checked-in assignments in `esp32/main.cpp` are:

| Signal | GPIO |
| --- | --- |
| X STEP / DIR | 26 / 27 |
| Y STEP / DIR | 20 / 19 |
| Shared driver enable | 25 |
| X / Y limit switch | 32 / 33 |
| Emergency stop | 7 |
| Trigger servo | 21 |
| Flame sensor analog output | 34 |

**Review these assignments before wiring or uploading.** The PlatformIO target is `esp32dev`, but GPIO20 is package-specific and GPIO7 is generally associated with flash on classic ESP32 modules. Select pins supported by your actual board, update the firmware, and align the PlatformIO board target with that hardware. See [Espressif's GPIO reference](https://docs.espressif.com/projects/esp-idf/en/release-v4.4/esp32/api-reference/peripherals/gpio.html).

The firmware configures limit switches and the e-stop as active-low inputs with internal pull-ups. Driver settings assume 1/16 microstepping and 80 steps/mm. Set the A4988 current limits for the motors, use suitable power for the servo and motors, and keep logic grounds common. Sensor signals connected to the ESP32 must stay within the board's input-voltage limits.

## Hardware software setup

### 1. Prepare the Raspberry Pi

On Raspberry Pi OS, install the system dependencies:

```bash
sudo apt update
sudo apt install -y python3-venv python3-pip python3-opencv python3-numpy python3-serial i2c-tools git
sudo raspi-config
```

In `raspi-config`, enable **Interface Options → I2C**. Edit `/boot/firmware/config.txt` and ensure these settings are present, without adding conflicting duplicates:

```ini
dtparam=i2c_arm=on
dtparam=i2c_arm_baudrate=1000000
```

Older OS images may use `/boot/config.txt`. See the [Raspberry Pi configuration reference](https://www.raspberrypi.com/documentation/computers/config_txt.html) and [I²C parameter reference](https://github.com/raspberrypi/firmware/blob/master/boot/overlays/README).

Allow serial access and restart:

```bash
sudo usermod -aG dialout "$USER"
sudo reboot
```

After reconnecting, check for the camera:

```bash
i2cdetect -y 1
```

The MLX90640 should appear at address `33`. From the repository root, create the Pi environment:

```bash
python3 -m venv --system-site-packages .venv
source .venv/bin/activate
python -m pip install adafruit-circuitpython-mlx90640 adafruit-blinka
cd raspberry_pi
```

This setup reuses the system NumPy, OpenCV, and pyserial packages and installs the camera libraries in the virtual environment.

### 2. Build and upload the ESP32 firmware

Install [Visual Studio Code](https://code.visualstudio.com/) and the [PlatformIO IDE extension](https://platformio.org/install/ide?install=vscode). Open the repository's `esp32` folder as the PlatformIO project.

After checking the board target and pins:

1. Connect the ESP32 by USB.
2. Run **PlatformIO: Build**.
3. Run **PlatformIO: Upload**.
4. Open the serial monitor at **115200 baud** to check for `LOG ready` and `T ...` telemetry lines.

`platformio.ini` declares the Arduino framework, ESP32Servo, and AccelStepper dependencies. PlatformIO installs them during the build. Close the serial monitor before connecting from Python.

### 3. Calibrate and check motion

Run these commands from `raspberry_pi` with the Python environment active.

**Check the camera:**

```bash
python scanner.py --headless
```

Use a hot object to check detection. The scanner accepts `--flip-h` and `--flip-v`; persist any required orientation changes in `config.py` for the main application.

**Calibrate image-to-floor coordinates:**

```bash
python calibrate.py
```

Mark the four positions in `CALIBRATION_POINTS_CM` and follow the prompts with a hot object. This generates `raspberry_pi/calibration.json`. Choose points that suit the actual camera view and floor dimensions.

**Check gantry alignment:**

```bash
python calibrate.py --aim-test
```

This homes the gantry and opens an interactive prompt. Enter `g 0 0` for gantry coordinates in millimeters, or `15 15` for floor coordinates in centimeters. Enter `q` to exit. The Python connection maintains the heartbeat during these checks.

- Measure the floor position under the nozzle at home and set `HOME_FLOOR_X_CM` and `HOME_FLOOR_Y_CM`.
- Check commanded versus measured travel and adjust `X_STEPS_PER_MM` and `Y_STEPS_PER_MM` in the firmware.
- Set measured travel limits in **both** `esp32/main.cpp` and `raspberry_pi/config.py`; defaults are 295 mm on X and 230 mm on Y.
- Adjust direction inversion if an axis homes away from its switch, and tune the servo's trigger-on/off angles to the mechanism.

### 4. Run the controller

```bash
python main.py --web
```

Open `http://<pi-ip-address>:8080` from a device on the same network. The dashboard is intended for a trusted local network and has no authentication. `--show` instead opens a desktop window and requires a graphical display.

For thermal-only testing with a hot object:

```bash
python main.py --web --no-flame-confirm --fire-temp 40
```

This mode bypasses flame confirmation and can still move the gantry and activate spraying. Use it with the mechanism prepared for that behavior.

## Configuration and commands

Most application tuning lives in `raspberry_pi/config.py`; motor, servo, and pin settings live in `esp32/main.cpp`.

| Setting | Default | Purpose |
| --- | --- | --- |
| `FIRE_TEMP_C` / `AMBIENT_DELTA_C` | 60 °C / 15 °C | Absolute and above-ambient detection thresholds |
| `CONFIRM_FRAMES` | 3 | Observations required to confirm a thermal track |
| `REFRESH_HZ` | 8 | Thermal-camera refresh rate |
| `FLAME_THRESHOLD` / `FLAME_OUT_THRESHOLD` | 1500 / 800 | Flame confirmation and extinguished thresholds |
| `FALSE_POSITIVE_COOLDOWN_S` | 30 s | Time to ignore rejected targets |
| `SWEEP_MM` | 8 mm | Suppression sweep radius |
| `NO_PROGRESS_S` | 3 s | Time without progress before repositioning |
| `VERIFY_WAIT_S` / `MAX_ATTEMPTS` | 2 s / 3 | Verification delay and attempt limit |
| `HEARTBEAT_S` | 0.3 s | Pi-to-ESP32 heartbeat interval |

Useful command-line options:

```bash
python main.py --help
python main.py --sim --web --web-port 8081
python main.py --port /dev/ttyUSB0 --web --log-file run.log
```

The newline-delimited serial protocol supports `PING`, `HB`, `HOME`, `MOVE x y`, `SPRAY 0`, `SPRAY 1`, and `STOP`. Coordinates are in millimeters. Telemetry has the form:

```text
T flame x_mm y_mm moving spraying homed estop
```

The firmware turns spraying off and requests a motion stop after one second without a command. Manual serial commands also need a continuing heartbeat for sustained operation.

## Repository guide

```text
.
├── requirements.txt          # Python dependencies
├── esp32/
│   ├── platformio.ini        # Firmware build target and libraries
│   └── main.cpp              # Motors, servo, homing, sensors, serial protocol
└── raspberry_pi/
    ├── main.py               # Application entry point and visualization
    ├── config.py             # Detection, geometry, timing, and serial settings
    ├── scanner.py            # Thermal acquisition, detection, and tracking
    ├── targeting.py          # Coordinate transforms and target ranking
    ├── state_machine.py      # Detection-to-suppression control loop
    ├── motion.py             # ESP32 connection, commands, and heartbeat
    ├── sensor.py             # Telemetry parsing and flame-reading history
    ├── calibrate.py          # Camera calibration and interactive aiming
    ├── sim.py                # Synthetic thermal scene and simulated hardware
    └── web.py                # Browser heat-map stream and status dashboard
```

For a code walkthrough, start with `main.py`, follow the decisions in `state_machine.py`, then inspect `targeting.py` and the ESP32 firmware to see how a detected region becomes physical motion.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Camera not detected | I²C enablement, wiring, and address `33` in `i2cdetect` |
| ESP32 not responding | USB data cable, serial permissions, explicit `--port`, and whether another serial monitor is open |
| Gantry stops during manual testing | Firmware command timeout; use the Python aim test, which maintains a heartbeat |
| Nozzle consistently misses to one side | Camera calibration, home-floor offsets, and measured steps/mm |
| Hot objects trigger detection too easily | `FIRE_TEMP_C`, `AMBIENT_DELTA_C`, and `MIN_AREA` |
| Flame confirmation fails | Sensor orientation, analog polarity, and thresholds against observed telemetry |
| Browser view does not load | Host IP, network access, port availability, or `--web-port 8081` |
| Camera view is obstructed | Camera placement and `PARK_X_MM` / `PARK_Y_MM` |

## Prototype scope

FireBusters demonstrates an integrated perception-and-control workflow built during a hackathon. It is an experimental prototype, not a certified fire-protection system. The simulator is a way to inspect software behavior; it does not establish physical suppression effectiveness.

Hardware behavior depends on board-specific pin assignments, mechanical calibration, sensor tuning, and the spray mechanism. The e-stop and watchdog are firmware behaviors, not an independent power-isolation system. Dependency versions are not pinned in this snapshot.
