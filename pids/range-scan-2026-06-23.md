# Range Scan Results — 2026-06-23

Brute-force Mode 22 PID discovery on ND3 (2025 MX-5 RF Club, Cal ID
`PFD2EB000PEP6010`, Engine Family `STKXV02.0FFB`).

Methodology: 11 range scans of 256 PIDs each, probed sequentially via OBDLink
MX+ over Bluetooth SPP. Vehicle stationary, engine running, steering centered,
foot off pedal. Total scan time: 388 seconds.

Hit = positive Mode 22 response (`62 PID DATA`). NACK = negative response
(`7F 22 NRC`). No-response = silence after timeout.

## Summary

| Range | Hits | Description |
|-------|------|-------------|
| `7E0/22 09 XX` | 8 | PCM cam / VVT / valve cluster |
| `7E0/22 13 XX` | 1 | PCM thermal (only `13 10` oil temp) |
| `7E0/22 F4 XX` | **60** | PCM fuel / boost / MAP — biggest trove |
| `7E0/22 DA XX` | 36 | PCM service / fuel learning |
| `7E0/22 03 XX` | 38 | PCM engine control |
| `760/22 2B XX` | 11 | DSC chassis cluster |
| `760/22 20 XX` | 4 | DSC steering cluster |
| `760/22 21 XX` | 0 | (yaw/lateral G hypothesis — wrong) |
| `760/22 06 XX` | 0 | (wheel speeds hypothesis — wrong) |
| `730/22 20 XX` | 0 | (EPS module — wrong header) |
| `730/22 06 XX` | 0 | (EPS alt — wrong header) |

## PCM Cam / VVT cluster (`7E0/22 09 XX`)

```
22 09 14  →  0319    (793 — possibly cam timing degrees)
22 09 15  →  018E    (398)
22 09 17  →  0294    (660 — possibly VVT intake)
22 09 18  →  1181    (4481 — possibly VVT exhaust)
22 09 1A  →  0627    (1575)
22 09 1B  →  FF      (255)
22 09 3C  →  061C    (1564) — ND2 candidate was "intake shutter valve angle"; needs validation
22 09 9B  →  0000
```

## PCM Thermal cluster (`7E0/22 13 XX`)

```
22 13 10  →  21BC    (oil temp ((A*256)+B)/100 - 40 = 46.84°C cold start)
```

Only one PID in this cluster on ND3. Other thermal cluster addresses (oil
change interval, etc.) silent.

## PCM Fuel / Boost / MAP cluster (`7E0/22 F4 XX`) — 60 PIDs

This is the **biggest discovery surface** on the ND3. Many of these likely
mirror standard Mode 01 PIDs but may give richer multi-channel packets:

```
22 F4 00  →  FE3FA813              (4-byte)
22 F4 01  →  0007E500              (4-byte)
22 F4 02  →  0000
22 F4 03  →  0200
22 F4 04  →  45                    (69 — possible)
22 F4 05  →  6B                    (107)
22 F4 06  →  83                    (131)
22 F4 07  →  85                    (133)
22 F4 0B  →  24                    (36 — possibly MAP kPa? cf 01 0B)
22 F4 0C  →  0BBB                  (3003 — possibly RPM ÷ ??? or fuel rail pressure)
22 F4 0D  →  00
22 F4 0E  →  94                    (148 — possible)
22 F4 0F  →  46                    (70)
22 F4 10  →  0113                  (275 — possibly MAF mirror)
22 F4 11  →  21                    (33 — ND2 candidate was "intake shutter valve %")
22 F4 13  →  03                    (3)
22 F4 15  →  0E80
22 F4 1C  →  03
22 F4 1F  →  00D6
22 F4 20  →  A007F011              (4-byte)
22 F4 21  →  0000
22 F4 23  →  03EF                  (1007 — possibly fuel rail pressure kPa)
22 F4 2E  →  0B                    (11)
22 F4 2F  →  6E                    (110)
22 F4 30  →  3C                    (60 — possibly coolant via Mode 22)
22 F4 31  →  0572                  (1394)
22 F4 32  →  013D                  (317)
22 F4 33  →  65                    (101 — BARO via Mode 22, matches 01 33)
22 F4 34  →  844C802E              (4-byte)
22 F4 3C  →  08EA                  (2282)
22 F4 40  →  FED08C85              (4-byte)
22 F4 41  →  0007E5A5              (4-byte)
22 F4 42  →  360F                  (13839)
22 F4 43  →  002E                  (46)
22 F4 44  →  8480                  (33920)
22 F4 45  →  07                    (7)
22 F4 46  →  46                    (70)
22 F4 47  →  20                    (32)
22 F4 49  →  27                    (39)
22 F4 4A  →  14                    (20)
22 F4 4C  →  09                    (9)
22 F4 51  →  01                    (1)
22 F4 55  →  80                    (128)
22 F4 56  →  80                    (128)
22 F4 59  →  03EA                  (1002)
22 F4 5E  →  0011                  (17 — possibly fuel rate via Mode 22)
22 F4 60  →  6B080001              (4-byte)
22 F4 62  →  89                    (137 — possibly engine torque % via Mode 22)
22 F4 63  →  00FA                  (250 — matches 01 63 reference torque!)
22 F4 65  →  1000
22 F4 67  →  016C00                (3-byte)
22 F4 68  →  0346481:00000000000000  (multi-frame)
22 F4 6D  →  0703E81:03DE54000000002:00000000000000  (multi-frame)
22 F4 80  →  0004000D
22 F4 8E  →  87                    (135 — possibly friction torque via Mode 22)
22 F4 9D  →  00080008              (4-byte — possibly fuel rate dual encoding)
22 F4 9E  →  00FC                  (252)
22 F4 A0  →  14000000              (4-byte)
22 F4 A4  →  030013DF              (4-byte — possibly gear ratio mirror)
22 F4 A6  →  000153C9              (4-byte — possibly odometer mirror)
```

Note `22 F4 XX` mirrors many Mode 01 standard PIDs (33 baro, 63 ref torque,
A4 gear, A6 odometer). This is consistent with Mazda using `22 F4 XX` as an
internal Mode 22 alias of standard Mode 01 data.

## PCM Service / Fuel learning cluster (`7E0/22 DA XX`)

Mostly slow-moving / config data. Notable: `DA 9E` contains CalID ASCII
fragments confirming engine cal `PFD2EB000PEP6010`.

```
22 DA 00  →  00
22 DA 01  →  02
22 DA 81  →  00
22 DA 82  →  00
22 DA 83  →  0000
22 DA 84  →  0000
22 DA 85  →  84
22 DA 86  →  85
22 DA 87  →  00
22 DA 88  →  00
22 DA 89  →  00
22 DA 8A  →  00
22 DA 8B  →  00000491
22 DA 8C  →  00000190
22 DA 8D  →  0000FFFF
22 DA 8E  →  2440031:80000064000000   (multi-frame)
22 DA 8F  →  2440031:80641B48000000   (multi-frame)
22 DA 90  →  2440031:9FB00064000000   (multi-frame)
22 DA 91  →  2440031:A0141B48000000   (multi-frame)
22 DA 93  →  1F
22 DA 94  →  0000
22 DA 95  →  0000
22 DA 99  →  00D1
22 DA 9B  →  0064
22 DA 9E  →  01FFFF1:FFFFFFFFFF5046...  (CalID ASCII fragments — PFD2EB000PEP6010)
22 DA 9F  →  0001621:0001620001...     (multi-frame)
22 DA A0  →  00008000
22 DA A6  →  64646464
22 DA A7  →  00
22 DA A8  →  0233
22 DA B3  →  1710261:0D0D000001F400
22 DA B4  →  FF
22 DA B5  →  ... (multi-frame)
22 DA B6  →  ...
22 DA B8  →  00
22 DA BA  →  ...
```

ND2 candidate `22 DA A1` (timing chain elongation) NOT present — ND3 chain
monitoring may have moved.

## PCM Engine Control cluster (`7E0/22 03 XX`)

```
22 03 01  →  05A5
22 03 04  →  703E
22 03 08  →  02EE
22 03 0B  →  00
22 03 14  →  0788
22 03 16  →  0CCD
22 03 17  →  0000
22 03 18  →  018A
22 03 19  →  0000
22 03 1C  →  0188
22 03 21  →  0000
22 03 24  →  012B
22 03 26  →  00
22 03 2B  →  00      (CONFIRMED: accel pedal % = A/2)
22 03 4E  →  0273
22 03 57  →  00D5
22 03 5D  →  00000000
22 03 5F  →  004A
22 03 66  →  0065
22 03 67  →  03A0
22 03 6C  →  0071
22 03 78  →  0000
22 03 89  →  40000000
22 03 8E  →  0A33
22 03 93  →  0231
22 03 A1  →  52
22 03 A2  →  6F
22 03 A9  →  C0000000
22 03 AB  →  00
22 03 AC  →  00FF
22 03 AD  →  00000000
22 03 B5  →  13FF
22 03 B9  →  0000
22 03 BA  →  01C9
22 03 BD  →  9A48      (possibly knock retard / calibration target)
22 03 DC  →  03E8      (1000)
22 03 E8  →  1151
22 03 EC  →  0000
```

## DSC Chassis cluster (`760/22 2B XX`)

```
22 2B 00  →  20000000
22 2B 05  →  40000000
22 2B 06  →  00
22 2B 07  →  00
22 2B 08  →  00
22 2B 09  →  00
22 2B 0B  →  0001
22 2B 0C  →  FFFF
22 2B 0D  →  FFFF      (CONFIRMED: brake position % — FFFF=no-pressure marker)
22 2B 11  →  0046
22 2B 2C  →  00000000
```

## DSC Steering cluster (`760/22 20 XX`)

```
22 20 1D  →  04        (status flag)
22 20 1F  →  00        (status flag)
22 20 33  →  0000      (steering angle hypothesis — centered)
22 20 34  →  0000      (steering rate hypothesis — static)
```

See main README for steering decode hypothesis + engine-on validation plan.

## Dead ends (0 hits across 256-PID scans)

- `760/22 21 XX` — yaw / lateral G NOT here
- `760/22 06 XX` — wheel speeds NOT exposed at this DSC sub-address
- `730/22 20 XX` — EPS module silent
- `730/22 06 XX` — EPS alt silent

## Tools used

- **OBDLink MX+** (Bluetooth Classic SPP) — purchase: https://www.obdlink.com/mxp/
- **Delta7 Android app** — open-source telemetry capture: https://github.com/drewid74/delta7 (private; reach out for code access)
- **Probe screen**: implements range-scan + per-PID auto-decode + raw byte logging
