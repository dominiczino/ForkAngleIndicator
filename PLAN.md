# Tractor Fork Angle Indicator — Implementation Plan

Source idea (Todoist › Ideas): *"Dashboard indicator that uses a multiaxis accelerometer to indicate
the angle of the front loader forks compared to level. Also add the ability to measure the
differential angle compared to a wireless reference unit (placed on a truck bed or something)."*

Hardware on hand: 2× WitMotion SINAT dual-axis inclinometer (IP68, accel + gyro fused).

---

## 1. Requirements

| # | Requirement | Target |
|---|---|---|
| R1 | Fork pitch (tip up/down) vs. gravity, shown at operator station | ±0.3° static, 0.1° resolution |
| R2 | Fork roll (side tilt) vs. gravity | ±0.3°, secondary display |
| R3 | Differential mode: fork plane vs. remote reference plane (truck bed, trailer, rack) | ±0.5° |
| R4 | Update/latency | ≥10 Hz display, <200 ms latency |
| R5 | Readable in direct sun (open station, no cab), operable with gloves | — |
| R6 | Survives tractor electrical (12 V nominal, 14.4 V charging, cranking dip, load dump) | — |
| R7 | **No cab: every unit fully sealed** — rain, wash-down, dust, UV, vibration, loader impacts | IP67 minimum for dash, reference unit and connectors; sensor IP68 |
| R8 | Wireless reference range | ≥50 m line of sight |
| R9 | Forks only (no bucket mode) | single stored fork zero offset |
| R10 | Boot to reading | <3 s from key-on |
| R11 | Reference unit battery-powered; **magnetically forced hard-off while in its tractor holster** | logic off when docked; total drain ≲2 µA |
| R13 | **No exposed electrical contacts** on the reference unit | no charge port; cells removed and charged externally |
| R14 | Reference unit **turns on automatically when pulled from the holster** | on within ~1 s of undock |
| R12 | Reference unit auto-off if left on out of the holster | configurable, default 30 min idle |

Decided: SSQA (skid-steer quick-attach) loader, forks only, 12 V hard-wired, open station (no cab),
reference on battery with magnetic holster shutdown and no exposed contacts.
Confirmed: both sensors are SINAT-485 (order WIT3914SHOP). Minimum supply voltage not stated →
design runs both sensors at 12 V, which satisfies either 5–36 V or 9–36 V.

---

## 2. Sensor facts (WitMotion SINAT) — verify on bench in Phase 0

From retailer listings (manufacturer page unreachable from where this plan was written):

- Supply 5–36 V (one listing says 9–36 V; order email doesn't say). **Both units feed it 12 V**, so it
  doesn't matter. <25 mA.
- Interface: **RS485, Modbus RTU** (SINAT-485, confirmed by label and order). Baud 4800–230400,
  default typically 9600, address 0x50.
- Range X ±180°, Y ±90°; accuracy 0.1–0.2°; resolution ~0.0055°/LSB.
- Wire colours (typical WitMotion): red VCC, black GND, yellow/green A/B — **confirm before power-up**.
- WitMotion standard Modbus map (expected, confirm): 0x34–0x36 accel, 0x37–0x39 gyro,
  0x3D–0x3F roll/pitch/yaw, scale `value/32768 × 180°` (signed int16). Bandwidth/filter and
  output-rate registers exist; set them in Phase 0.

---

## 3. System architecture

```
 FORK SENSOR (SINAT #1)                     DASH UNIT (IP67)                   REFERENCE UNIT (IP67, portable)
 on loader SSQA plate                       ESP32-S3 + display                 SINAT #2 + ESP32 + battery
 ┌─────────────┐  RS485, 4-core shielded   ┌──────────────────────┐  ESP-NOW  ┌──────────────────────┐
 │  SINAT-485  │──────────────────────────▶│ RS485 xcvr → UART    │◀ ─ ─ ─ ─ ─│ ESP32 ← RS485 ← SINAT│
 │  12 V in    │  ~5–8 m, M12 at loader    │ 2.7" memory LCD      │  2.4 GHz  │ 2×18650, removable   │
 └─────────────┘  disconnect               │ + 3 status LEDs      │           │ magnet base, arrow   │
                                           │ 2 sealed buttons     │           │ hall: holster magnet │
                                           │ 12 V in (M12)        │           │ docked=off, out=on   │
                                           └──────────────────────┘           └──────────────────────┘
                                                                     HOLSTER: passive cradle + N52 magnet, no wiring
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
5. **Display for an open station:** reflective Sharp Memory LCD 2.7" (LS027B7DH01, 400×240, SPI).
   Contrast *improves* in direct sun, no backlight washout, wide temperature range. Paired with three
   bright LEDs (▼ tips down / ● level / ▲ tips up) readable at night and in peripheral vision while
   watching the forks. Emissive TFTs lose badly in full sun without a cab roof.
6. **Sealing approach:** gasketed enclosures, sealed (IP67) panel parts only, M12 connectors,
   ePTFE pressure-equalisation vent in each enclosure (stops it pumping in moisture through the
   gasket as it heats/cools), conformal coat on all boards.

---

## 4. Bill of materials (estimate, excluding the two SINATs you have)

| Item | Qty | Notes | ~USD |
|---|---|---|---|
| ESP32-S3 dev board (e.g. DevKitC-1; external antenna variant for the dash if the enclosure is metal) | 2 | dash + ref | 25 |
| RS485 transceiver module, 3.3 V, auto-direction (MAX13487 / SP3485 + auto-DE) | 2 | | 8 |
| Sharp Memory LCD 2.7" LS027B7DH01 + breakout | 1 | reflective, sun-readable | 30 |
| 3× 5 mm high-brightness LEDs (R/G/R) + IP67 panel lens/holders | 1 set | | 10 |
| Automotive 12 V→5 V buck, 6–40 V in, ≥1 A (e.g. Pololu D36V6F5 / D24V10F5) | 2 | | 20 |
| Protection: 1N5822 or ideal-diode reverse protect, SMBJ33A TVS, 1 A blade fuse + holder | 2 sets | load dump | 10 |
| 4-core shielded PUR cable, oil/UV resistant, 22–24 AWG | 8 m | | 25 |
| M12 4-pin A-coded connector pair (panel + cable), IP67 | 2 | loader detach point + dash | 25 |
| Cable glands, spiral wrap / hose sleeve, P-clips, zip ties | — | | 15 |
| Steel guard for fork sensor (bent 3 mm plate) + M5 stainless bolts | 1 | | 10 |
| IP67 momentary buttons ×2 (stainless anti-vandal, O-ring), IP67 piezo buzzer | 1 set | gloves | 20 |
| Dash enclosure: IP67 polycarbonate box (e.g. Hammond 1555 / Bud PN series) with display window cut + bonded PC/glass window, RAM ball mount, sun visor | 1 | UV-stable; printed ASA only with gasket groove | 40 |
| ePTFE breather vents (M12 screw-in, e.g. Amphenol LTW VENT-PS1) | 2 | dash + reference | 10 |
| Conformal coating (acrylic, e.g. MG 419D) | 1 | | 15 |
| Reference: 2× **protected** 18650 (1S2P) + holder | 1 | ~6 Ah; pulled and charged in your existing charger | 15 |
| Reference: soft-latching power switch with ON/OFF inputs (Pololu Mini Pushbutton Power Switch LV) | 1 | hard-off, see §8.1 | 10 |
| Reference: TI DRV5032 nano-power hall switch (low-sensitivity variant, push-pull) + 74LVC1G04 inverter + RC parts | 1 | holster: off when docked, ON pulse at undock | 3 |
| Reference: 1-cell → 12 V boost for SINAT (e.g. Pololu U3V16F12 or MT3608 module) | 1 | covers 5 V or 9 V min | 6 |
| Reference: 3.3 V buck-boost for ESP32 (Pololu S7V8F3) | 1 | full use of 3.0–4.2 V cell range | 8 |
| Reference: IP67 momentary button, bi-colour LED behind sealed light pipe | 1 | | 10 |
| Reference: IP67 PC enclosure with gasketed screw lid over the cell holder (captive screws), 4× pot (cup) magnets ≥10 kg pull, flat base plate | 1 | cup magnets keep stray field low | 30 |
| Holster: cradle (bent steel or printed ASA, drain holes), 1× N52 disc magnet ~12×5 mm, rubber pads | 1 | passive, no wiring | 15 |
| USB-RS485 dongle for PC bench work | 1 | | 10 |
| **Total** | | | **~330** |

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
- Memory LCD is monochrome: unsteady → number shown outlined/hatched instead of solid;
  in target band → solid "LEVEL" banner. The three LEDs carry the colour cue (▼/●/▲, ● lit
  within ±1°, flashing when unsteady). Optional beep.
- Buttons: short A = cycle ABS/DIFF, short B = cycle ψ (DIFF screen), long B = quick-zero,
  long A+B = settings/calibration.
- Dash shows ref state: "REF ON" + battery %, "REF DOCKED", "REF IDLE-OFF", "REF LOW BATT",
  or "REF LOST" (no farewell packet received).

Reference-unit packet `flags` carries a reason code when it is about to shut down (docked / idle /
low battery). It sends that farewell packet 3× during the ~1 s hardware delay (§8.1), so the
dash knows *why* the link dropped.

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
- **Dash unit (open station):** IP67 polycarbonate box, gasketed lid, window bonded (not just
  clamped) over the LCD, M12 panel connector for 12 V + RS485 (no cable glands that loosen with
  vibration), ePTFE vent on the lowest-but-sheltered face, stainless hardware, conformal-coated
  boards. Mount on a RAM ball off the ROPS or fender with a short visor. Assume it gets
  pressure-washed.
- **Wiring:** fork cable and power land on M12 connectors at the dash; all splices adhesive-lined
  heat-shrink; fused 12 V tapped at the battery/ignition circuit with a sealed fuse holder.
- **Reference unit:** IP67 PC enclosure, no external contacts or ports. Cells behind a gasketed
  lid with captive screws (you'll open it every few weeks of use — make the gasket a glued-in
  O-ring cord or a moulded gasket so it survives repeated opening). Flat base (aluminium plate
  with three feet or full flat face), cup magnets recessed so the base carries the contact, large
  arrow on top, one sealed button, LED behind a sealed light pipe, ePTFE vent. SINAT #2 bolted
  rigidly to the base plate (its cable is fully internal; coil the excess).

### 8.1 Reference unit power and magnetic holster on/off

```
                     ┌───────────── always powered from cell, ~1 µA ─────────────┐
 holster N52 ~~~▶ DRV5032 hall (out LOW = docked)                                │
                     │                                                           │
                     ├──▶ inverter ──RC delay ~1 s──▶ OFF  ┐                     │
                     ├──▶ RC differentiator (rising edge = undock) ──▶ ON          │
                     └──▶ ESP32 GPIO (dock IRQ → farewell packet)                  │
                                                           ▼                     │
 2× protected 18650 (1S2P) ──────────────────▶ Pololu pushbutton power switch ───┴─▶ 3.3 V buck-boost ─▶ ESP32
                                               ▲ A: sealed button (toggle)        └─▶ 12 V boost ───────▶ SINAT
                                               ▲ OFF ◀── ESP32 GPIO (idle / low batt)
```

Power state is driven by the holster:

| Event | Hardware action | Result |
|---|---|---|
| Unit placed in holster | Hall → LOW; after ~1 s RC, OFF held high | Off; button ineffective while docked |
| Unit pulled from holster | Hall → HIGH; rising edge → ~10 ms pulse into ON; OFF released | **Turns on automatically** |
| Out of holster, idle 30 min (no dash packets or no tilt change > 0.5°) | ESP32 drives OFF | Off; press button to turn back on |
| Cell < 3.3 V | ESP32 drives OFF | Off; flagged to dash first |
| Button press out of holster | Toggles switch | Manual on/off |

- **Docked = off is hardware-only:** firmware cannot hold it on. The ~1 s delay lets firmware
  send its "docked" farewell packet; if firmware is hung the RC fires anyway.
- **Undock = on** also hardware-only, so it works even if the unit had timed out earlier.
  Firmware checks the hall pin at boot and powers straight back off if it somehow booted while
  docked (e.g. bumped and re-seated).
- **Hall switch, not reed:** reeds are glass and fail/chatter under tractor shock. DRV5032 is
  solid-state, ~0.5–1 µA; docked drain ≲2 µA total → irrelevant vs. cell self-discharge.
- **Stray-field risk (main one):** the unit's own mounting magnets, or steel, must never trip the
  hall sensor — out on a truck bed that would power it *off*. Mitigations: cup/pot magnets
  (field confined to base face); hall IC at the top face, far end from them; lowest-sensitivity
  DRV5032 variant; holster keyed so the unit seats one way with the N52 directly over the IC.
  **Verify in Phase 3b** by measuring at the IC location with the unit stuck to a steel bed,
  a toolbox, and the tractor frame.
- **Cells:** pulled and charged in an external charger. Use **protected** cells (second layer of
  over-discharge/short protection) and insert both freshly charged — paralleling cells at
  different voltage drives a large equalising current. Holder polarity marked on the lid inside.
- **Regulation:** 3.3 V buck-boost for the ESP32 uses the whole 4.2→3.0 V range; separate
  1-cell→12 V boost for the SINAT covers either minimum-voltage spec.
- **Runtime:** ESP32 with ESP-NOW TX at 20 Hz (~80–120 mA) + SINAT (~25 mA at 12 V ≈ 90 mA from
  the cell) ≈ 180–210 mA → ~30 h of on-time per charge. With holster-off and idle-off, that's
  weeks of normal use between cell swaps. Dash shows battery %.

---

## 9. Phased build plan

| Phase | Work | Exit criteria |
|---|---|---|
| **0. Bench characterisation** (PC + USB-RS485) | Identify variant/wiring; verify Modbus map & scaling; set baud, rate, bandwidth; log static noise, tilt-table accuracy vs. digital level, step response; vibration test (handheld on running engine or orbital sander). Write `sinat_bench.py`, `analyze.py`. | Confirmed register map; chosen filter setting; static noise <0.05° RMS; measured bias between the two units |
| **1. Dash MVP (wired, absolute only)** | ESP32 + RS485 + display on breadboard; `modbus_rtu`, `sinat`, ABS screen; bench-powered from 12 V supply | Shows pitch/roll at ≥10 Hz, matches PC tool |
| **2. Tractor install, absolute mode** | Mount sensor + guard, run cable, power from tractor, temporary display mount; fork zero calibration flow | ±0.3° vs. digital level on tines across ±20°; readable in sun; survives a day of work |
| **3. Reference unit + differential** | Second ESP32 + SINAT + removable cells + power switch/hall circuit (§8.1); ESP-NOW link; `tiltmath` relative tilt w/ ψ; DIFF screen; cross-zero procedure | Δ within ±0.5° vs. digital level on a tilted plank/truck bed; link ≥50 m; stale handling works |
| **3b. Holster on/off** | Hall + inverter + RC delay into OFF, RC edge pulse into ON; cradle with N52; stray-field survey (§8.1) | Docked: off within ~1 s, button ineffective, drain ≤2 µA (measured); undocked: on within ~1 s every time (50 cycles); no false trip on steel truck bed/toolbox/frame; idle-off fires |
| **4. Hardening / sealing** | IP67 enclosures, bonded windows, M12 connectors, vents, conformal coat, power protection board (perfboard or small PCB), ref magnet base | Hose test (garden hose, all angles, 3 min) and 30 min submersion of ref unit at 0.5 m with no ingress; 1 month in service, no faults; LCD readable at noon |
| **5. Nice-to-haves** | Target-angle mode with audible tone; data log; OTA updates | — |

Rough effort: Phase 0 one evening, 1 a weekend, 2 a weekend, 3 a weekend, 4 one–two weekends.

---

## 10. Test plan

- **Unit (PC, pytest):** Modbus CRC vectors; scaling; `tiltmath` relative tilt against brute-force
  rotation-matrix reference over random angles and ψ; packet round-trip.
- **Bench:** tilt table steps of 1° over ±30°, both axes; temperature soak (truck cab in sun /
  freezer) to check bias drift; dropout test (unplug fork sensor → "SENSOR" error, not stale number).
- **Ingress:** after each hose/submersion test open and inspect; check the vent area for
  condensation after a hot-sun → cold-rain cycle (park in sun, hose with cold water).
- **Holster:** 50 dock/undock cycles; drop-in at an angle; ride a rough field docked — must not
  wake or rattle loose.
- **Field:** engine idle vs. 2000 rpm noise; slow drive transients (unsteady flag behaviour);
  approach a known-tilt bed; link range with steel in the path; cold crank recovery.

---

## 11. Decisions log

- SINAT-485 ×2 (order WIT3914SHOP); min supply voltage unknown → run at 12 V everywhere.
- SSQA loader, user-built sensor bracket; forks only.
- 12 V hard-wired at dash; open station → everything IP67.
- Reference unit: removable protected 18650s charged externally (no Qi, no charge port, no
  exposed contacts); passive magnetic holster: docked = hard off, undocked = auto on; idle and
  low-battery auto-off.

No open questions; next step is Phase 0 bench work.
