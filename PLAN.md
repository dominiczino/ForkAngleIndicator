# Tractor Fork Angle Indicator — Implementation Plan

Source idea (Todoist › Ideas): *"Dashboard indicator that uses a multiaxis accelerometer to indicate
the angle of the front loader forks compared to level. Also add the ability to measure the
differential angle compared to a wireless reference unit (placed on a truck bed or something)."*

Hardware on hand: 2× WitMotion SINAT dual-axis inclinometer (IP68, accel + gyro fused).

---

## 1. Requirements

| # | Requirement | Target |
|---|---|---|
| R1 | Fork pitch (tip up/down) vs. gravity, shown in cab | ±0.3° static, 0.1° resolution |
| R2 | Fork roll (side tilt) vs. gravity | ±0.3°, secondary display |
| R3 | Differential mode: fork plane vs. remote reference plane (truck bed, trailer, rack) | ±0.5° |
| R4 | Update/latency | ≥10 Hz display, <200 ms latency |
| R5 | Readable in direct sun, operable with gloves | — |
| R6 | Survives tractor electrical (12 V nominal, 14.4 V charging, cranking dip, load dump) | — |
| R7 | Survives vibration, wash-down, impacts on the loader | sensor IP68, cable/connector IP67 |
| R8 | Wireless reference range | ≥50 m line of sight |
| R9 | Forks only (no bucket mode) | single stored fork zero offset |
| R10 | Boot to reading | <3 s from key-on |
| R11 | Reference unit battery-powered; **forced hard-off while in its tractor holster** (and charges there) | 0 µA drain from logic when docked |
| R12 | Reference unit auto-off if left on out of the holster | configurable, default 30 min idle |

Decided: SSQA (skid-steer quick-attach) loader, forks only, 12 V hard-wired, reference on battery.
Pending: confirm SINAT label reads RS485.

---

## 2. Sensor facts (WitMotion SINAT) — verify on bench in Phase 0

From retailer listings (manufacturer page unreachable from where this plan was written):

- Supply 5–36 V (one listing says 9–36 V — power it from 12 V and both are satisfied), <25 mA.
- Interface: **RS485, Modbus RTU** (SINAT-485). Baud 4800–230400, default typically 9600, address 0x50.
  Check your label: if yours is the RS232 or TTL variant the transceiver choice in §4 changes.
- Range X ±180°, Y ±90°; accuracy 0.1–0.2°; resolution ~0.0055°/LSB.
- Wire colours (typical WitMotion): red VCC, black GND, yellow/green A/B — **confirm before power-up**.
- WitMotion standard Modbus map (expected, confirm): 0x34–0x36 accel, 0x37–0x39 gyro,
  0x3D–0x3F roll/pitch/yaw, scale `value/32768 × 180°` (signed int16). Bandwidth/filter and
  output-rate registers exist; set them in Phase 0.

---

## 3. System architecture

```
 FORK SENSOR (SINAT #1)                     DASH UNIT (cab)                    REFERENCE UNIT (portable)
 on loader SSQA plate                       ESP32-S3 + display                 SINAT #2 + ESP32 + battery
 ┌─────────────┐  RS485, 4-core shielded   ┌──────────────────────┐  ESP-NOW  ┌──────────────────────┐
 │  SINAT-485  │──────────────────────────▶│ RS485 xcvr → UART    │◀ ─ ─ ─ ─ ─│ ESP32 ← RS485 ← SINAT│
 │  12 V in    │  ~5–8 m, M12 at loader    │ 2.0–2.8" IPS TFT     │  2.4 GHz  │ 2×18650, BQ24074     │
 └─────────────┘  disconnect               │ 2 buttons, buzzer    │           │ magnet base, arrow   │
                                           │ 12 V in, 5 V→holster │◀── pogo ──│ docked: off + charge │
                                           └──────────────────────┘           └──────────────────────┘
```

Key decisions:

1. **Fork sensor wired, not wireless.** Power is available, RS485 is robust over long cable,
   no battery on the loader, no RF dropout behind steel. One cable along the existing hose route.
2. **Mount the sensor on the loader's SSQA plate (tractor side), not on the fork frame.** It stays put
   when the forks come off, so no connector at the attachment, and the SSQA wedge/latch makes the
   plate→tine angle repeatable. That angle is one fixed offset, zeroed once (§6). Mounting/bracket
   is yours; the only requirements are rigid, X axis along the tractor, and guarded from pallets.
3. **ESP32-S3 running MicroPython** for both dash and reference units. Instant boot, no SD card to
   corrupt at key-off (a Raspberry Pi is the wrong tool here), native ESP-NOW for the wireless link,
   and it stays in Python.
4. **ESP-NOW** for the reference link: connectionless, no router, ~1–5 ms latency, 100 m+ LOS.
   Optional long-range mode (`WIFI_PROTOCOL_LR`) if range is short.
5. Display: high-brightness IPS TFT (ST7789/ILI9341, ≥600 nit if you can find it) in a sun hood.
   Fallback if sunlight readability is poor: 4-digit 7-seg LED + WS2812 LED bar.

---

## 4. Bill of materials (estimate, excluding the two SINATs you have)

| Item | Qty | Notes | ~USD |
|---|---|---|---|
| ESP32-S3 dev board (e.g. DevKitC-1, or LilyGO T-Display-S3 w/ built-in 1.9" screen) | 2 | dash + ref | 25 |
| RS485 transceiver module, 3.3 V, auto-direction (MAX13487 / SP3485 + auto-DE) | 2 | | 8 |
| 2.0–2.8" IPS TFT, ST7789/ILI9341, SPI | 1 | skip if T-Display-S3 | 12 |
| Automotive 12 V→5 V buck, 6–40 V in, ≥1 A (e.g. Pololu D36V6F5 / D24V10F5) | 2 | | 20 |
| Protection: 1N5822 or ideal-diode reverse protect, SMBJ33A TVS, 1 A blade fuse + holder | 2 sets | load dump | 10 |
| 4-core shielded PUR cable, oil/UV resistant, 22–24 AWG | 8 m | | 25 |
| M12 4-pin A-coded connector pair (panel + cable), IP67 | 2 | loader detach point + dash | 25 |
| Cable glands, spiral wrap / hose sleeve, P-clips, zip ties | — | | 15 |
| Steel guard for fork sensor (bent 3 mm plate) + M5 stainless bolts | 1 | | 10 |
| Momentary buttons IP67 ×2 (or rotary encoder w/ push), piezo buzzer | 1 set | | 10 |
| Dash enclosure (printed ASA/PETG) + RAM ball mount | 1 | | 20 |
| Reference: 2× 18650 (1S2P) + holder | 1 | ~6 Ah | 15 |
| Reference: Li-ion charger with power-path + thermistor input (TI BQ24074 breakout, e.g. Adafruit #4755) | 1 | blocks charging <0 °C / >45 °C in a cab | 15 |
| Reference: soft-latching power switch with OFF input (Pololu Mini Pushbutton Power Switch LV, #2808) | 1 | holster + firmware hard-off, see §8.1 | 10 |
| Reference: 3.3→5 V or 3.7→5 V boost for SINAT (or 5→12 V if your SINAT needs 9 V min) | 1 | check Phase 0 | 5 |
| Reference: IP67 momentary button, bi-colour LED | 1 | | 8 |
| Reference: 2-pin magnetic pogo connector pair (panel-mount, IP67-ish) | 1 | holster charge contact | 10 |
| Reference: enclosure, 4× pot magnets (≥10 kg pull each), flat base plate | 1 | | 25 |
| Holster: steel/aluminium cradle, M8 cable to dash 5 V, 500 mA polyfuse | 1 | | 15 |
| USB-RS485 dongle for PC bench work | 1 | | 10 |
| **Total** | | | **~300** |

---

## 5. Measurement math

### 5.1 Absolute fork angle
The SINAT outputs fused roll/pitch directly. Fork pitch shown to operator:

```
θ_fork = θ_sensor − θ_offset
```

Sign convention to fix once: **positive = tips up** (rolled back). Display with an arrow, not just a sign.

### 5.2 Dynamic acceleration error
A pure accelerometer tilt reading is wrong by `atan(a_long / g)` under acceleration:
0.1 g braking → 5.7° error. The SINAT's gyro fusion suppresses short transients, but a sustained
acceleration still leaks in. The task (approaching a pallet or a bed) is done slow/stationary, so:

- Compute `|a|` from accel registers; if `| |a| − 1 g | > 0.03 g` or `|ω| > 5°/s`, flag
  reading as **unsteady** (dim/grey the number) instead of trying to correct it.
- Tune the sensor's internal filter bandwidth in Phase 0 (trade latency vs. engine vibration noise).
- On the ESP32, a light EMA (τ ≈ 0.2 s) on top. Do not over-filter — the operator is closing a loop
  through the curl valve.

### 5.3 Differential mode (fork vs. reference plane)
Each unit only knows its tilt w.r.t. gravity, so heading between the two is unobservable.
Resolve by convention: **the reference unit has an arrow; the operator places it pointing in the
direction the forks will enter** (default ψ = 0). Allow ψ ∈ {0, 90, 180, 270}° from a menu for
side loading.

Small-angle form (tilt as a 2D vector, error ~ θ·φ, <0.1° for angles under ~10°):

```
t_ref' = R(ψ) · [θ_ref, φ_ref]ᵀ
Δ = [θ_fork, φ_fork]ᵀ − t_ref'
```

Exact form (use this; it is cheap): build gravity unit vectors `g_f`, `g_r` from each unit's
roll/pitch, rotate `g_r` by ψ about z, then the relative tilt is the rotation taking `g_r'` to `g_f`;
report its components about the fork's lateral (pitch) and longitudinal (roll) axes.

Display Δpitch big, Δroll small. Green band ±1° (configurable).

Truck on its suspension and a tractor on soft ground both move; differential mode makes that
irrelevant — that is the point of it.

### 5.4 Error budget (differential, static)

| Source | Magnitude | Mitigation |
|---|---|---|
| Sensor accuracy ×2 | 2 × 0.1–0.2° | cross-zero units together (§6.1) |
| Ref unit base seating on dirty/ribbed bed | 0.5°+ | flat 3-point or wide base, wipe bed |
| Tine deflection under load | 0.5–2° at rated load | note only; sensor sees the plate, not tine tips |
| Loader linkage slop | 0.2–0.5° | zero with typical preload |
| Heading misalignment ψ error 10° on a 5° bed | ~0.1° | negligible |

---

## 6. Calibration

1. **Cross-zero the pair** (removes inter-sensor bias, most important for differential accuracy):
   put both SINATs on the same surface-plate/flat steel, record difference in pitch and roll,
   rotate both 180° and repeat (separates surface tilt from sensor bias). Store bias in the
   reference unit's config.
2. **Fork zero offset**: lay the reference unit on a tine, lower forks flat onto the floor (light
   down-pressure to take out linkage slop), `θ_offset = θ_sensor − θ_ref`. This references the
   tine directly, so the floor need not be level. Roll offset the same way.
3. **Quick re-zero** (long-press, with confirm): same as 2 but trusts "forks are level now";
   for when the forks are re-hung and seat slightly differently.
4. All config in a JSON file on the ESP32 flash; written only on explicit save (flash wear, power cut).

---

## 7. Firmware design (MicroPython, ESP32-S3)

```
firmware/
  common/
    modbus_rtu.py      # minimal master: fn 0x03 read, 0x06 write, CRC16, timeout/retry
    sinat.py           # register map, scaling, config (baud, rate, bandwidth), read_tilt()
    tiltmath.py        # gravity vectors, rotate ψ, relative tilt, unsteady detect, EMA
    proto.py           # ESP-NOW packet pack/unpack (struct), seq, CRC
  dash/
    main.py            # asyncio: sensor task 20 Hz, radio rx task, UI task 10 Hz, button task
    ui.py              # screens: ABS, DIFF, CAL, SETTINGS; large font render
    config.py          # fork offset, cross-zero bias, ψ, thresholds, load/save JSON
  ref/
    main.py            # read sensor 20 Hz → ESP-NOW; battery sense; idle timer → assert OFF pin (§8.1)
tools/ (CPython, run on PC)
  sinat_bench.py       # pyserial/minimalmodbus: read, configure, log CSV
  analyze.py           # noise PSD, Allan deviation, step response from logs (numpy/matplotlib)
tests/                 # pytest for tiltmath, modbus CRC, proto pack/unpack (pure Python, run on PC)
```

ESP-NOW packet (ref → dash, 20 Hz, broadcast to paired MAC):
`<u8 ver, u16 seq, u32 t_ms, i16 pitch×100, i16 roll×100, i16 |a|×1000, u8 flags, u8 batt%>`
Dash marks reference **stale** after 500 ms without packets and blanks the Δ readout.

UI:
- **ABS screen**: big pitch number with ▲/▼ arrow, horizontal bar ±15°, small roll.
- **DIFF screen**: big Δpitch, Δroll, link RSSI/age, ref battery, ψ indicator.
- Unsteady → number goes grey. Within target band → green + optional short beep.
- Buttons: short A = cycle ABS/DIFF, short B = cycle ψ (DIFF screen), long B = quick-zero,
  long A+B = settings/calibration.
- Dash shows "REF DOCKED" / "REF OFF" / ref battery % so you know its state without looking at it.

Reference-unit packet gets a `flags` bit for "shutting down (idle/low batt)" so the dash can
say why the link dropped.

---

## 8. Electrical / mechanical details

- **Power (dash):** fused switched-ignition 12 V → reverse-polarity protection → TVS (SMBJ33A) →
  buck 5 V → ESP32. SINAT fed from the protected 12 V rail directly (it accepts up to 36 V).
  Cold-crank brownout is acceptable; firmware must recover cleanly (nothing written on boot).
- **RS485:** A/B twisted pair + 12 V + GND in one shielded cable; shield grounded at dash end only;
  120 Ω termination at both ends (often on the modules, check). Keep sensor and dash at same
  baud; 115200 is plenty for 20 Hz polling.
- **Cable route:** along loader hose bundle with spiral wrap, P-clips at every hose clamp,
  service loops at every pivot (boom pivot and SSQA/curl pivot) sized for full range of motion.
  M12 disconnect at the loader-to-tractor parking point so the loader can be dropped.
- **Sensor mount:** your bracket on the SSQA plate; X axis along tractor longitudinal axis
  (a few degrees of yaw misalignment only couples ~sin(err) of roll into pitch — negligible).
  Guard it; thread-locker on bolts.
- **Dash unit:** RAM ball on ROPS post or fender; sun hood; IP65 print (ASA, gasketed lid,
  cable gland).
- **Reference unit:** flat base (aluminium plate with three hard-anodised feet or full flat face),
  magnets recessed so the feet/face carry the contact, large printed arrow, one push button,
  bi-colour status LED, charge contacts on a **side** face (not the base, so a truck bed can't
  touch them).

### 8.1 Reference unit power and holster interlock

```
 2×18650 ─┬─ BQ24074 (charger + power path, NTC) ── SYS ──▶ Pololu pushbutton switch ──▶ boost 5 V ─▶ ESP32 + SINAT
          │        ▲ IN                                     ▲ A (button)   ▲ OFF
          │        │                                        │              ├── VDOCK via divider  (holster: forced off)
 holster  │        │                                    push button        └── ESP32 GPIO        (idle / low-batt off)
 pogo ────┴── VDOCK (5 V from dash buck, polyfuse)
```

- **Holster = forced off + charging.** Docked, the pogo pins deliver 5 V (`VDOCK`). It feeds the
  charger and also drives the power switch's OFF input. While docked the switch is held off in
  hardware: button presses do nothing, ESP32 and SINAT draw zero, battery charges whenever the
  ignition is on. No firmware involved, so a crash can't defeat it.
- **Leaving the holster:** stays off until you press the button (prevents it switching on in a
  toolbox if the contacts brush something). Optional variant: auto-on at undock by sensing the
  falling edge of `VDOCK` with an RC pulse into the switch's ON input — decide after using it.
- **Forgot it on the truck bed:** firmware asserts OFF after 30 min with no dash packets *or* no
  change in tilt > 0.5°, and at 3.3 V/cell. Same OFF pin = true off (~0.01 µA switch leakage),
  not deep sleep.
- **Holster mechanics:** cradle near the dash; the unit's own base magnets hold it to a steel
  backplate, pogo pins on the side align via the magnetic connector. Use a **3-pin** connector:
  VDOCK, GND, SENSE. SENSE is tied to GND inside the reference unit; the dash pulls it up and
  reads it on a GPIO, so the dash knows "REF DOCKED" (vs. "REF OFF/out of range") without any radio.
- **Runtime:** ESP32 with ESP-NOW TX at 20 Hz (~80–120 mA avg) + SINAT (~25 mA at 12 V-equivalent)
  ≈ 150 mA from 6 Ah → ~35 h on. Easily a season of occasional use between charges even
  without the holster.
- Charging thermistor on the cells: a cab in sun exceeds 45 °C; BQ24074's TS input suspends
  charge outside 0–45 °C.

---

## 9. Phased build plan

| Phase | Work | Exit criteria |
|---|---|---|
| **0. Bench characterisation** (PC + USB-RS485) | Identify variant/wiring; verify Modbus map & scaling; set baud, rate, bandwidth; log static noise, tilt-table accuracy vs. digital level, step response; vibration test (handheld on running engine or orbital sander). Write `sinat_bench.py`, `analyze.py`. | Confirmed register map; chosen filter setting; static noise <0.05° RMS; measured bias between the two units |
| **1. Dash MVP (wired, absolute only)** | ESP32 + RS485 + display on breadboard; `modbus_rtu`, `sinat`, ABS screen; bench-powered from 12 V supply | Shows pitch/roll at ≥10 Hz, matches PC tool |
| **2. Tractor install, absolute mode** | Mount sensor + guard, run cable, power from tractor, temporary display mount; fork zero calibration flow | ±0.3° vs. digital level on tines across ±20°; readable in sun; survives a day of work |
| **3. Reference unit + differential** | Second ESP32 + SINAT + battery + power-switch/charger (§8.1); ESP-NOW link; `tiltmath` relative tilt w/ ψ; DIFF screen; cross-zero procedure | Δ within ±0.5° vs. digital level on a tilted plank/truck bed; link ≥50 m; stale handling works |
| **3b. Holster** | Cradle + pogo contacts + 5 V feed from dash; verify forced-off and idle-off | Docked: button dead, unit draws 0 from battery (measure), charges with ignition on; idle-off fires at timeout |
| **4. Hardening** | Proper enclosures, M12 connectors, power protection board (perfboard or small PCB), sun hood, ref magnet base | 1 month in service, no faults; wash-down OK |
| **5. Nice-to-haves** | Target-angle mode with audible tone; data log; OTA updates | — |

Rough effort: Phase 0 one evening, 1 a weekend, 2 a weekend, 3 a weekend, 4 one–two weekends.

---

## 10. Test plan

- **Unit (PC, pytest):** Modbus CRC vectors; scaling; `tiltmath` relative tilt against brute-force
  rotation-matrix reference over random angles and ψ; packet round-trip.
- **Bench:** tilt table steps of 1° over ±30°, both axes; temperature soak (truck cab in sun /
  freezer) to check bias drift; dropout test (unplug fork sensor → "SENSOR" error, not stale number).
- **Field:** engine idle vs. 2000 rpm noise; slow drive transients (unsteady flag behaviour);
  approach a known-tilt bed; link range with steel in the path; cold crank recovery.

---

## 11. Open questions for you

1. Confirm SINAT label reads RS485 (vs. RS232/TTL) and its minimum supply voltage
   (5 V vs. 9 V decides the reference unit's boost converter).
2. Undock behaviour: stay off until button (default) or auto-on when pulled from the holster?

Resolved: SSQA loader, user-built sensor bracket; forks only; 12 V hard-wired at dash;
reference unit on battery with holster forced-off + charging.
