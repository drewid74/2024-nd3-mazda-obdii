# 2024+ Mazda MX-5 ND3 — OBD-II PID Research

Vehicle-observed PIDs, parked diagnostic replies, and implementation notes for
the 2024+ MX-5 (ND3 platform).

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
lists supported standard PIDs, monitor results, and vehicle information. It
was the source of the Mode 01 additions and the Mode 06/09 observations below.
The report is a faster discovery starting point than range scanning, but its
generic sections do not mean every OBD mode is supported by this PCM.

**Additional Mode 22 data** (2026-10-02): a saved `0x7E0` request / `0x7E8`
reply scan of a parked, engine-off PCM with Cal ID `PFD2EB000PEP6010` adds
the [Mode 22 table below](#mode-22-manufacturer-specific). A parked positive reply does
not establish sensor validity, update rate, or behavior while driving. The
Mode 01 ranges in the tables below remain the original vehicle observations.
This MDARS read-data catalog does not provide additional verified labels for
Modes 02, 05, 06, or 09. The Mode 06 and 09 descriptions below remain based on
the separate vehicle diagnostic report.

## PID inventory

### Mode 01 (standard OBD-II)

The first 44 entries have vehicle or Diagnostic Report observations. The
last three have named, positive Mode 22 counterparts but **no saved direct
Mode 01 reply**; their Mode 01 availability still needs a probe. MDARS
names are shown where the PCM data logger supplies them.

| Signal | Header | PID | Formula | Units | Observed / evidence | MDARS label(s) |
|---|---|---|---|---|---|---|
| Engine RPM | 7DF | `01 0C` | `((A*256)+B)/4` | RPM | 806 → 2367 | `ENG_RPM` |
| Vehicle speed | 7DF | `01 0D` | `A` | km/h | 13 → 58 | `VSS` |
| Coolant temp | 7DF | `01 05` | `A-40` | °C | 80 → 89 (warming) | `ECT` |
| Coolant temp 1 & 2 | 7DF | `01 67` | `B-40` (byte B) | °C | mirrors 0x05 | `ECT2` |
| Throttle position | 7DF | `01 11` | `A*100/255` | % | 13 → 92 | `TP1` |
| Engine load | 7DF | `01 04` | `A*100/255` | % | 14 → 99 | `LOAD_CALC` |
| Absolute load | 7DF | `01 43` | `((A*256)+B)*100/255` | % | 11 → 55 | `LOAD` |
| Intake air temp | 7DF | `01 0F` | `A-40` | °C | 31 → 35 | `IAT` |
| Ambient air temp | 7DF | `01 46` | `A-40` | °C | 27 → 35 | `AMB_AIR_TEMP` |
| MAF airflow | 7DF | `01 10` | `((A*256)+B)/100` | g/s | 2.5 → 43.8 | `MAF` |
| Intake manifold pressure | 7DF | `01 0B` | `A` | kPa | 19 → 94 | `MAP` |
| Barometric pressure | 7DF | `01 33` | `A` | kPa | 101 (constant) | `BARO` |
| Ignition timing advance | 7DF | `01 0E` | `(A/2)-64` | °BTDC | 7.5 → 34.5 | `SPARK_ADV` |
| Fuel/Air **commanded** equiv ratio (λ) | 7DF | `01 44` | `((A*256)+B)*2/65536` | — | 0.91 → 2.00 | `EQUIVALENCE_RATIO_COMD` |
| Fuel rail gauge pressure | 7DF | `01 23` | `((A*256)+B)*10` | kPa | 9540 → 25110 | `FRP` |
| Fuel rail pressure abs | 7DF | `01 59` | `((A*256)+B)*10` | kPa | 10080 → 25590 | `FUEL_PRES` |
| Fuel tank level | 7DF | `01 2F` | `A*100/255` | % | matches dash gauge | `FLI` |
| Engine fuel rate | 7DF | `01 5E` | `((A*256)+B)*0.05` | L/h | 0.9 → 10.5 | — |
| Engine/Vehicle fuel rate | 7DF | `01 9D` | `((A*256)+B)/50` | g/s | 0.45 → 6.05 | — |
| Engine exhaust flow rate | 7DF | `01 9E` | `((A*256)+B)/20` | kg/h | 11 → 120 | `EX_FLOW_RATE` |
| Catalyst temp B1S1 | 7DF | `01 3C` | `((A*256)+B)/10-40` | °C | 365 → 622 | `CAT_TEMP` |
| Battery voltage | 7DF | `01 42` | `((A*256)+B)/1000` | V | 12.5 (off) / 13.8 (on) | `VPWR` |
| Actual engine torque | 7DF | `01 62` | `A-125` | % | 10 → 78 | — |
| Engine reference torque | 7DF | `01 63` | `(A*256)+B` | Nm | 250 (constant) | — |
| Engine friction torque | 7DF | `01 8E` | `A-125` | % | 7 → 10 | — |
| Accel pedal position D | 7DF | `01 49` | `A*100/255` | % | 16 → 47 | `APP1` |
| Accel pedal position E | 7DF | `01 4A` | `A*100/255` | % | 7.8 → 22.7 | `APP2` |
| Relative throttle position | 7DF | `01 45` | `A*100/255` | % | 1.96 → 30 | `TP_RELAT` |
| Absolute throttle position B | 7DF | `01 47` | `A*100/255` | % | 12 → 39.6 | `TP2` |
| Commanded throttle actuator | 7DF | `01 4C` | `A*100/255` | % | 0 → 17 | `THRO_ACTAT_COMD` |
| Transmission gear status + ratio | 7DF | `01 A4` | see [Gear Detection](#gear-detection--01-a4) | — | 1.594 / 2.035 / 2.991 / 3.127 | — |
| Vehicle odometer | 7DF | `01 A6` | `((A*256³)+(B*256²)+(C*256)+D)/10` | km | drift matched real | — |
| **O2 lambda wide-range, actual (B1S1)** | 7DF | `01 34` | λ = `((A*256)+B)*2/65536`; current mA = `((C*256)+D)/256-128` | — / mA | λ 0.995, −0.004 mA | `A/F_SEN_EQUIVALENCE_RATIO`; `A/F_SEN_CUR` |
| Short-term fuel trim, bank 1 | 7DF | `01 06` | `(A-128)*100/128` | % | 3.91 | `SHRT_FUEL_TRIM` |
| Long-term fuel trim, bank 1 | 7DF | `01 07` | `(A-128)*100/128` | % | 0.78 | `LONG_FUEL_TRIM` |
| **Intake air temp, sensors 1 & 2** | 7DF | `01 68` | A = support bitmap (`3`); IAT1 = `B-40`; IAT2 = `C-40` | °C | 36.0 / 46.0 | `IAT1`; `IAT2` |
| Fuel rail pressure control (A) | 7DF | `01 6D` | A = support (`7`); commanded, actual, and **rail temp** follow | kPa / °C | rail temp 56 °C | `FUEL_PRES_COMD`; `FUEL_RAIL_TEMP` |
| O2 sensor B1S2 voltage + STFT | 7DF | `01 15` | V = `A/200`; STFT = `(B-128)*100/128` | V / % | 0.73 V, 0% | `HO2S_OUT_VOLT` |
| Secondary O2 trim, short-term B1 | 7DF | `01 55` | `(A-128)*100/128` | % | 0 | `SHRT_FUEL_TRIM_HO2S_SEN` |
| Secondary O2 trim, long-term B1 | 7DF | `01 56` | `(A-128)*100/128` | % | 0 | `LONG_FUEL_TRIM_HO2S_SEN` |
| Time since engine start | 7DF | `01 1F` | `(A*256)+B` | sec | 1402 | — |
| Fuel system 1 / 2 status | 7DF | `01 03` | bitmap | — | Closed-loop (B1); Unknown (B2 — N/A, single bank) | `FUELSYS_CL_FAULT`; `FUELSYS_OL_FAULT`; `FUELSYS_OL_DRIVE`; `FUELSYS_CL`; `FUELSYS_OL` |
| O2 sensor locations | 7DF | `01 13` | bitmap | — | B1S1 + B1S2 only | — |
| Commanded evaporative purge | 7DF | `01 2E` | `A*100/255` | % | 32.94 | — |
| Distance traveled with MIL on | 7DF | `01 21` | — | — | Mode 22 counterpart replied; Mode 01 untested | `MIL_DIST` |
| Distance since DTC clear | 7DF | `01 31` | — | — | Mode 22 counterpart replied; Mode 01 untested | `CLR_DTC_DIST` |
| Fuel-tank/EVAP pressure | 7DF | `01 32` | — | — | Mode 22 counterpart replied; Mode 01 untested | `FTP` |

### Mode 22 (manufacturer-specific)

Every PCM DID in this table has a positive reply in the saved parked,
engine-off scan. `7E0` PCM requests reply from
`7E8`; the DSC brake entry uses `760`. `A` is the first byte after
`62 HH LL`, and `U16(A,B) = 256*A+B`. A dash means the byte conversion
has not been established here. Parked replies do not prove running
validity or sensor fitment.

| Signal / MDARS label(s) | Header | PID | Formula | Units | Observed / evidence |
|---|---|---|---|---|---|
| MAP sensor voltage (`MAP_VOLT`) | 7E0 | `22 03 01` | `U16(A,B)/1024` | V | `13 F7` ≈ 4.99 V |
| `GEN_VOLT_COMD` | 7E0 | `22 03 04` | — | — | parked positive |
| Desired engine speed (`ENG_RPM_DSD`) | 7E0 | `22 03 08` | `U16(A,B)` | RPM | `03 20` = 800 RPM |
| `INJ` | 7E0 | `22 03 0B` | — | — | parked positive |
| MAF sensor voltage (`MAF`) | 7E0 | `22 03 14` | `U16(A,B)/1024` | V | `00 00` |
| Intake VVT control (`VLV_TIMING_IN`) | 7E0 | `22 03 16` | `U16(A,B)*100/32768` | % | `2C CD` ≈ 35.0% |
| Exhaust VVT control (`VLV_TIMING_EX`) | 7E0 | `22 03 17` | `U16(A,B)*100/32768` | % | `00 00` |
| Intake cam actual (`VLV_TIMING_ACT_IN`) | 7E0 | `22 03 18` | `U16(A,B)/16` | ° | `00 14` = 1.25° |
| Exhaust cam actual (`VLV_TIMING_ACT_EX`) | 7E0 | `22 03 19` | `U16(A,B)/16` | ° | `00 00` |
| Intake cam desired (`VLV_TIMING_DSD_IN`) | 7E0 | `22 03 1C` | `U16(A,B)/16` | ° | `01 90` = 25° |
| Exhaust cam desired (`VLV_TIMING_DSD_EX`) | 7E0 | `22 03 21` | `U16(A,B)/16` | ° | `00 00` |
| `FUEL_PRES` | 7E0 | `22 03 24` | — | — | parked positive |
| Processed relative accelerator pedal (`APP_RELAT`) | 7E0 | `22 03 2B` | `A/2` | % | parked `62 03 2B 00` |
| `IAT2` | 7E0 | `22 03 4E` | — | — | parked positive |
| `ECT_VOLT` | 7E0 | `22 03 57` | — | — | parked positive |
| `CHARGE_LAMP` | 7E0 | `22 03 5D` | — | — | parked positive |
| `TP_VOLT_CLOSE_STOP1` | 7E0 | `22 03 66` | — | — | parked positive |
| `TP_VOLT_CLOSE_STOP2` | 7E0 | `22 03 67` | — | — | parked positive |
| Fuel pulse-width report (`FUEL_PULSE_WD`) | 7E0 | `22 03 6C` | `U16(A,B)*8` | µs | `00 00` |
| `GEN_DUTY_ACT` | 7E0 | `22 03 78` | — | — | parked positive |
| `FUEL_CAP_LAMP` | 7E0 | `22 03 89` | — | — | parked positive |
| `EVAP_PRES` | 7E0 | `22 03 8E` | — | — | parked positive |
| `A/C_PRES_BEF_EXP_VLV` | 7E0 | `22 03 93` | — | — | parked positive |
| A/F sensor heater duty (`A/F_SEN_HEAT`) | 7E0 | `22 03 A1` | `A*100/255` | % | `00` |
| `HO2S_HEAT` | 7E0 | `22 03 A2` | — | — | parked positive |
| `A/F_SEN_HEAT`; `HO2S_HEAT` | 7E0 | `22 03 A9` | — | — | parked positive |
| `FAN` | 7E0 | `22 03 B9` | — | — | parked positive |
| `HO2S_FUEL_TRIM` | 7E0 | `22 03 BD` | — | — | parked positive |
| `FUEL_PUMP_DSD` | 7E0 | `22 03 DC` | — | — | parked positive |
| Learned knock (`KNOCK_LRN`) | 7E0 | `22 03 E8` | `U16(A,B)/16384` (scalar) | — | `00 00` |
| Reported knock retard (`KNOCK_RTD`) | 7E0 | `22 03 EC` | `U16(A,B)/512` | ° | `00 00` |
| `MISFIRE_CYL1_CAT`; `MISFIRE_CYL2_CAT`; `MISFIRE_CYL3_CAT`; `MISFIRE_CYL4_CAT`; `MISFIRE_CYL5_CAT`; `MISFIRE_CYL6_CAT`; `MISFIRE_CYL1_EMI`; `MISFIRE_CYL2_EMI`; `MISFIRE_CYL3_EMI`; `MISFIRE_CYL4_EMI`; `MISFIRE_CYL5_EMI`; `MISFIRE_CYL6_EMI`; `MISFIRE_FIRCYC_COUNT_CAT`; `MISFIRE_FIRCYC_COUNT_EMI`; `MISFIRE_CAT_TOTAL`; `MISFIRE_EMI_TOTAL` | 7E0 | `22 05 3B` | — | — | parked positive |
| `AMB_AIR_TEMP_RAW` | 7E0 | `22 05 7D` | — | — | parked positive |
| `OIL_PUMP_DUTY` | 7E0 | `22 05 94` | — | — | parked positive |
| `APP1` | 7E0 | `22 09 14` | — | — | parked positive |
| `APP2` | 7E0 | `22 09 15` | — | — | parked positive |
| `TP1` | 7E0 | `22 09 17` | — | — | parked positive |
| `TP2` | 7E0 | `22 09 18` | — | — | parked positive |
| Commanded throttle angle (`THRO_ANG_COMD`) | 7E0 | `22 09 1A` | `U16(A,B)/512` | ° | `0C 0A` ≈ 6.02° |
| `APP_MODE` | 7E0 | `22 09 1B` | — | — | parked positive |
| Reported throttle angle (`THRO_ANG_ACT`) | 7E0 | `22 09 3C` | `U16(A,B)/512` | ° | `BC F0`; engine-off value needs validity check |
| `A/C_REQ`; `L_SPD_FAN`; `H_SPD_FAN`; `A/C_SW`; `M_SPD_FAN` | 7E0 | `22 09 9B` | — | — | parked positive |
| `PRG_DUTY` | 7E0 | `22 11 66` | — | — | parked positive |
| `CSTR_VENT_SOL_DUTY` | 7E0 | `22 11 67` | — | — | parked positive |
| `IAT` | 7E0 | `22 12 79` | — | — | parked positive |
| Engine oil temperature (`OIL_TEMP`) | 7E0 | `22 13 10` | `(U16(A,B)/100)-40` | °C | 66.7 → 78.4 °C |
| `ALT_FIELD_CTRL_DUTY_OUT` | 7E0 | `22 16 E8` | — | — | parked positive |
| `ALT_VOLT_OUT_ACT` | 7E0 | `22 16 E9` | — | — | parked positive |
| `MANUAL_GEAR_POS` | 7E0 | `22 19 85` | — | — | parked positive |
| `INGEAR`; `REV_SW` | 7E0 | `22 1E 04` | — | — | parked positive |
| `BRK_SW2`; `BRK_SW1` | 7E0 | `22 2B 00` | — | — | parked positive |
| `FUEL_PUMP_REQ` | 7E0 | `22 41 33` | — | — | parked positive |
| `MIL` | 7E0 | `22 61 A5` | — | — | parked positive |
| `A/C_PRES_BEF_EXP_VLV` | 7E0 | `22 98 00` | — | — | parked positive |
| `OIL_PRES_SOL` | 7E0 | `22 DA 01` | — | — | parked positive |
| `THERMOSTAT` | 7E0 | `22 DA 8B` | — | — | parked positive |
| `THERMOSTAT_MIN` | 7E0 | `22 DA 8C` | — | — | parked positive |
| `THERMOSTAT_MAX` | 7E0 | `22 DA 8D` | — | — | parked positive |
| Fuel amount report (`FUEL_INJ_AMOUNT`) | 7E0 | `22 DA 99` | `U16(A,B)/32768` | g/cylinder | `00 00` |
| `FUELSYS_CL_FAULT`; `FUELSYS_OL_FAULT`; `FUELSYS_OL_DRIVE`; `FUELSYS_CL`; `FUELSYS_OL` | 7E0 | `22 F4 03` | — | — | parked positive |
| `LOAD_CALC` | 7E0 | `22 F4 04` | — | — | parked positive |
| `ECT` | 7E0 | `22 F4 05` | — | — | parked positive |
| `SHRT_FUEL_TRIM` | 7E0 | `22 F4 06` | — | — | parked positive |
| `LONG_FUEL_TRIM` | 7E0 | `22 F4 07` | — | — | parked positive |
| `MAP` | 7E0 | `22 F4 0B` | — | — | parked positive |
| `ENG_RPM` | 7E0 | `22 F4 0C` | — | — | parked positive |
| `VSS` | 7E0 | `22 F4 0D` | — | — | parked positive |
| `SPARK_ADV` | 7E0 | `22 F4 0E` | — | — | parked positive |
| `IAT` | 7E0 | `22 F4 0F` | — | — | parked positive |
| `MAF` | 7E0 | `22 F4 10` | — | — | parked positive |
| `TP1` | 7E0 | `22 F4 11` | — | — | parked positive |
| `HO2S_OUT_VOLT` | 7E0 | `22 F4 15` | — | — | parked positive |
| `MIL_DIST` | 7E0 | `22 F4 21` | — | — | parked positive |
| `FRP` | 7E0 | `22 F4 23` | — | — | parked positive |
| `FLI` | 7E0 | `22 F4 2F` | — | — | parked positive |
| `CLR_DTC_DIST` | 7E0 | `22 F4 31` | — | — | parked positive |
| `FTP` | 7E0 | `22 F4 32` | — | — | parked positive |
| `BARO` | 7E0 | `22 F4 33` | — | — | parked positive |
| `A/F_SEN_EQUIVALENCE_RATIO`; `A/F_SEN_CUR` | 7E0 | `22 F4 34` | see notes below | — | parked positive |
| `CAT_TEMP` | 7E0 | `22 F4 3C` | — | — | parked positive |
| `VPWR` | 7E0 | `22 F4 42` | — | — | parked positive |
| `LOAD` | 7E0 | `22 F4 43` | — | — | parked positive |
| `EQUIVALENCE_RATIO_COMD` | 7E0 | `22 F4 44` | — | — | parked positive |
| `TP_RELAT` | 7E0 | `22 F4 45` | — | — | parked positive |
| `AMB_AIR_TEMP` | 7E0 | `22 F4 46` | — | — | parked positive |
| `TP2` | 7E0 | `22 F4 47` | — | — | parked positive |
| `APP1` | 7E0 | `22 F4 49` | — | — | parked positive |
| `APP2` | 7E0 | `22 F4 4A` | — | — | parked positive |
| `THRO_ACTAT_COMD` | 7E0 | `22 F4 4C` | — | — | parked positive |
| `SHRT_FUEL_TRIM_HO2S_SEN` | 7E0 | `22 F4 55` | — | — | parked positive |
| `LONG_FUEL_TRIM_HO2S_SEN` | 7E0 | `22 F4 56` | — | — | parked positive |
| `FUEL_PRES` | 7E0 | `22 F4 59` | — | — | parked positive |
| `ECT2` | 7E0 | `22 F4 67` | — | — | parked positive |
| `IAT1`; `IAT2` | 7E0 | `22 F4 68` | see notes below | — | parked positive |
| `FUEL_PRES_COMD`; `FUEL_RAIL_TEMP` | 7E0 | `22 F4 6D` | see notes below | — | parked positive |
| `EX_FLOW_RATE` | 7E0 | `22 F4 9E` | — | — | parked positive |
| Brake position (DSC) | 760 | `22 2B 0D` | `max(0, signed_int16(A,B))/2.3` | % | 0 → 0.43 (light tap) |
| Steering angle hypothesis (DSC) | 760 | `22 20 33` | tentative `signed16(A,B)/10` | °? | `00 00` centered; `FF FF`/`00 00` engine off |
| Steering rate or alternate angle hypothesis (DSC) | 760 | `22 20 34` | tentative `signed16(A,B)/10` | — | `00 00` stationary |
| DSC status, meaning unknown | 760 | `22 20 1D` | — | — | `04` stationary |
| DSC status, meaning unknown | 760 | `22 20 1F` | — | — | `00` stationary |

### Notes on the PID tables

The first 44 Mode 01 values above are original vehicle observations. The three
untested Mode 01 candidates have `F421`, `F431`, and `F432` Mode 22 counterparts
that replied in the parked scan. A positive Mode 22 reply does not establish a
Mode 01 response. The `F4xx` rows in the Mode 22 table correspond to mapped
Mode 01 PIDs `xx` and preserve MDARS's exact labels.

`22 13 10` (`OIL_TEMP`) reports a modeled engine oil temperature, not a direct
measurement from an oil-temperature sensor.

`01 34` reports processed B1S1 wideband lambda, while `01 44` is commanded
lambda. Polling both gives a reported-versus-commanded comparison, but their
difference is not necessarily the controller's live error. `01 68` exposes
two IAT fields; their physical locations and fallback behavior need validation
before treating the difference as intake heat soak. `01 33` barometric
pressure and `01 63` reference torque were constant in the original short
drive, as expected for those observations.

The Mode 22 VVT percentages (`0316/0317`), desired angles (`031C/0321`), and
actual angles (`0318/0319`) are different signals. A zero or unchanged angle
need not be a fresh zero-degree measurement. `036C` reports fuel pulse width
and `DA99` reports fuel amount; parked replies do not identify a
cylinder-specific injector event. `22 03 2B` reports processed relative
pedal position; `01 49` reports pedal position D. The `760` brake PID belongs
to the DSC, not the PCM. The four DSC `22 20 XX` entries are raw stationary
observations with tentative meanings. Five engine-off snapshots did not vary
when the clutch and shifter moved; an engine-on steering turn test is still
needed before labeling `22 20 33` as steering angle.

For composite `F434` / `01 34`, data bytes `A,B` report lambda at about
`U16(A,B)*2/65536` and `C,D` report current at `U16(C,D)/256-128` mA.
`F46D` / `01 6D` uses status byte `A`, commanded rail pressure in `B,C` at
`U16(B,C)*10` kPa, and rail temperature in `F` at `F-40` °C; that
reported temperature alone does not prove a dedicated sensor. `F468` /
`01 68` uses status byte `A`, IAT1 in `B`, and IAT2 in `C`, each at
`byte-40` °C. Check operating state before treating them as measured air.

MDARS `053B` names misfire fields for cylinders 1–6; the generic names for
5–6 do not establish six cylinders on this four-cylinder MX-5. `0393` and
`9800` share an A/C-pressure label but report different quantities (kPa and
sensor voltage). Composite status DIDs `099B`, `1E04`, and `2B00` still need
byte/bit decoding. Catalog-only MDARS entries without a positive saved PCM
reply are excluded.

The original range scan found 8 replies in `22 09 XX`, 1 in `22 13 XX`,
60 in `22 F4 XX`, 36 in `22 DA XX`, and 38 in `22 03 XX`; the DSC answered
11 in `760/22 2B XX`. These are response counts, not a decoded-signal count.
Full raw results are in
[`pids/range-scan-2026-06-23.md`](pids/range-scan-2026-06-23.md).
Mode 06 monitor `$35` has six VVT results with units and limits that may
help check the eight `22 09 XX` responses.

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

## Mode $02 — Freeze Frame

The PCM implements the Mode 02 freeze-frame request path, which selects a
standard PID and freeze-frame record. We do not have a saved, decoded Mode 02
response for this car, so there is no verified ND3 freeze-frame PID list or
value to add here. A Mode 01 PID's meaning does not by itself prove that its
Mode 02 counterpart has a stored record.

## Mode $05 — Oxygen Sensor Monitoring

The PCM's diagnostic service list does not include Mode 05. We have no positive
Mode 05 response or supported PID to list. The generic Mode 05 section in an
OBDLink report is not evidence that this ND3 PCM implements it; the captured
oxygen-sensor monitoring results are in Mode 06 below.

## Mode $06 — On-Board Monitoring

Mode $06 returns **measured monitor values with min/max thresholds** in addition
to pass/fail or incomplete status.

See [`docs/mode-06-monitoring.md`](docs/mode-06-monitoring.md) for the full TID
table and a captured catalyst-failure example.

Concrete captured results include `$02` TIDs `$05`/`$06` (B1S2 rich-to-lean
and lean-to-rich switch times), `$35` TIDs `$80`–`$85` (VVT monitor results),
`$41`/`$42` TID `$80` (oxygen-sensor heater resistance), and `$A2`–`$A5`
TIDs `$0B`/`$0C` (per-cylinder misfire records). The captured result values,
limits, and units are in the linked Mode 06 table.

Monitor `$21` TID `$82` measured `0.60` against a `0.67` minimum when this car
had P0421. Interpreting this manufacturer-defined result as catalyst **oxygen
storage capacity** is a strong inference from that case, not an MDARS label:

```
$21 - Catalyst Monitor Bank 1
  TID $82    value 0.60    min 0.67    max 655.35    FAIL
```

These are monitor/test IDs, not Mode 01 or Mode 22 PIDs.

## Mode $09 — Vehicle Information and In-Performance Tracking

Direct read-only requests returned:

| Request | Confirmed response | Meaning |
|---|---|---|
| `09 00` | `49 00 55 40 37 8F` | Supported Mode 09 information-type bitmap |
| `09 02` | `49 02 01 ...` | VIN `JM1NDAL78R0605874` in a separate connection capture |
| `09 04` | `49 04 01 ...` | Calibration ID `PFD2EB000PEP6010` |
| `09 06` | `49 06 01 EC 50 4E A3` | Calibration verification number `EC504EA3` |

The OBDLink Diagnostic Report also decoded in-performance tracking counters,
including monitor completion/condition counts and lifetime/recent totals:

- Ignition cycle counter (recent / lifetime)
- Total engine run time, total idle run time
- Total distance traveled, total fuel consumed
- Positive kinetic energy (km/hr²)
- Engine output energy (kWh)
- Catalyst / O2 / EGR-VVT monitor completion counts

Most "Active Off-Cycle Credit" / "Driver Coaching" / "Grille Shutter" timers
return `4294967295` (`0xFFFFFFFF` = not supported) — expected on this platform.

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
- Decode the remaining `22 F4 XX` PIDs
- Cross-check `22 09 XX` cam/VVT decodes against Mode $06 `$35` TIDs
- Find correct EPS address (try `0x7E2`, `0x720`, physical addressing)
- Validate the `01 6D` status bits and unnamed bytes 3–4 under running pressure

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
