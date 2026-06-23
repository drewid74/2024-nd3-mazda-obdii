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

**Methodology** (2026-06-23 update): mixed Mode 01 + Mode 22 candidate list
probed via auto-probe-all with raw response logging. Decoded values validated
against:

- Sanity ranges (e.g., RPM 700–7000, throttle 0–100%)
- Cross-PID correlation (throttle ↔ engine load ↔ MAF ↔ fuel rate)
- Real-world dynamics during 2-km drive captures (idle → cruise → WOT)
- Temperature warming curves (cold start → warm idle)
- Odometer drift matching actual distance driven (8698.8 → 8700.8 km = 2.0 km ✓)

## Confirmed Working PIDs (36)

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
| 14 | Fuel/Air equiv ratio (λ) | 7DF | 01 44 | `((A*256)+B)*2/65536` | — | 0.91 → 2.00 |
| 15 | Fuel rail gauge pressure | 7DF | 01 23 | `((A*256)+B)*10` | kPa | 9540 → 25110 |
| 16 | Fuel rail pressure abs | 7DF | 01 59 | `((A*256)+B)*10` | kPa | 10080 → 25590 |
| 17 | Fuel tank level | 7DF | 01 2F | `A*100/255` | % | matches dash gauge |
| 18 | Engine fuel rate | 7DF | 01 5E | `((A*256)+B)*0.05` | L/h | 0.9 → 10.5 |
| 19 | Engine/Vehicle fuel rate | 7DF | 01 9D | `((A*256)+B)/50` | g/s | 0.45 → 6.05 |
| 20 | Engine exhaust flow rate | 7DF | 01 9E | `((A*256)+B)/20` | kg/h | 11 → 120 |
| 21 | Catalyst temp B1S1 | 7DF | 01 3C | `((A*256)+B)/10-40` | °C | 365 → 591 |
| 22 | Battery voltage | 7DF | 01 42 | `((A*256)+B)/1000` | V | 12.5 (off) / 13.8 (on) |
| 23 | Actual engine torque | 7DF | 01 62 | `A-125` | % | 10 → 78 |
| 24 | Engine reference torque | 7DF | 01 63 | `(A*256)+B` | Nm | 250 (constant) |
| 25 | Engine friction torque | 7DF | 01 8E | `A-125` | % | 7 → 10 |
| 26 | Accel pedal position D | 7DF | 01 49 | `A*100/255` | % | 16 → 47 |
| 27 | Accel pedal position E | 7DF | 01 4A | `A*100/255` | % | 7.8 → 22.7 |
| 28 | Relative throttle position | 7DF | 01 45 | `A*100/255` | % | 1.96 → 30 |
| 29 | Absolute throttle position B | 7DF | 01 47 | `A*100/255` | % | 12 → 39.6 |
| 30 | Commanded throttle actuator | 7DF | 01 4C | `A*100/255` | % | 0 → 17 |
| 31 | Transmission gear ratio | 7DF | 01 A4 | `((C*256)+D)/1000` | — | 1.594 / 2.035 / 2.991 / 3.127 |
| 32 | Vehicle odometer | 7DF | 01 A6 | `((A*256³)+(B*256²)+(C*256)+D)/10` | km | drift matched real |

### Mode 22 (Manufacturer-specific)

| # | Signal | Header | PID | Formula | Units | Observed |
|---|--------|--------|-----|---------|-------|----------|
| 33 | Accel pedal % (encoding B) | 7E0 | 22 03 2B | `A/2` | % | 0 → 43.5 |
| 34 | Engine oil temperature | 7E0 | 22 13 10 | `(((A*256)+B)/100)-40` | °C | 66.7 → 78.4 |
| 35 | Brake position | 760 | 22 2B 0D | `max(0, signed_int16(A,B))/2.3` | % | 0 → 0.43 (light tap) |

### Static-by-design (correctly constant — not bugs)

- **01 33 Baro** = 101 kPa (sea level pressure; doesn't change in a 2-minute drive)
- **01 63 Reference torque** = 250 Nm (vehicle constant per SAE spec — NOT a runtime value)

## Gear Detection — `01 A4` (Engine-Running Required)

The standard SAE J1979-2 PID `0xA4` (Transmission Actual Gear) is supported on
ND3. **Critical quirk:** requires engine running.

**Byte layout** (data after `41 A4` prefix):

| Offset | Bytes | Meaning |
|--------|-------|---------|
| 0 | A | Support bitmap (always `0x03`) |
| 1 | B | Gear nibble — high nibble varies with engaged gear |
| 2–3 | C, D | Gear ratio = `((C*256)+D)/1000` |

**Engine-off behavior**: PCM holds the last-known gear ratio (often `5.087` =
1st gear default, the literal ND2/ND3 1st gear ratio). Gear is computed from
`RPM ÷ output-shaft-speed`, which is undefined when RPM=0. Shifter Hall sensor
alone is NOT what drives this PID.

**Observed ratios during real drive (engine on, 2026-06-23):**

- `1.594` (likely 4th-5th)
- `2.035` (likely 3rd)
- `2.991` (likely 2nd)
- `3.127` (likely 1st or transitional)

Byte B nibble observed varying `0x00 / 0x20 / 0x30 / 0x40` during drive — likely
encodes gear number directly (`nibble = gear`), but mapping needs confirmation
via parking-lot creep through 1st → 2nd → 3rd.

## Neutral / Gear Status — `01 65` (Byte B, NOT Byte A)

Common misimplementation: reading byte A which is **always `0x10`** (just the
SAE J1979 I/O support bitmap). The actual neutral/gear state lives in **byte B**,
which was observed varying `0x30 / 0x00 / 0x60 / 0x40` across captures during a
real drive.

**Bit-to-gear mapping still TBD** — needs correlated shift testing (neutral ↔
1st ↔ 2nd, engine running) to identify which bit encodes neutral and which
bits encode each gear number.

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

## Next / Unconfirmed

See [`pids/nd3_candidates.csv`](pids/nd3_candidates.csv) for the current
probe queue (updated with 2026-06-23 status). PRs with test results welcome.

Open questions:

- Bit-to-gear mapping for `01 65` byte B
- Confirm `01 A4` byte B high-nibble = gear number (parking-lot creep test)
- Confirm `760/22 20 33` is steering angle (engine-on turn test)
- Decode the 60 PIDs in the `22 F4 XX` cluster
- Find correct EPS address (try `0x7E2`, `0x720`, physical addressing)

## Related

- [berumiya/CAN_DBC_6thGenMazda](https://github.com/berumiya/CAN_DBC_6thGenMazda) — ND1/ND2 raw CAN DBC (does not apply to ND3)
- [timurrrr/RaceChronoDiyBleDevice](https://github.com/timurrrr/RaceChronoDiyBleDevice) — ND1/ND2 CAN IDs for RaceChrono
- [miata.net — 2024 CAN bus research thread](https://forum.miata.net/vb/showthread.php?t=782326)

## Contributing

Test a candidate PID from `pids/nd3_candidates.csv` and open a PR updating
the status column. Include your car trim and model year in the PR description.
