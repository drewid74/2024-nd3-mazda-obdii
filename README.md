# 2024+ Mazda MX-5 ND3 — OBD-II PID Research

Confirmed working PIDs and implementation notes for the 2024+ MX-5 (ND3 platform).

> **ND3 is a different platform.** All prior ND1/ND2 PID work (berumiya DBC,
> timurrrr RaceChrono DB, agronick CSV) was built on the old architecture.
> The ND3 uses the CX-60 generation electronics with isolated CAN buses;
> the OBD port no longer exposes raw CAN traffic — only diagnostic
> request/response (Mode 01, Mode 22, etc.) is accessible from the port.

## Status

Validated on: **2025 MX-5 RF Club** with OBDLink MX+

| # | Signal | Mode | Header | PID | Formula | Units |
|---|--------|------|--------|-----|---------|-------|
| 1 | Engine RPM | 01 | 7DF | 01 0C | ((A*256)+B)/4 | RPM |
| 2 | Vehicle speed | 01 | 7DF | 01 0D | A | km/h |
| 3 | Coolant temp | 01 | 7DF | 01 05 | A-40 | °C |
| 4 | Throttle % | 01 | 7DF | 01 11 | A*100/255 | % |
| 5 | Engine load % | 01 | 7DF | 01 04 | A*100/255 | % |
| 6 | Intake air temp | 01 | 7DF | 01 0F | A-40 | °C |
| 7 | MAF | 01 | 7DF | 01 10 | ((A*256)+B)/100 | g/s |
| 8 | Fuel level % | 01 | 7DF | 01 2F | A*100/255 | % |
| 9 | Ambient temp | 01 | 7DF | 01 46 | A-40 | °C |
| 10 | Battery voltage | 01 | 7DF | 01 42 | ((A*256)+B)/1000 | V |
| 11 | Accel pedal % | 22 | 7E0 | 22 03 2B | A*100/255 | % |
| 12 | Oil temp | 22 | 7E0 | 22 13 10 | (((A*256)+B)/100)-40 | °C |
| 13 | Brake position % | 22 | 760 | 22 2B 0D | A*100/255 | % |

## The Response-Prefix Fix

If you are building an OBD app that mixes Mode 01 and Mode 22 PIDs,
see [`docs/prefix-fix.md`](docs/prefix-fix.md).

**TL;DR:** positive response SID = request SID + 0x40. Do not hardcode `0x62`.

The bug: apps that only ever tested Mode 22 PIDs hardcode `62` as the expected
response prefix. Mode 01 responds with `41`, Mode 09 with `49`, etc. The
generic fix:

```kotlin
val serviceId = pidHex.substring(0, 2).toInt(16)
val responseService = (serviceId + 0x40).toString(16).uppercase().padStart(2, 
'
0
'
)
```

This is not ND3-specific — it affects any app mixing modes against any car.

## Next / Unconfirmed

See [`pids/nd3_candidates.csv`](pids/nd3_candidates.csv) for the current
probe queue. PRs with test results welcome.

## Related

- [berumiya/CAN_DBC_6thGenMazda](https://github.com/berumiya/CAN_DBC_6thGenMazda) — ND1/ND2 raw CAN DBC (does not apply to ND3)
- [timurrrr/RaceChronoDiyBleDevice](https://github.com/timurrrr/RaceChronoDiyBleDevice) — ND1/ND2 CAN IDs for RaceChrono
- [miata.net — 2024 CAN bus research thread](https://forum.miata.net/vb/showthread.php?t=782326)

## Contributing

Test a candidate PID from `pids/nd3_candidates.csv` and open a PR updating
the status column. Include your car trim and model year in the PR description.
