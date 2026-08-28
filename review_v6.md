# SlimeVR_Dock — Design Review v6 (Part 3 resolved to concrete fixes)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-08-28
**Purpose of this pass:** every open item from Part 3 of the review series (H1, H3, H5, H7, M1–M7) gets turned into an exact, ready-to-apply change — a resistor value, a wire move, a new part — instead of a range or a "worth considering." One item (H1) turned out to be a firmware-only concern once actually worked through, so it's noted but **not** treated as a defect, per your instruction. One item (H7) hit a genuine gap in TI's own published datasheet; that's called out explicitly rather than papered over with an invented number.

**Revision note:** H3 originally recommended dropping to a 15 V contract. You've chosen to keep 20 V — 45 W of real headroom over the ~30–34 W actual draw is a legitimate reason to want the margin, especially if more/hungrier trackers get added later. That decision changes two other items downstream (M6, H7); both are updated below to match. H3 itself now needs **zero hardware change** — the board is already wired for 20 V.

---

## Action list — apply these

| # | Component(s) | Current | New | Type of change |
|---|---|---|---|---|
| H5 | `R6`, `R9`, `R12`, `R15` | 10 kΩ | **14.3 kΩ (1%, E96)** | value swap, same 0603 footprint |
| M1a | `R5`, `R8`, `R11`, `R14` | 10 kΩ | **22 kΩ** | value swap, same footprint |
| M1b | new resistor in series with `Q2`/`Q3`/`Q4`/`Q5` base | — | **10 kΩ, ×4 new parts** | new component + reroute |
| M2 | wire from `U17` pin 7 | → `U18` pin 10 | **move to `U17` pin 9** | wiring-only, no new parts |
| M3 | new cap at `U1` `EN` (pin 2) | — | **1 µF ceramic, 0603, new part, to GND** | new component |
| M4 | new resistors at `U1` `IO2` (pin 16) and `IO8` (pin 7) | floating | **10 kΩ to `+3V3` on each, ×2 new parts** | new components |
| M5 | new caps at `U17`/`U18`/`U19` VCC, `U1` 3V3, `U3` input | — | **100 nF ceramic, 0603, ×5 new parts** | new components |
| M6 | `R1` | 1 kΩ, 0603 | **1 kΩ, upsize to 2010** | footprint swap, same value |
| M7 | `D+`/`D−` on `J2`–`J9` | unconnected | **short together per port, ×8** | new 0 Ω links or direct trace |
| H3 | `CFG3` (`U2` pin 3) | tied to `Earth` | **no change — staying at 20 V, see note below** | none |
| H1 | — | active-low `AP22652` | **no hardware change — see note below** | firmware-only |
| H7 | `L1` | 2.2 µH, unverified `I_sat` | **verify against ≥17 A `I_sat`, see derivation** | BOM check, not yet a part change |

Detail and the math behind each follows.

---

## H1 — not a hardware defect, just a firmware polarity note

Worked through what an `AP22652`→`AP22653` swap would actually require: not just a BOM value change, but also flipping `R51`/`R52`/`R53`/`R58` back to pull `EN0`–`EN3` to `GND` instead of `+3V3`, to keep the safe-default-off behavior. That's a coordinated hardware+firmware change for something that's purely "does a 1 in the shift register mean on or off." Per your instruction: this is a firmware detail, not a hardware bug.

**Note for firmware:** `EN` on `U5`/`U8`/`U11`/`U14` (`AP22652`) is **active-low** — the shift-register bit for each port must be driven **low** to turn that port's power **on**, high to turn it off. No hardware change needed or recommended.

---

## H5 — load-switch current limit, exact value

AP22652 current-limit is set by a resistor from `ILIM` to GND, per Diodes Inc.'s best-fit equations (datasheet DS41186):

```
I_LIMIT_typ [mA] = 30321 / R[kΩ]^1.055
I_LIMIT_min [mA] = 28955 / R[kΩ]^1.075
I_LIMIT_max [mA] = 31033 / R[kΩ]^1.031
```

Target: each switch (`U5`/`U8`/`U11`/`U14`) serves **two ports**, so at your measured ~0.75 A/port the actual combined load per switch is **~1.5 A**. Need the limit's worst-case-low bound comfortably above that (avoid nuisance trips) and the worst-case-high bound comfortably under the AP22652's **2.1 A continuous rating** (avoid thermal-shutdown-before-current-limit).

Checked several values; **R = 14.3 kΩ** is the best balance:

| R | typ | min | max |
|---|---|---|---|
| 13 kΩ | 2.03 A | 1.84 A | 2.21 A ❌ over rating |
| **14.3 kΩ** | **1.83 A** | **1.66 A** | **2.00 A** ✅ |
| 15 kΩ | 1.74 A | 1.58 A | 1.90 A (tighter margin over 1.5 A load) |

**Decision: `R6`/`R9`/`R12`/`R15` → 14.3 kΩ (1%).** Gives ~11% margin above the actual 2-port load at the worst-case-low bound, and ~5% margin under the switch's rating at the worst-case-high bound — a real improvement over the current 10 kΩ (which computes to max ≈2.9 A, guaranteeing thermal shutdown fires before the current limiter ever would).

---

## M1 — CC advertisement and base resistor, both resolved

**Advertisement:** `R5`/`R8`/`R11`/`R14` 10 kΩ → **22 kΩ**, the standard USB-C `Rp` code for 1.5 A — matches your measured ~0.75–1 A/port with real headroom, instead of claiming 3 A you don't deliver.

**Base resistor:** `Q2`/`Q3`/`Q4`/`Q5` (BC857, per-port CC detect) currently have their base pin wired straight to the raw `CC_Line` net. Decision: **10 kΩ in series**, inserted between `CC_Line` and each base pin (4 new resistors). This is standard for a CC-line sense transistor at this bias level — small enough not to meaningfully load the CC divider a compliant device is reading, large enough to stop the base clamping the line to ~0.6–0.7 V.

---

## M2 — 165 cascade, wiring-only

Move the wire currently on `U17` pin 7 (`~Q7`, inverted) so it originates from `U17` pin 9 (`Q7`, true) instead, still landing on `U18` pin 10. `U17` pin 7 goes unconnected. No new parts.

---

## M3 / M4 / M5 — new components, standard values

- **M3:** 1 µF ceramic (0603), `U1` pin 2 (`EN`) to `GND`, placed close to the pin.
- **M4:** confirmed against Espressif's ESP32-C3 strapping table — `GPIO2` must read **1** in both SPI-boot and download-boot modes (never left to chance), and `GPIO8` has no internal default pull and must be driven **high** for reliable download-mode entry over the existing `Boot` header. Decision: **10 kΩ pull-up to `+3V3` on both `IO2` (pin 16) and `IO8` (pin 7)**, matching the pattern already used for `EN`/`IO9` elsewhere on this board.
- **M5:** 100 nF ceramic (0603) at `U17`/`U18`/`U19`'s `VCC` pins, `U1`'s `3V3` pin, and `U3`'s input — 5 new parts, each placed as close to its pin as possible.

---

## M6 — R1, updated for staying at 20 V

With the 20 V contract kept (see H3 below), `R1`'s dissipation stays at the original ~279 mW figure (from the earlier `VBUS`→`R1`→CH224K `VDD` drop analysis) — there's no reduction from a lower contract to lean on here. That number changes the footprint decision from what I'd proposed under the 15 V option:

- **1206** (rated ~250 mW): 279 mW would be **112% of rating** — actually over-stressed, not a fix.
- **1210** (rated ~330 mW): 279/330 ≈ 85% derated — technically fits, but thin margin for a part sitting near a switching regulator.
- **2010** (rated ~750 mW): 279/750 ≈ 37% derated — comfortable margin.

**Decision: `R1` → 2010 footprint, value unchanged at 1 kΩ.** Bigger jump than the 15 V-contract version of this fix, but it's what the higher dissipation actually calls for.

---

## M7 — DCP signature, all 8 ports

Short `D+` to `D−` directly on each of `J2`–`J9` (8 new 0 Ω links / direct traces). Standard BC1.2 Dedicated-Charging-Port signature; these connectors aren't carrying USB data, so there's no downside.

---

## H3 — PD contract, kept at 20 V

Pulled the CH224K's full config table (WCH datasheet) for reference:

| Target | CFG1 | CFG2 | CFG3 |
|---|---|---|---|
| 5 V | 1 | — | — |
| 9 V | 0 | 0 | 0 |
| 12 V | 0 | 0 | 1 |
| 15 V | 0 | 1 | 1 |
| **20 V (current)** | **0** | **1** | **0** |

Checked the current netlist against this: `CFG1`=`Earth`(0), `CFG2`=`VDD`(1), `CFG3`=`Earth`(0) — the board is already correctly wired for the 20 V row.

**Decision, updated: stay at 20 V.** Your call — 45 W of real headroom over the ~30–34 W actual draw is a legitimate reason to want the margin for future expansion. **No wiring change needed.** Two things this decision affects downstream: `R1`'s dissipation stays at the higher ~279 mW figure rather than dropping (see M6, updated above), and the `N3` gate/PG-clamp margin from earlier in this review series stays at its original (narrower but already-acceptable) headroom rather than widening — neither is a new problem, just the trade-off of keeping the higher contract voltage.

---

## H7 — inductor vs. OCP, worked as far as the datasheet allows

**What's solid:** `N4` gave `TRIP` a real ground reference (`R_TRIP` = 43 kΩ). TI's Equation 3: `V_TRIP = R_TRIP × I_TRIP = 43 kΩ × 10 µA = 430 mV` (typical, 25°C). Equation 4:

```
I_OCP = V_TRIP/(8×R_DS(on)) + [1/(2×L×f_SW)] × (V_IN−V_OUT)×V_OUT/V_IN
```

DC term (dominant): using typical `R_DS(on)` = 4.3 mΩ → **12.5 A**; using worst-case `R_DS(on)` = 5.7 mΩ (TI's own guidance for minimum-OCP design) → **9.43 A**.

**Where I hit a real gap, not a guess:** the ripple term needs `f_SW`, which is set by a resistor divider on the `RF` pin (`R40`=240 kΩ, `R41`=75 kΩ on this board) — TI's datasheet text says *"Refer to \[Table X\] for the relationship between the switching frequency and resistor-divider configuration,"* but that table reference is broken in the published PDF — no table follows it. I checked this isn't my extraction failing: **TI's own official HTML datasheet viewer has the identical broken cross-reference with no table**, confirming this is a genuine gap in TI's published documentation for this part, not something I can look up correctly.

Rather than invent a resistor-to-frequency mapping, I bounded the ripple term across the device's entire published range (250 kHz–1 MHz) at `V_IN`=**20 V** (updated — you're keeping the 20 V contract, see H3):

| `f_SW` | ripple term |
|---|---|
| 250 kHz (slowest) | +3.41 A |
| 1 MHz (fastest) | +0.85 A |

**Combined `I_OCP` range: ~10.3 A to ~15.9 A**, regardless of which of the 8 frequencies `R40`/`R41` actually select. (Barely different from the 15 V case — `V_IN` only enters the ripple term, which is the smaller of the two contributions, so keeping 20 V nudges the upper bound up by about 0.4 A, not a material change.)

**What this means for `L1`:** total buck output current (all 8 ports through one inductor) at your measured load is ~6–8 A — comfortably under even the tightest bound of that range (~26%+ margin), so the OCP threshold itself looks fine relative to actual operation. **The remaining open question is whether `L1` saturates before that ~10.3–15.9 A trip point does its job in an actual fault** — if `I_sat` is below ~15.9 A, a hard short could saturate the inductor (uncontrolled current spike) before the electronic limiter ever engages. The schematic only specifies the generic `Vishay IHLP-2525` family and 2.2 µH — the exact part number (which sets `I_sat`) isn't recorded anywhere I can check from schematic/netlist alone.

**Decision:** not a component change yet — **verify whatever specific IHLP-2525CZ P/N is in the actual BOM has `I_sat` ≥ ~17 A** (comfortable margin above the calculated 15.9 A worst-case trip point). If it's lower, that's the one item in this whole pass that would need an actual part swap.

---

## Sources

- TPS548A20 datasheet (SLUSC78A) — Equations 3 & 4 and R_DS(on) typ/max values read directly from the rendered datasheet page (§7.3.7), not from OCR text (the equations are embedded as graphics, not extractable text). The RF-pin frequency table gap was independently confirmed against TI's own `ti.com` HTML datasheet viewer, not just the PDF.
- AP22652/AP22653 datasheet (Diodes Inc., DS41186) — best-fit `I_LIMIT` equations, via web search of the published datasheet content.
- CH224K datasheet (WCH) — `CFG1`/`CFG2`/`CFG3` voltage-selection table, matches the `CFG1=0/CFG2=1/CFG3=1` figure already cited in earlier passes of this review, now with the full table for context.
- ESP32-C3 datasheet / hardware design guidelines (Espressif) — strapping-pin requirements for `GPIO2`/`GPIO8`/`GPIO9`.
