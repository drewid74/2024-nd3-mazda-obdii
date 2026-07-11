# Mode $06 — On-Board Monitoring Test Results (ND3)

**Vehicle:** 2025 MX-5 RF Club (ND3), Cal ID `PFD2EB000PEP6010`
**Adapter:** OBDLink MX+
**Captured:** 2026-07-10, 5,862 mi / 9,434 km

---

## Why you should care

Mode $06 is the OBD-II service that returns **on-board monitoring test results**.
Unlike readiness flags (which are just "complete / not complete") and unlike DTCs
(which only exist *after* something has already failed), Mode $06 returns the
**actual measured value** each monitor computed, together with the **min/max
thresholds** the ECU compares it against.

That means Mode $06 lets you watch a component degrade **before it sets a code**.

Consumer OBD apps essentially ignore this service. To our knowledge it has not
been documented for the ND3 platform before.

**Response prefix:** Mode $06 requests return SID `0x46` (request `0x06` + `0x40`).
See [`prefix-fix.md`](prefix-fix.md) — the same rule that catches people out on
mixed Mode 01 / Mode 22 apps applies here.

---

## Worked example: catching a catalyst failure before the MIL

This car developed a **P0421** (Catalyst Efficiency Below Threshold, Bank 1).
Root cause: **leaded race fuel** (Sunoco Supreme 112 is leaded; tetraethyl lead
is a catalyst poison) run at track events, which poisoned the OEM converter at
only 5,862 miles.

Here is what Mode $06 showed:

```
$21 - Catalyst Monitor Bank 1
  TID $82 - Manufacturer Defined
    value 0.60    min 0.67    max 655.35    g    FAIL
```

`TID $82` is the catalyst's **oxygen storage capacity (OSC)** — the physical
property the catalyst monitor actually thresholds against. The measured `0.60`
sits about 10% under the `0.67` floor.

The point: **this is a continuous number, not a pass/fail bit.** Had it been
logged at each track session, the decline would have been visible as a trend
long before the check-engine light came on. That is the entire argument for
putting Mode $06 into a telemetry app.

### Diagnostic bonus — it also tells you what *isn't* broken

In the same capture, every other monitor passed:

- Both O2 sensors responding normally (see `$02` switch times below)
- O2 heater circuits in range (`$41` / `$42`)
- VVT within bounds (`$35`)
- Zero misfires on all four cylinders (`$A2`–`$A5`)

This is how you distinguish **a genuinely dead catalyst** from **a lazy
downstream O2 sensor faking a catalyst fault**. A glazed/lazy B1S2 sensor makes
a cat look *good*, not bad — so a P0421 with *fast, healthy* B1S2 switch times
means the substrate really is spent. Mode $06 gives you that discrimination
directly; readiness flags and DTCs do not.

---

## Full monitor set observed on ND3

Scaling for manufacturer-defined TIDs is inferred from the reported min/max
bounds and units. Treat the `$01` and `$21` TID semantics as best-effort.

### `$01` — Exhaust Gas Sensor Monitor, Bank 1 Sensor 1 (wideband)

| TID | Value | Min | Max | Notes |
|-----|-------|-----|-----|-------|
| `$80` | 0.0508 | 0 | 1 | Manufacturer defined |
| `$81` | 0.0117 | 0 | 1 | Manufacturer defined |
| `$82` | 0.2969 | 0 | 1 | Manufacturer defined |
| `$83` | 0 | 0 | 1 | Manufacturer defined |

### `$02` — Exhaust Gas Sensor Monitor, Bank 1 Sensor 2 (post-cat narrowband)

| TID | Description | Value | Min | Max | Units |
|-----|-------------|-------|-----|-----|-------|
| `$03` | Low sensor voltage for switch-time calc (constant) | 0.30 | 0.30 | 0.30 | V |
| `$04` | High sensor voltage for switch-time calc (constant) | 0.55 | 0.55 | 0.55 | V |
| `$05` | **Rich→lean switch time** (calculated) | 0.015 | 0 | 1 | sec |
| `$06` | **Lean→rich switch time** (calculated) | 0.19 | 0 | 1 | sec |
| `$80` | Manufacturer defined | 0 | 0 | 1.1 | g |

> `$05` / `$06` are the sensor-response channels. A lead-poisoned or aged
> narrowband sensor goes **slow** here. If you are chasing a catalyst code,
> check these first — healthy switch times mean the cat monitor's verdict is
> trustworthy.

### `$21` — Catalyst Monitor, Bank 1

| TID | Value | Min | Max | Units | Result |
|-----|-------|-----|-----|-------|--------|
| `$82` | **0.60** | **0.67** | 655.35 | g | **FAIL** |

Oxygen storage capacity. 2-byte value, ×0.01 scaling (hence the 655.35 ceiling =
`0xFFFF` × 0.01). **This is the trendable cat-health number.**

### `$35` — VVT Monitor, Bank 1

| TID | Value | Min | Max | Units |
|-----|-------|-----|-----|-------|
| `$80` | −0.10 | −327.68 | 15 | deg |
| `$81` | −0.04 | −15 | 327.67 | deg |
| `$82` | −0.22 | −20 | 327.67 | deg |
| `$83` | 1.42 | −327.68 | 20 | deg |
| `$84` | 0.06 | 0 | 9 | sec |
| `$85` | 0.02 | 0 | 9 | sec |

Four angle errors (deg) and two response times (sec). These have real
engineering units and known bounds — potentially useful as a **free
cross-validation** for decoding the `22 09 XX` Mode 22 cam/VVT cluster.

### `$41` / `$42` — Exhaust Gas Sensor Heater Monitors

| Monitor | TID | Value | Min | Max | Units |
|---------|-----|-------|-----|-----|-------|
| `$41` (B1S1 heater) | `$80` | 0.155 | 0.025 | 0.403 | kOhm |
| `$42` (B1S2 heater) | `$80` | 0.100 | 0.005 | 0.450 | kOhm |

Heater circuit resistance. **Relevant if you relocate O2 sensors with extension
cables** (common with aftermarket headers/midpipes) — added cable resistance
stacks onto this circuit and can push it toward the limit.

### `$A2` – `$A5` — Per-Cylinder Misfire Data

| Monitor | TID | Description | Value |
|---------|-----|-------------|-------|
| `$A2` (cyl 1) | `$0B` | EWMA misfire counts, last 10 driving cycles | 0 |
| `$A2` (cyl 1) | `$0C` | Misfire counts, last/current driving cycle | 0 |
| `$A3` (cyl 2) | `$0B` / `$0C` | as above | 0 / 0 |
| `$A4` (cyl 3) | `$0B` / `$0C` | as above | 0 / 0 |
| `$A5` (cyl 4) | `$0B` / `$0C` | as above | 0 / 0 |

**Per-cylinder misfire counts as a track telemetry channel.** `$0C` is
last/current driving cycle, so it is effectively per-session. Pull it at session
end and you get a per-cylinder combustion health readout for the run — without
waiting for a misfire DTC threshold to trip.

### `$3A` / `$3B` / `$3C` — EVAP Monitors

Returned `Incomplete` in this capture (EVAP monitor had not run). Listed for
completeness — `0.090"`, `0.040"`, and `0.020"` leak checks, values in Pa.

### `$3D` — Purge Flow Monitor

| TID | Value | Units | Result |
|-----|-------|-------|--------|
| `$80` | 0 | kPa/s | Pass |
| `$81` | 0 | Pa | Pass |

---

## Proposed use: session-boundary health snapshot

For a track telemetry app, the pattern that gets the most value out of this:

1. **At session start**, pull Mode $06 + readiness status + stored DTCs.
2. **At session end**, pull the same.
3. Persist both, and trend the interesting scalars across sessions.

Channels worth trending:

| Channel | Why |
|---------|-----|
| `$21` TID `$82` | Catalyst OSC — the cat is a consumable, especially with track heat |
| `$02` TID `$05` / `$06` | O2 sensor response times — sensors age and go lazy |
| `$41` / `$42` TID `$80` | Heater resistance — catches wiring/extension problems |
| `$A2`–`$A5` TID `$0C` | Per-cylinder misfire, per session |
| `$35` TID `$84` / `$85` | VVT response times — oil condition / solenoid wear proxy |

This is cheap (a handful of requests, twice per session, at zero cost to your
high-rate polling loop) and gives you a **predictive** maintenance signal rather
than a reactive one.

---

## Caveats

- **Monitors must have run.** A monitor that has not executed this drive cycle
  reports `Incomplete` (see EVAP above). Mode $09 In-Performance Tracking gives
  you the completion counters to know whether a value is fresh.
- **TID semantics for "Manufacturer Defined" entries are inferred**, not from a
  Mazda spec. The `$21 / $82` = OSC identification is well-supported by its
  behavior (it failed in lockstep with a P0421 and a physically poisoned cat) but
  is still an inference.
- Values here are from **one car, one capture**. Trim/model-year variation across
  ND3 builds is unmapped. Additional captures welcome — open a PR.
