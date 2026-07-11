# 2024+ Mazda MX-5 ND3 — OBD-II PID Research

Confirmed working PIDs and implementation notes for the 2024+ MX-5 (ND3 platform).

> **ND3 is a different platform.** All prior ND1/ND2 PID work (berumiya DBC,
> timurrrr RaceChrono DB, agronick CSV) was built on the old architecture.
> The ND3 uses the CX-60 generation electronics with isolated CAN buses;
> the OBD port no longer exposes raw CAN traffic — only diagnostic
> request/response (Mode 01, Mode 22, etc.) is accessible from the port.

## Status

Validated on: **2025 MX-5 RF Club** (Cal ID `PFD2EB000PEP6010`, Engine Family
`STKXV02.0FFB`) with **OBDLink MX+** over Bluetooth Classic SPP.

**Methodology** (2026-06-23): mixed Mode 01 + Mode 22 candidate list
probed via auto-probe-all with raw response logging. Decoded values validated
against:

- Sanity ranges (e.g., RPM 700–7000, throttle 0–100%)
- Cross-PID correlation (throttle ↔ engine load ↔ MAF ↔ fuel rate)
- Real-world dynamics during 2-km drive captures (idle → cruise → WOT)
- Temperature warming curves (cold start → warm idle)
- Odometer drift matching actual distance driven (8698.8 → 8700.8 km = 2.0 km ✓)

**Methodology addendum** (2026-07-10): an **OBDLink "Diagnostic Report" export**
enumerates every PID the ECU actually answers, across Modes 01/02/05/06/09, in a
single pull. This is a dramatically faster discovery path than range-scanning
and is how the additions below were found. If you own an MX+, generate one
before you start probing — it tells you what to look for.

## Confirmed Working PIDs (47)

### Mode 01 (Standard OBD-II — should work on any car responding to `01 00`)

| # | Signal | Header | PID | Formula | Units | Observed range |
|---|--------|--------|-----|---------|-------|----------------|
| 1 | Engine RPM | 7DF | 01 0C | `((A*256)+B)/4` | RPM | 806 → 2367 |
| 2 | Vehicle speed | 7DF | 01 0D | `A` | km/h | 13 → 58 |
| 3 | Coolant temp | 7DF | 01 05 | `A-40` | °C | 80 → 89 (warming) |
| 4 | Coolant temp 1 & 2 | 7DF | 01 67 | `B-40` (byte B) | °C | mirrors 0x05 |
| 5 | Throttle position | 7DF | 01 11 | `A*100/255` | % | 13 → 92 |
| 6 | Engine load | 7DF | 01 04 | `A*100/255` | % | 14 → 99 |
| 7 | Absolute load | 7DF | 01 43 | `((A*256)+B)*100/255` | % | 11 → 55 |
| 8 | Intake air temp | 7DF | 01 0F | `A-40` | °C | 31 → 35 |
| 9 | Ambient air temp | 7DF | 01 46 | `A-40` | °C | 27 → 35 |
| 10 | MAF airflow | 7DF | 01 10 | `((A*256)+B)/100` | g/s | 2.5 → 43.8 |
| 11 | Intake manifold pressure | 7DF | 01 0B | `A` | kPa | 19 → 94 |
| 12 | Barometric pressure | 7DF | 01 33 | `A` | kPa | 101 (constant) |
| 13 | Ignition timing advance | 7DF | 01 0E | `(A/2)-64` | °BTDC | 7.5 → 34.5 |
| 14 | Fuel/Air **commanded** equiv ratio (λ) | 7DF | 01 44 | `((A*256)+B)*2/65536` | — | 0.91 → 2.00 |
| 15 | Fuel rail gauge pressure | 7DF | 01 23 | `((A*256)+B)*10` | kPa | 9540 → 25110 |
| 16 | Fuel rail pressure abs | 7DF | 01 59 | `((A*256)+B)*10` | kPa | 10080 → 25590 |
| 17 | Fuel tank level | 7DF | 01 2F | `A*100/255` | % | matches dash gauge |
| 18 | Engine fuel rate | 7DF | 01 5E | `((A*256)+B)*0.05` | L/h | 0.9 → 10.5 |
| 19 | Engine/Vehicle fuel rate | 7DF | 01 9D | `((A*256)+B)/50` | g/s | 0.45 → 6.05 |
| 20 | Engine exhaust flow rate | 7DF | 01 9E | `((A*256)+B)/20` | kg/h | 11 → 120 |
| 21 | Catalyst temp B1S1 | 7DF | 01 3C | `((A*256)+B)/10-40` | °C | 365 → 622 |
| 22 | Battery voltage | 7DF | 01 42 | `((A*256)+B)/1000` | V | 12.5 (off) / 13.8 (on) |
| 23 | Actual engine torque | 7DF | 01 62 | `A-125` | % | 10 → 78 |
| 24 | Engine reference torque | 7DF | 01 63 | `(A*256)+B` | Nm | 250 (constant) |
| 25 | Engine friction torque | 7DF | 01 8E | `A-125` | % | 7 → 10 |
| 26 | Accel pedal position D | 7DF | 01 49 | `A*100/255` | % | 16 → 47 |
| 27 | Accel pedal position E | 7DF | 01 4A | `A*100/255` | % | 7.8 → 22.7 |
| 28 | Relative throttle position | 7DF | 01 45 | `A*100/255` | % | 1.96 → 30 |
| 29 | Absolute throttle position B | 7DF | 01 47 | `A*100/255` | % | 12 → 39.6 |
| 30 | Commanded throttle actuator | 7DF | 01 4C | `A*100/255` | % | 0 → 17 |
| 31 | Transmission gear status + ratio | 7DF | 01 A4 | see [Gear Detection](#gear-detection--01-a4) | — | 1.594 / 2.035 / 2.991 / 3.127 |
| 32 | Vehicle odometer | 7DF | 01 A6 | `((A*256³)+(B*256²)+(C*256)+D)/10` | km | drift matched real |

### Mode 01 — additions confirmed 2026-07-10

All of these were answered by the ECU but were **absent from the original probe
list**. Found via OBDLink Diagnostic Report export.

| # | Signal | Header | PID | Formula | Units | Observed |
|---|--------|--------|-----|---------|-------|----------|
| 36 | **O2 lambda wide-range, actual (B1S1)** | 7DF | 01 34 | λ = `((A*256)+B)*2/65536`; current mA = `((C*256)+D)/256-128` | — / mA | λ 0.995, −0.004 mA |
| 37 | Short-term fuel trim, bank 1 | 7DF | 01 06 | `(A-128)*100/128` | % | 3.91 |
| 38 | Long-term fuel trim, bank 1 | 7DF | 01 07 | `(A-128)*100/128` | % | 0.78 |
| 39 | **Intake air temp, sensors 1 & 2** | 7DF | 01 68 | A = support bitmap (`3`); IAT1 = `B-40`; IAT2 = `C-40` | °C | 36.0 / 46.0 |
| 40 | Fuel rail pressure control (A) | 7DF | 01 6D | A = support (`7`); commanded, actual, and **rail temp** follow | kPa / °C | rail temp 56 °C |
| 41 | O2 sensor B1S2 voltage + STFT | 7DF | 01 15 | V = `A/200`; STFT = `(B-128)*100/128` | V / % | 0.73 V, 0% |
| 42 | Secondary O2 trim, short-term B1 | 7DF | 01 55 | `(A-128)*100/128` | % | 0 |
| 43 | Secondary O2 trim, long-term B1 | 7DF | 01 56 | `(A-128)*100/128` | % | 0 |
| 44 | Time since engine start | 7DF | 01 1F | `(A*256)+B` | sec | 1402 |
| 45 | Fuel system 1 / 2 status | 7DF | 01 03 | bitmap | — | Closed-loop (B1); Unknown (B2 — N/A, single bank) |
| 46 | O2 sensor locations | 7DF | 01 13 | bitmap | — | B1S1 + B1S2 only |
| 47 | Commanded evaporative purge | 7DF | 01 2E | `A*100/255` | % | 32.94 |

**`01 34` is the important one.** The repo previously listed only `01 44`
(*commanded* λ). `01 34` is the **actual measured wideband λ** from the B1S1
sensor. Poll both and the difference is your closed-loop fuelling error, direct
from the ECU — no AFR inference needed.

**`01 68` matters on track.** There are *two* IAT sensors. Sensor 1 reads inlet
air; sensor 2 reads further into the tract. The **delta between them is intake
heat-soak**, and it climbs measurably across a session.

### Mode 22 (Manufacturer-specific)

| # | Signal | Header | PID | Formula | Units | Observed |
|---|--------|--------|-----|---------|-------|----------|
| 33 | ~~Accel pedal % (encoding B)~~ | 7E0 | 22 03 2B | `A/2` | % | **REDUNDANT — use `01 49` / `01 4A`** |
| 34 | Engine oil temperature | 7E0 | 22 13 10 | `(((A*256)+B)/100)-40` | °C | 66.7 → 78.4 |
| 35 | Brake position | 760 | 22 2B 0D | `max(0, signed_int16(A,B))/2.3` | % | 0 → 0.43 (light tap) |

**Only oil temp and brake position justify Mode 22.** Everything else the PCM
exposes via Mode 22 that anyone has needed so far is also in the standard J1979
space. `22 03 2B` duplicates `01 49`; drop it and save the bus round-trip.

> **General lesson — do not skip the standard PID space.**
> A Mode 22 NACK means *that ND2-inherited module address is dead*, **not** that
> the signal is unavailable. Several signals initially written off as
> "needs a physical CAN tap" turned out to sit in plain SAE J1979. Always probe
> `01 00 → 01 20 → 01 40 → 01 60 → 01 80 → 01 A0 → 01 C0` and enumerate what the
> car actually advertises **before** concluding anything is out of scope.

### Static-by-design (correctly constant — not bugs)

- **01 33 Baro** = 101 kPa (sea level pressure; doesn't change in a 2-minute drive)
- **01 63 Reference torque** = 250 Nm (vehicle constant per SAE spec — NOT a runtime value)

### Derived: ECU-reported torque

`01 62` (actual engine torque, %) × `01 63` (reference torque, Nm) gives a live
flywheel torque estimate straight from the ECU:

```
torque_Nm = (pid_62_percent / 100) * 250
```

Useful as a free before/after instrument for intake/exhaust/tune changes — no
dyno required for a relative comparison.

## Gear Detection — `01 A4`

The standard SAE J1979-2 PID `0xA4` (Transmission Actual Gear) **is supported on
ND3**, and returns both an enumerated gear status and the gear ratio.

**Byte layout** (data after `41 A4` prefix):

| Offset | Bytes | Meaning |
|--------|-------|---------|
| 0 | A | Support bitmap (always `0x03`) |
| 1 | B | Gear status — enumerated; decodes to `Neutral` when stopped |
| 2–3 | C, D | Gear ratio = `((C*256)+D)/1000` |

**Stopped behavior — corrected 2026-07-10.** The previous revision of this README
attributed the `5.087` ratio to *engine-off* state. That is **wrong**. A capture
with the **engine running** (747 RPM, warm idle, vehicle speed 0, in neutral)
returns:

```
Transmission Actual Gear Status Supported : 3
Transmission Actual Gear Status           : Neutral
Transmission Actual Gear Ratio            : 5.09
```

The mechanism is right but the trigger was misidentified: gear ratio is computed
from `engine speed ÷ output-shaft speed`, which is undefined whenever **output
shaft speed = 0** — i.e. any time the car is **stopped**, engine on or off. The
`5.087` value is the **neutral / undefined-ratio placeholder**, not an engine-off
artifact. It happens to equal the ND 1st-gear ratio, which is what made it
look like a "last known gear" hold.

Practical consequence: **`01 A4` is only meaningful while the car is moving.**
Gate it on `01 0D > 0`.

**Observed ratios during real drive (engine on, 2026-06-23):**

- `1.594` (likely 4th–5th)
- `2.035` (likely 3rd)
- `2.991` (likely 2nd)
- `3.127` (likely 1st or transitional)

Byte B was observed varying `0x00 / 0x20 / 0x30 / 0x40` during drive and cleanly
decodes `Neutral` at rest — so it *is* an enumerated gear status, not a raw
nibble. Exact value→gear mapping still wants a parking-lot creep test through
1st → 2nd → 3rd.

## `01 65` — Recommended Gear (Shift Indicator), NOT Current Gear

**Corrected 2026-07-10. The previous revision had this wrong.**

This PID was previously documented here as "Neutral / Gear Status" with a
bit-to-gear mapping listed as an open question. It is **not** current gear state.

OBDLink's decode of the same PID:

```
0x65  Auxiliary inputs / outputs supported                          16
0x65  Recommended transmission gear for current vehicle conditions  Neutral
```

`01 65` is the SAE **shift indicator / recommended gear** parameter — the
"upshift now" hint used for fuel-economy coaching. That is why byte B was seen
varying `0x30 / 0x00 / 0x60 / 0x40` during a drive: it was recommending
*different gears*, not reporting the engaged one.

- Byte A is the aux I/O **support bitmap** (`0x10` / `16`) — as previously noted,
  do not read it as data. That part was correct.
- **For actual engaged gear, use `01 A4`.**
- This is still a *useful* channel — it's a free shift light — just not the one
  the section title claimed.

**Open question closed.**

## Mode $06 — On-Board Monitoring (undocumented for ND3 until now)

Mode $06 returns **continuous measured values with min/max thresholds** — not
pass/fail bits. It is completely unexploited by consumer OBD apps and is, as far
as we know, undocumented for the ND3 platform.

See [`docs/mode-06-monitoring.md`](docs/mode-06-monitoring.md) for the full TID
table and a real-world worked example (a catalyst failure that was measurable
in Mode $06 well before the MIL illuminated).

Headline: monitor `$21` TID `$82` is the catalyst **oxygen storage capacity** —
a trendable cat-health gauge:

```
$21 - Catalyst Monitor Bank 1
  TID $82    value 0.60    min 0.67    max 655.35    FAIL
```

Also live on ND3: `$01`/`$02` exhaust gas sensor monitors (including B1S2 rich→lean
and lean→rich **switch times**), `$35` VVT (6 TIDs), `$41`/`$42` O2 heater
resistance in kOhm, and `$A2`–`$A5` **per-cylinder misfire EWMA counts**.

Per-cylinder misfire counts as a track telemetry channel is, to our knowledge,
novel.

## Mode $09 — In-Performance Tracking

Also answered by the ND3 and not previously documented here. Includes monitor
completion/condition counters (useful for knowing whether a readiness monitor has
actually *run*), plus lifetime/recent counters:

- Ignition cycle counter (recent / lifetime)
- Total engine run time, total idle run time
- Total distance traveled, total fuel consumed
- Positive kinetic energy (km/hr²)
- Engine output energy (kWh)
- Catalyst / O2 / EGR-VVT monitor completion counts

Most "Active Off-Cycle Credit" / "Driver Coaching" / "Grille Shutter" timers
return `4294967295` (`0xFFFFFFFF` = not supported) — expected on this platform.

## Steering Cluster — DSC `0x760/22 20 XX`

Range scan revealed 4 PIDs in this cluster on the **DSC module** (not the EPS
module at `0x730`, which is silent). Currently pending engine-on turn-test
validation.

| PID | Hypothesis | Stationary value |
|-----|-----------|------------------|
| `22 20 33` | Steering angle (signed 16-bit / 10 = deg) | `00 00` centered (engine on); `FF FF`/`00 00` no-data marker (engine off) |
| `22 20 34` | Steering rate or alt angle (signed 16-bit / 10) | `00 00` static |
| `22 20 1D` | Status flag (DSC armed?) | `0x04` |
| `22 20 1F` | Status flag (fault indicator?) | `0x00` |

**Engine-off test** (5 snapshots, clutch in, shifter moved through gears):
none of these varied → SAS/DSC does NOT actively measure steering with the
engine off. Engine-on + parked + wheel `center → full-right → full-left` is
the required validation procedure.

## Mode 22 PCM Cluster Discovery

Range scan of `7E0/22 XX XX` clusters returned a massive trove (256 PIDs ×
6 clusters = 1536 probes in ~6 minutes):

| Cluster | Hits | Description |
|---------|------|-------------|
| `22 09 XX` | 8 | Cam / VVT / valve cluster |
| `22 13 XX` | 1 | Thermal (just oil temp `13 10`) |
| `22 F4 XX` | **60** | Fuel / boost / MAP — biggest trove |
| `22 DA XX` | 36 | Service / fuel learning (includes VIN + Cal ID fragments) |
| `22 03 XX` | 38 | Engine control (includes accel pedal `03 2B`) |
| `22 2B XX` (DSC) | 11 | Chassis (includes brake `2B 0D`) |

Full byte-level results: [`pids/range-scan-2026-06-23.md`](pids/range-scan-2026-06-23.md)

> **Cross-validation opportunity:** Mode $06 monitor `$35` exposes six VVT TIDs
> with engineering units (degrees, seconds) and known min/max bounds. These may
> serve as a free decode check against the 8 hits in the `22 09 XX` cam/VVT
> cluster.

## Dead Ends (Don't Re-Probe)

These were thoroughly tested and returned silence; do not waste time re-trying:

- **EPS at `0x730`** — all subranges (`22 20 XX`, `22 06 XX`) silent across 512
  PIDs. EPS does NOT respond to `0x730` header on ND3. Workshop manual address
  may be wrong, or EPS requires different addressing (try `0x7E2`, `0x720`?).
- **`760/22 21 XX`** — yaw rate / lateral G cluster NOT on DSC at this range
  (0 hits in 256-PID scan). Use IMU/GPS instead (RaceBox provides at 25 Hz).
- **`760/22 06 XX`** — wheel speeds NOT exposed via DSC at this range. The
  DSC module knows wheel speeds (it has to for ABS/traction), but does not
  publish them at this Mode 22 sub-address.

**Caveat added 2026-07-10:** "dead end" here means *dead at this Mode 22 address*.
Before adding anything to this list, confirm it is also absent from the standard
Mode 01 support bitmaps. Gear position was previously (incorrectly) treated as
unreachable on exactly this reasoning — it was in `01 A4` the whole time.

## The Response-Prefix Fix

If you are building an OBD app that mixes Mode 01 and Mode 22 PIDs,
see [`docs/prefix-fix.md`](docs/prefix-fix.md).

**TL;DR:** positive response SID = request SID + 0x40. Do not hardcode `0x62`.

The bug: apps that only ever tested Mode 22 PIDs hardcode `62` as the expected
response prefix. Mode 01 responds with `41`, Mode 09 with `49`, etc. The
generic fix:

```kotlin
val serviceId = pidHex.substring(0, 2).toInt(16)
val responseService = (serviceId + 0x40).toString(16).uppercase().padStart(2, '0')
```

This is not ND3-specific — it affects any app mixing modes against any car.

Note that Mode $06 responds with `46` and Mode $02 (freeze frame) with `42`,
both of which fall out of the same rule.

## Next / Unconfirmed

See [`pids/nd3_candidates.csv`](pids/nd3_candidates.csv) for the current
probe queue. PRs with test results welcome.

Open questions:

- Confirm `01 A4` byte B value→gear mapping (parking-lot creep test, car moving)
- Confirm `760/22 20 33` is steering angle (engine-on turn test)
- Decode the 60 PIDs in the `22 F4 XX` cluster
- Cross-check `22 09 XX` cam/VVT decodes against Mode $06 `$35` TIDs
- Find correct EPS address (try `0x7E2`, `0x720`, physical addressing)
- Full `01 6D` byte layout (support bitmap + commanded/actual/temp triplet)

Closed:

- ~~Bit-to-gear mapping for `01 65` byte B~~ — `01 65` is *recommended* gear
  (shift indicator), not current gear. Not applicable.

## Related

- [berumiya/CAN_DBC_6thGenMazda](https://github.com/berumiya/CAN_DBC_6thGenMazda) — ND1/ND2 raw CAN DBC (does not apply to ND3)
- [timurrrr/RaceChronoDiyBleDevice](https://github.com/timurrrr/RaceChronoDiyBleDevice) — ND1/ND2 CAN IDs for RaceChrono
- [miata.net — 2024 CAN bus research thread](https://forum.miata.net/vb/showthread.php?t=782326)

## Contributing

Test a candidate PID from `pids/nd3_candidates.csv` and open a PR updating
the status column. Include your car trim and model year in the PR description.

If you have an OBDLink adapter, a **Diagnostic Report export** from your own ND3
is extremely valuable — it enumerates everything your ECU answers in one shot,
including Modes $02/$05/$06/$09. Trim and model-year variation across ND3 builds
is currently unmapped.
