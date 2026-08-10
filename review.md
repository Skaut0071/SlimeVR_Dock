# SlimeVR_Dock — Design Review v2 (Follow-up on your in-progress fixes)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-08-09
**Reviewing:** the **current uncommitted working tree** (`git status` shows `DUSBSS.kicad_sch`, `SlimeVR_Dock.kicad_sch`, `StepDown.kicad_sch`, `USB-C.kicad_sch`, `SlimeVR_Dock.kicad_pro` all modified since your last commit `1a47599 Updated documentation`) — i.e. the fixes you've made since the 2026-07-28 review, not yet committed.
**New input for this pass:** you measured **~0.7–0.8 A actual draw per connected device**, not the 1 A/port design target the first review assumed. That number changes a few conclusions below — flagged where relevant.

---

## Read this part first

You clearly went through the 2026-07-28 review and fixed things. **Three of the four Criticals are genuinely fixed**, and the CH224K `PG`-overvoltage half of H2 is also correctly solved (see N3 below — an earlier draft of this review mischaracterized that one as broken; it isn't). But in the process of fixing the rest, you introduced **new wiring defects that are just as fatal as the ones they replaced** — in one case by connecting the fix to the wrong node, in another by two resistors getting their second legs swapped. These are the kind of mistake that's easy to make in a schematic editor (drag a wire to the nearest pin instead of the intended one) and easy to miss on a visual review, because *the right-looking parts are all there*. Only netlist tracing catches it, which is exactly what this pass did.

**Also important:** the `.kicad_pcb` file has not been touched since **2026-05-20**, while the schematics were edited **2026-08-09**. `kicad-cli pcb drc --schematic-parity` now reports **63 parity issues**, including **7 footprints missing entirely** (the zener D2, resistor R60, and capacitors C40/C41/C42 you just added have no home on the board yet). The PCB-side findings from the first review (thermal vias, Kelvin sensing, ground fragmentation, mounting holes, etc.) are **all still exactly as described in the first review** — nothing there has changed, because nothing there *could* have changed. This pass re-verifies only what could change: schematic wiring and part values.

---

## How this pass was done

- `kicad-cli sch export netlist` on the current working tree, every affected net re-traced by hand from the raw netlist (not just visually from the schematic — coordinates in `.kicad_sch` don't reveal connectivity reliably without resolving symbol/pin geometry, so netlist ground-truth was used throughout).
- `kicad-cli sch erc --severity-all` → 35 violations (19 `pin_to_pin`, 8 `power_pin_not_driven`, 6 `lib_symbol_mismatch`, 1 `multiple_net_names`, 1 `footprint_link_issues`) — essentially the same noise floor as the first review, see [L2].
- `kicad-cli pcb drc --severity-all --schematic-parity` → 32 DRC violations + **63 schematic-parity violations** (see above).
- **TPS548A20 datasheet (SLUSC78A) re-fetched and text-extracted directly** (the web-fetch tool couldn't parse the PDF's tables, so it was downloaded and read locally with `pypdf`) specifically to verify the MODE/TRIP/PGOOD pin functions and Table 3 — this is what caught finding **N4** below, and I did not want to rely on memory for a part with this many resistor-programmed pins.

---

## Part 1 — What you fixed (confirmed correct)

| # | Original finding | Status | Evidence |
|---|---|---|---|
| **C1** | TPS548A20 `VDD` fed through 4.7 kΩ with no bypass cap | ✅ **Fixed correctly** | `R55` changed **4.7 kΩ → 2.2 Ω**; new `C40` = **1 µF** added from `VDD` to GND. Exactly the fix recommended (2.2 Ω + 1 µF). |
| **C3** | 74HC595 `SRCLR` tied to 5 V while `VCC` = 3.3 V | ✅ **Fixed correctly** | `U19` pin 10 (`SRCLR`) now sits on the `+3V3` net alongside `VCC` (pin 16). Register now has a defined reset. |
| **H1** (pull direction) | `EN0`–`EN3` pull resistors wired the wrong way for an active-low switch | ✅ **Fixed correctly** | `R51`, `R52`, `R53`, `R58` (10 kΩ, the EN0–EN3 pulls) now go to **`+3V3`**, not GND. Correct polarity for the AP22652's active-low `EN`: with the 595 outputs tri-stated, all four ports now default OFF instead of ON. |

Good work on these three — they're clean, minimal, and match what was asked for.

---

## Part 2 — New defects introduced by the fix attempts

This is the part that needs your attention before anything else. N1, N2, N4 and N5 below are genuine wiring defects — the right parts are on the board with the right values, but the netlist shows the connection is wrong. N3 turned out **not** to be one (see the correction in that section) — it's included because it's a real tradeoff worth understanding, not because it needs fixing before fab.

### N1 — Bootstrap capacitor and snubber now share one node (was C2)

You added `C41` (0.1 µF, 0805) — the correct value for the bootstrap cap TI's datasheet asks for between `VBST` and `SW` — and you moved the `R54`(3 Ω)/`C27`(470 pF) snubber pair, which used to be wired to `VBST`, to sit between `SW` and GND instead. Both of those are exactly what the first review recommended.

But tracing the actual netlist:

```
VBST ──C41(0.1µF)── NodeX ──R54(3Ω)── GND      (Net-(U4-VBST) + Net-(C27-Pad1))
                       │
                     C27(470pF)
                       │
                      SW                        (Net-(C27-Pad2), also L1 pin1 + U4 SW pins)
```

`C41` pin 2, `C27` pin 1, and `R54` pin 1 all land on the **same node** (`Net-(C27-Pad1)`) instead of `C41` going straight to `SW`. So instead of two independent branches —

- bootstrap: `VBST` — `C41` — `SW` (direct)
- snubber: `SW` — `C27` — `R54` — `GND` (direct)

— you have `C41`'s "SW-side" terminal reaching `SW` only through the 470 pF snubber cap, and that same node is also being pulled toward GND continuously through the 3 Ω snubber resistor. Two consequences:

1. **The bootstrap cap can't do its job.** Its bottom plate is supposed to slew with `SW` (0 V → 20 V → 0 V each switching cycle) so the internal boot PMOS can recharge `VBST−SW` every cycle. Here it only sees `SW` through a 470 pF divider and is simultaneously bled toward GND through 3 Ω. The high-side gate driver will be underdriven — the same "linear-region dissipation, converter dies at 8 A" failure mode as the original C2, just via a different mechanism.
2. **This is precisely what TI's layout guidance warns against.** From the datasheet: *"use separated vias or trace to connect SW node to the snubber, bootstrap capacitor... Do not combine these connections."* You've combined them.

**Fix:** Split the node. `C41` pin 2 needs its own trace straight to the `SW` copper (the same net as `C27` pin 2 / `L1` pin 1 / `U4` SW pins) — not to the `C27`/`R54` junction. `R54`+`C27` stay as the independent SW→GND snubber, untouched by `C41`.

---

### N2 — 74HC595 `OE` is now hard-wired to the shift-register clock/latch line (new — introduced while fixing H1)

The first review's H1 fix option 2 was: *"pull `U19`'s `OE` high through a 10 kΩ resistor with a GPIO driving it low once firmware is ready."* You added a resistor for this (`R50`), but its value is **0 Ω**, and both its ends land on:

```
Net-(U19-~{OE}):  R50 pin 1 — U19 pin 13 (OE)
GET:               R50 pin 2 — U1 pin 10 (ESP32 IO10) — U17 pin 1 (PL̄) — U18 pin 1 (PL̄) — U19 pin 12 (RCLK)
```

`R50` is a 0 Ω jumper, so `OE` is now **directly shorted to `GET`** — the same line the previous review praised as *"a neat trick sharing one clock and one latch line between the 595 and the 165s"* (`GET` idles high, pulses low to parallel-load the two 74HC165s, and the same rising edge latches the 74HC595's shift register into its outputs via `RCLK`).

`OE` is active-low (low = outputs driven, high = outputs Hi-Z). Tying it to `GET` means: whenever `GET` sits at its idle-high level (which is essentially all the time between polls), `OE` is also high, and the 595's outputs are **tri-stated** — at which point the pull-ups you just fixed in H1 take over and hold every port disabled. The 595 only drives real data onto `EN0`–`EN3` for the instant `GET` is pulsed low. Net effect: **the load switches will only ever be briefly, glitchily enabled during the register-load pulse and forced off the rest of the time** — the ports will not stay powered.

This isn't what the recommended fix described (a dedicated pull-up plus a *separate* GPIO for OE, decoupled from the shared shift-register clock). It looks like `R50` was repurposed from its old role (dead EN0 pull-down) without giving `OE` its own line.

**Fix:** Disconnect `R50`/`OE` from `GET`. Either dedicate a spare ESP32 GPIO to drive `OE` directly (firmware pulls it low once boot is complete and the desired port state is loaded), or — simpler, since you already fixed the EN pull-up direction — just tie `OE` permanently to GND (always enabled) and rely on the 595 outputs plus the H1 pull-ups for the safe default state. Either way, `OE` must not share a net with `GET`/`RCLK`/`PL̄`.

---

### N3 — Gate/PG clamp: correctly protects PG, but the shared node caps Q1's off-state V_GS at −10 V as a side effect (revised — see below)

**Correction from an earlier version of this review:** I originally described this as the zener being "wired to the wrong node" and framed it as a straightforward mistake. That was wrong, and worth setting straight rather than leaving in place. You added `D2` (10 V zener) and `R60` (1 kΩ) specifically to solve the *other* half of the original H2 finding — the CH224K's `PG` pin has a **13.5 V absolute maximum**, and in the original (pre-fix) circuit `PG` was pulled straight to `VBUS` = 20 V whenever it released. That's the problem this component pair actually targets, and it works:

```
VBUS ──R3(10k)──┬── Q1 gate
                 │
PG ──R60(1k)─────┤
                 │
             D2 cathode
                 │
             D2 anode ── GND (Earth)
```

Solving the network for both states (VBUS = 20 V, R3 = 10 kΩ, R60 = 1 kΩ, Vz = 10 V):

- **`PG` released / floating** (pre-negotiation or between assertions): `R3` pulls the shared gate/PG node up until `D2` breaks down at **10 V**, drawing `(20−10)/10 kΩ = 1 mA` through `R3` into the zener. Because `PG` is open-drain and not sinking in this state, **no current flows through `R60`**, so `PG` sees the same 10 V as the gate node — comfortably under the 13.5 V limit. **This is exactly what you built it for, and it does the job.** I should have credited that the first time.
- **`PG` asserted (sinking, ~0.2 V)**: solving the `R3`/`R60` divider gives `V_gate ≈ 2.0 V`, so `V_GS ≈ −18 V` — inside the NDT452AP's ±20 V rating, with the same thin margin the original review flagged, not made worse by this circuit.

**The side effect worth knowing about** (a real tradeoff, not a bug): because `PG` and `Q1`'s gate share that node through only 1 kΩ, the same 10 V clamp that protects `PG` also caps the gate in the *released* state at 10 V — giving `V_GS ≈ 10 − 20 = −10 V` even when `PG` has never asserted. The NDT452AP's `R_DS(on)` is *specified at* `V_GS = −10 V`, i.e. that's the datasheet's own "fully enhanced" test point, not a marginal leakage condition — so whenever `VBUS` is up at the full 20 V and `PG` happens to be floating rather than actively sinking, `Q1` will be conducting at close to its rated on-resistance, not cleanly off.

Whether that matters in practice comes down to CH224K timing I don't have solid documentation for: if `PG` reliably asserts low *synchronously* with `VBUS` reaching the negotiated voltage (which is the whole point of a "power good" signal), the exposure window is just transient/fault conditions, not steady-state operation — and given `PG` only floats at `VBUS` ≤ 5 V during normal pre-negotiation power-up (the zener doesn't even engage below ~13.3 V of `R3` pull-up, so a plain 5 V source never reaches breakdown), the "board partially defeats PD gating" framing I used before is too strong for normal operation. It's a real, narrow edge case (CH224K fault, or a `PG` glitch during the voltage transition), not a guaranteed failure mode.

**If you want to remove the tradeoff entirely** rather than accept it: the two requirements (`PG` ≤ 13.5 V, and a clean `V_GS ≈ 0` OFF state for `Q1`) can't both be satisfied by one resistor-divider-plus-single-clamp on a shared node — any current the clamp sinks to protect `PG` also pulls the gate down through `R3`, regardless of which side of `R60` you put the zener on (I checked both placements; they land within ~1 V of each other). Decoupling them properly needs `PG`'s protection to not load the gate at all — e.g. a much larger value in place of `R60` combined with `Q1`'s gate getting its own gate-source zener (cathode at gate, anode at `VBUS`/source, per the original H2 sketch) so the OFF state is set purely by `R3`, independent of whatever's protecting `PG`. That's a bigger change than swapping one connection, so I'd treat it as optional hardening rather than a must-fix — the current circuit does correctly solve the problem you built it for.

---

### N4 — TPS548A20 `MODE`, `TRIP`, and `PGOOD` resistors are cross-wired; overcurrent protection has no ground reference (new)

This one needed the actual datasheet text to pin down, so I fetched and read it directly rather than relying on the first review's summary. Two relevant passages:

> *"Connect the TRIP pin to GND through the trip-voltage setting resistor, R<sub>TRIP</sub> (20 kΩ < R<sub>TRIP</sub> < 65 kΩ). The TRIP terminal sources I<sub>TRIP</sub> current... and the trip level is set to the OCL trip voltage V<sub>TRIP</sub>."* — §7.3.7
> Table 3, **FCCM** row: *"Connect to PGOOD, R<sub>MODE</sub> = 20 kΩ"* → §7.3.6: *"When the MODE pin is tied to the PGOOD pin through a resistor, the controller operates in continuous conduction mode (CCM)..."*

So the two documented, correct connections are: **`MODE` — 20 kΩ — `PGOOD`** (this is how you select FCCM, which the first review confirmed was your intended mode) and **`TRIP` — R<sub>TRIP</sub> (here 43 kΩ, inside the valid 20–65 kΩ window) — `GND`**.

The current netlist has these two resistors' second legs swapped:

| Component | Currently wired | Should be |
|---|---|---|
| `R36` (20 kΩ) | `PGOOD` (pin 2) — `TRIP` (pin 25) | `MODE` (pin 21) — `PGOOD` (pin 2) |
| `R39` (43 kΩ) | `GND` — `MODE` (pin 21) | `GND` — `TRIP` (pin 25) |

Both pins are now broken:

- **`TRIP`** has no path to GND at all — it only reaches `PGOOD` through `R36`. `PGOOD` is an open-drain status pin pulled to `VREG` (~5 V, via the new `R56` = 100 kΩ) whenever the converter is healthy — so instead of the OCL threshold being set by a current sourced through a resistor *to ground*, it's now referenced to a pin sitting around 5 V. The datasheet's whole OCL-setting equation assumes `TRIP` is grounded through `R_TRIP`; referencing it to `PGOOD` instead produces an undefined, almost certainly far-too-high trip voltage — **cycle-by-cycle overcurrent limiting is not functioning as designed.**
- **`MODE`** is tied to GND through 43 kΩ. Table 3 tabulates specific `R_MODE` values for Auto-skip mode pull-down-to-GND (0/30/40/50/60/80/100/120/150 kΩ, each mapping to a specific RC time constant / switching-frequency pair) — 43 kΩ isn't one of them, so you've landed in an undocumented gap between the 40 kΩ and 50 kΩ bins. More importantly, **this also silently switches the converter from FCCM to Auto-skip mode** (a pull-down to GND selects Auto-skip; only "tied to PGOOD" selects FCCM), which is a different converter behavior than what you validated and intended in the first review.

Everything downstream of this (H7's saturation-vs-OCP-threshold analysis in the first review, which assumed a working 43 kΩ TRIP setting) needs to be revisited once this is fixed — right now there effectively **is no ground-referenced overcurrent trip**, which is a worse starting point than H7 described.

**Fix:** Swap the two resistors' second-leg connections back: `R36` (20 kΩ) between `MODE` and `PGOOD`; `R39` (43 kΩ) between `TRIP` and `GND`.

---

### N5 — New 2.2 nF input-snubber cap landed on the PGOOD net instead of PGND (new)

The first review's C2/M13 recommended TI's *"2.2 nF 0402 between VIN and PGND"*, placed right at the IC. You added `C42` = 2.2 nF — but its second pin sits on `Net-(U4-PGOOD)` (the same node as `R36` and `R56` from N4 above), not on the `Earth`/GND net.

Given N4 above is already scrambling that same cluster of pins, this is likely the same slip (a wire dragged to the nearest pin in a crowded area of the schematic) rather than a separate mistake, but it needs its own fix: `C42` should go from `VIN` straight to `PGND`/`Earth`, not to `PGOOD`.

**Fix:** Move `C42` pin 2 to the `Earth` net.

---

## Part 3 — Still open (unchanged since 2026-07-28)

Confirmed via netlist that these are untouched — the PCB file's May 20 timestamp and the fact that `USB-C.kicad_sch` only changed 2 lines (unrelated) both corroborate this; your edits this session were focused on the buck (`StepDown.kicad_sch`) and the 595/165 logic.

- **H1 (part choice):** `U5/U8/U11/U14` are still `AP22652W6-7` (active-low). The pull-direction fix (Part 1) makes the *default* state safe now, but the part is still active-low against a design that reads more naturally active-high; still worth the AP22653 swap if you respin.
- **H5 (load-switch loading) — updated with your measurement:** at your measured **~0.75 A/port actual**, two ports on one switch is **~1.5 A**, which is **71% of the AP22652's 2.1 A continuous rating** — meaningfully better than the 95% the first review calculated against the 1 A/port design target. This lowers the urgency of the "one switch per port" recommendation. **However, `R6/R9/R12/R15` are still 10 kΩ**, which per the datasheet's own equations still sets the current limit at **2.4–2.9 A** — above the switch's 2.1 A continuous rating regardless of your actual load, so a genuine fault (shorted port) still hits thermal shutdown before the current limit engages. Worth raising `R_ILIM` to ~15–18 kΩ (≈1.5–1.8 A limit) even though normal-load thermal margin is now fine.
- **H7 (inductor/OCP mismatch):** `L1` still 2.2 µH. As noted in N4, this finding needs to be re-derived once `TRIP` is reconnected to GND — right now the more urgent problem is that there's no working trip threshold at all.
- **M1 (CC advertisement) — updated with your measurement:** `R5/R8/R11/R14` = 10 kΩ (3 A Rp code) is now an even bigger mismatch against reality: you're advertising 3 A to any compliant sink while actually delivering ~0.75 A. CC1/CC2 are still shorted together per port-pair, and the BC857 detect transistors still have no base resistor and still clamp CC to ~2.6 V (undefined result for anything reading the Rd threshold table). Unchanged, still worth fixing for anything other than your own trackers.
- **M2 (165 cascade):** `U17` pin 7 (`Q̄H`, inverted) still feeds `U18` pin 10 (`SER`); pin 9 (`QH`, true) is still unconnected.
- **M3 (ESP32 EN cap):** `Enable` net is still just `R17` (10 kΩ) to `U1` pin 2 — no capacitor added.
- **M4 (floating strap pins):** `GPIO2` (pin 16) and `GPIO8` (pin 7) are both still completely unconnected.
- **M5 (decoupling gaps):** the three new capacitors this session (`C40`, `C41`, `C42`) all went to the buck circuit — `U17`, `U18`, `U19`, `U1`'s 3V3 pin, and `U3`'s input still have no new local decoupling.
- **M6:** `R1` is still 0603 (100 mW) dissipating ~279 mW at the 20 V contract — unchanged.
- **M7 (D+/D−):** all eight downstream ports still show `A6`/`A7`/`B6`/`B7` unconnected — no DCP signature.
- **H3 / PD contract choice — worth reconsidering given your measurement.** The first review defended the 20 V contract against a 40 W (8×1 A×5 V) budget. At your actual ~0.75 A/port, real output power is **8 × 0.75 A × 5 V = 30 W**, i.e. ~34 W in at typical efficiency. A **15 V** contract (15 V/3 A = 45 W available, `CFG1=0/CFG2=1/CFG3=1`) now comfortably covers that with margin, and would simultaneously widen N3's OFF-state margin (a lower `VBUS` means the gate/PG clamp node needs less headroom before the zener even engages) and reduce M6's `R1` dissipation (drops to ~137 mW at 15 V, inside a 0603's rating). If the real-world number holds up under more devices/longer measurement, dropping the contract is a low-cost way to ease margin on two findings at once.
- **All PCB/layout findings from the first review** (H4 thermal vias, H6 non-Kelvin shunts, M9 mounting holes, M11 fragmented ground pour, M12/M13 capacitor voltage ratings, L1–L15) are **unchanged**, because the `.kicad_pcb` file hasn't been touched. They still apply exactly as written on 2026-07-28.

---

## PCB/schematic parity — needs attention before you touch the layout again

`kicad-cli pcb drc --schematic-parity` on the current schematic against the current (May-20) PCB:

- **7 missing footprints** — `D2`, `R60`, `C40`, `C41`, `C42` (and 2 others) exist in the schematic with no matching footprint on the board yet.
- **21 net conflicts, 23 footprint/symbol mismatches, 8 extra footprints, 4 duplicates.**

None of this is a design defect — it's just that the netlist hasn't been re-imported into the PCB editor ("Update PCB from Schematic") since these edits. But it means **the PCB as it currently sits does not reflect any of Part 1 or Part 2 above**, fixed or broken. Re-import before doing anything else with the layout, and re-run DRC after — some of N1–N5's corrected wiring will change trace/via requirements around `U4`.

---

## Suggested order of work

1. **N4** — swap `R36`/`R39` second legs back (`MODE`–`PGOOD` via 20 kΩ, `TRIP`–`GND` via 43 kΩ). This restores overcurrent protection, which right now doesn't exist.
2. **N1** — split `C41` off the snubber node; run it directly `VBST`→`SW`.
3. **N2** — give `OE` its own line (dedicated GPIO or hard tie to GND), disconnect it from `GET`.
4. **N5** — move `C42` pin 2 from `PGOOD` to `Earth`.
5. **N3** — no action required; the `PG`/gate clamp is working as intended. Optionally harden it later (decouple `PG`'s protection from `Q1`'s gate) if you want a cleaner OFF state than the current −10 V floor.
6. Re-import netlist into the PCB editor, place the 5 orphaned footprints, re-run DRC + ERC to zero (consciously excluding only what you accept).
7. Then work back through the still-open items in Part 3, roughly in the same priority order as the first review's "Suggested order of work" — H5's `R_ILIM`, M1's CC network, M2's cascade pin, M3/M4's ESP32 hygiene, M5's decoupling — before returning to the PCB-side items (H4, H6, M9, M11) which are unaffected by any of this and can be worked in parallel.

---

## Sources

- [TPS548A20 datasheet (SLUSC78A), Texas Instruments](https://www.ti.com/lit/ds/symlink/tps548a20.pdf) — re-fetched and text-extracted this session specifically to verify Pin Functions (§ Pin Functions), §7.3.6 Forced Continuous-Conduction Mode, §7.3.7 Current Sense and Overcurrent Protection, and Table 3 (Mode Selection and Internal RAMP RC Time Constant), for findings N1, N4 and N5.
- [CH224 series manual, WCH](https://components101.com/sites/default/files/component_datasheet/WCH_CH224K_ENG.pdf) — re-fetched and text-extracted this session to confirm `PG`'s 13.5 V absolute maximum and open-drain behavior for N3. The manual doesn't document `PG`-vs-`VBUS` assertion timing in enough detail to fully rule out the fault-condition edge case noted in N3; flagged as an open question rather than assumed either way.
- Previous review (`review.md`, 2026-07-28) for all "Part 3 — still open" items and every PCB/layout finding, which this pass did not re-derive since the underlying `.kicad_pcb` is unchanged. Full datasheet source list for AP22652/53, INA226, XC6220, NDT452AP, CH224, ESP32-C3 carries over unchanged from that review.
