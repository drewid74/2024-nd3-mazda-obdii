# OBD-II Response Prefix Fix

## Background

This bug affects any OBD app that mixes Mode 01 and Mode 22 PIDs in the same
probe pass. It manifests as Mode 01 PIDs always returning NO_RESPONSE despite
the ECU replying correctly. This was discovered during ND3 validation because
the ND3 was the first car tested with a mixed-mode candidate list.

## The Bug

The OBD-II positive response SID is defined by ISO 14229 / SAE J1979 as:

```
response_SID = request_SID + 0x40
```

Apps that only ever implemented Mode 22 often hardcode the expected prefix:

```kotlin
// WRONG — assumes every request is Mode 22
val expectedPrefix = "62" + candidate.pidHex.replace(" ", ").takeLast(4)
```

`0x62` is the positive response for Mode `0x22` (`0x22 + 0x40 = 0x62`).
When you send a Mode 01 request (`0x01 0C` for RPM), the ECU correctly
responds with `0x41 0C <data>`. The parser looks for "620C"`, never finds
it, and reports NO_RESPONSE — even though the bytes are sitting right there.

## The Fix

Compute the expected prefix generically from the request SID:

```kotlin
private fun computeExpectedPrefix(pidHex: String): String {
    val clean = pidHex.replace(" ", ").uppercase()
    if (clean.length < 4) return clean
    val serviceId = clean.substring(0, 2).toInt(16)
    val responseService = (serviceId + 0x40).toString(16).uppercase().padStart(2, 
'
0
'
)
    val pidBytes = clean.substring(2)
    return responseService + pidBytes
}
```

Mode reference:

| Request SID | Mode | Response SID |
|---|---|---|
| `0x01` | Live data | `0x41` |
| `0x02` | Freeze frame | `0x42` |
| `0x03` | DTCs | `0x43` |
| `0x09` | Vehicle info (VIN etc.) | `0x49` |
| `0x22` | Manufacturer-specific | `0x62` |

## What Was Not Affected

- Negative response detection (`0x7F mode NRC`) — already mode-agnostic
- ISO-TP frame reassembly — handled by the STN chip (OBDLink MX+) before
  bytes reach the app; the payload that hits the parser always starts at
  the service ID byte regardless of frame count
- Source address / header filtering — separate path, not touched

## Scope

This is a **portable fix**, not an ND3-specific one. Any app mixing OBD modes
against any car will exhibit this bug. The ND3 exposed it because it was the
first car probed with a mixed Mode 01 + Mode 22 candidate list.
