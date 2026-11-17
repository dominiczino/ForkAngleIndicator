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
| R9 | Works with attachment swaps (forks ↔ bucket) | per-attachment zero offsets |
| R10 | Boot to reading | <3 s from key-on |

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
 on loader quick-attach plate               ESP32-S3 + display                 SINAT #2 + ESP32 + battery
 ┌─────────────┐  RS485, 4-core shielded   ┌──────────────────────┐  ESP-NOW  ┌──────────────────────┐
 │  SINAT-485  │──────────────────────────▶│ RS485 xcvr → UART    │◀ ─ ─ ─ ─ ─│ ESP32 ← RS485 ← SINAT│
 │  12 V in    │  ~5–8 m, M12 at loader    │ 2.0–2.8" IPS TFT     │  2.4 GHz  │ 18650 / 12 V socket  │
 └─────────────┘  disconnect               │ 2 buttons / encoder  │           │ magnet base, arrow   │
                                           │ buzzer, 12 V input   │           └──────────────────────┘
                                           └──────────────────────┘
```

Key decisions:

1. **Fork sensor wired, not wireless.** Power is available, RS485 is robust over long cable,
   no battery on the loader, no RF dropout behind steel. One cable along the existing hose route.
2. **Mount the sensor on the loader quick-attach plate, not on the fork frame.** The plate is
   rigidly coupled to whatever attachment is on, never leaves the tractor, and gets the same reading
   for forks or bucket. The plate→tine angle is a fixed offset handled by per-attachment calibration (§6).
   Mount on the back side, inside a steel guard, away from pallet contact.
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
| Reference: 18650 ×2 + holder, TP4056/IP5306 charger-boost or USB-C PD board, power switch | 1 | 5 V for ESP32; SINAT runs on 5 V | 20 |
| Reference: boost 5→12 V (only if your SINAT needs 9 V min) | 0–1 | check Phase 0 | 5 |
| Reference: enclosure, 4× pot magnets (≥10 kg pull each), machined/flat base plate | 1 | | 25 |
| USB-RS485 dongle for PC bench work | 1 | | 10 |
| **Total** | | | **~240** |

---

## 5. Measurement math

### 5.1 Absolute fork angle
The SINAT outputs fused roll/pitch directly. Fork pitch shown to operator:

```
θ_fork = θ_sensor − θ_offset[attachment]
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
2. **Mount offset per attachment**: forks resting flat on a floor whose tilt is measured by the
   reference unit (or a digital level); `θ_offset[forks] = θ_sensor − θ_floor`. Same for bucket
   (cutting edge level / bottom flat). Store 3–4 named profiles in flash; select with a button.
3. **Quick re-zero** (long-press): "forks are level now" for when an attachment is re-hung.
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
    config.py          # profiles, offsets, ψ, thresholds, load/save JSON
  ref/
    main.py            # read sensor 20 Hz → ESP-NOW broadcast; battery sense; deep sleep on idle
tools/ (CPython, run on PC)
  sinat_bench.py       # pyserial/minimalmodbus: read, configure, log CSV
  analyze.py           # noise PSD, Allan deviation, step response from logs (numpy/matplotlib)
tests/                 # pytest for tiltmath, modbus CRC, proto pack/unpack (pure Python, run on PC)
```

ESP-NOW packet (ref → dash, 20 Hz, broadcast to paired MAC):
`<u8 ver, u16 seq, u32 t_ms, i16 pitch×100, i16 roll×100, i16 |a|×1000, u8 flags, u8 batt%>`
Dash marks reference **stale** after 500 ms without packets and blanks the Δ readout.

UI:
- **ABS screen**: big pitch number with ▲/▼ arrow, horizontal bar ±15°, small roll, profile name.
- **DIFF screen**: big Δpitch, Δroll, link RSSI/age, ref battery, ψ indicator.
- Unsteady → number goes grey. Within target band → green + optional short beep.
- Buttons: short A = cycle screen, short B = cycle profile, long B = quick-zero, long A+B = settings.

---

## 8. Electrical / mechanical details

- **Power (dash):** fused switched-ignition 12 V → reverse-polarity protection → TVS (SMBJ33A) →
  buck 5 V → ESP32. SINAT fed from the protected 12 V rail directly (it accepts up to 36 V).
  Cold-crank brownout is acceptable; firmware must recover cleanly (nothing written on boot).
- **RS485:** A/B twisted pair + 12 V + GND in one shielded cable; shield grounded at dash end only;
  120 Ω termination at both ends (often on the modules, check). Keep sensor and dash at same
  baud; 115200 is plenty for 20 Hz polling.
- **Cable route:** along loader hose bundle with spiral wrap, P-clips at every hose clamp,
  service loops at every pivot (boom pivot and bucket pivot) sized for full range of motion.
  M12 disconnect at the loader-to-tractor parking point so the loader can be dropped.
- **Sensor mount:** bolt to a plate welded/clamped on the quick-attach frame; orient sensor X
  axis along tractor longitudinal axis. Steel U-guard over it. Thread-locker on bolts.
- **Dash unit:** RAM ball on ROPS post or fender; sun hood; IP65 print (ASA, gasketed lid,
  cable gland).
- **Reference unit:** flat base (aluminium plate with three hard-anodised feet or full flat face),
  magnets recessed so the feet/face carry the contact, large printed arrow, power switch,
  USB-C charging, battery % on a small LED or sent to dash.

---

## 9. Phased build plan

| Phase | Work | Exit criteria |
|---|---|---|
| **0. Bench characterisation** (PC + USB-RS485) | Identify variant/wiring; verify Modbus map & scaling; set baud, rate, bandwidth; log static noise, tilt-table accuracy vs. digital level, step response; vibration test (handheld on running engine or orbital sander). Write `sinat_bench.py`, `analyze.py`. | Confirmed register map; chosen filter setting; static noise <0.05° RMS; measured bias between the two units |
| **1. Dash MVP (wired, absolute only)** | ESP32 + RS485 + display on breadboard; `modbus_rtu`, `sinat`, ABS screen; bench-powered from 12 V supply | Shows pitch/roll at ≥10 Hz, matches PC tool |
| **2. Tractor install, absolute mode** | Mount sensor + guard, run cable, power from tractor, temporary display mount; calibration flow + profiles | ±0.3° vs. digital level on tines across ±20°; readable in sun; survives a day of work |
| **3. Reference unit + differential** | Second ESP32 + SINAT + battery; ESP-NOW link; `tiltmath` relative tilt w/ ψ; DIFF screen; cross-zero procedure | Δ within ±0.5° vs. digital level on a tilted plank/truck bed; link ≥50 m; stale handling works |
| **4. Hardening** | Proper enclosures, M12 connectors, power protection board (perfboard or small PCB), sun hood, ref magnet base | 1 month in service, no faults; wash-down OK |
| **5. Nice-to-haves** | Target-angle mode with audible tone; data log; OTA updates; bucket "return-to-dig" angle preset; second fork sensor for differential tilt between carriage halves (unlikely needed) | — |

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

1. Which SINAT variant is printed on the label — RS485, RS232, or TTL? (Drives transceiver choice.)
2. Tractor/loader model and quick-attach type (SSQA / Euro / JD / proprietary)? Determines bracket.
3. Is 12 V switched power easily available near the dash (accessory socket vs. hard-wire)?
4. Reference unit power: rechargeable battery, or plug into truck 12 V socket?
5. Want bucket mode (return-to-level) included from Phase 2, or forks only?
