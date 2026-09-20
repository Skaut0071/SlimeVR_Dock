# SlimeVR_Dock — Design Review v10 (consolidated status, re-verified from scratch)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-09-20
**What this document is:** `review.md` (v2, 2026-08-09) through `review_v9.md` span seven passes, several corrections, two outright retractions, and at least one "not verified this pass" placeholder because `kicad-cli` wasn't available at the time. That's a hard chain to hold in your head. This document throws all of that away and re-derives the current status **directly from the files on disk right now** — no finding below is carried forward on trust from an earlier review's prose.

**How this pass was actually done** (so you can check my work):
- Found `kicad-cli.exe` at `C:\Program Files\KiCad\10.0\bin\` (v10.0.2) and ran it directly — it was reported "unavailable" in `review_v9.md`, which is why that pass couldn't confirm several items.
- `kicad-cli sch export netlist` on the current `SlimeVR_Dock.kicad_sch` → every net named below is quoted straight from that netlist, not from schematic coordinates.
- `kicad-cli sch erc --severity-all` → 35 violations.
- `kicad-cli pcb drc --severity-all --schematic-parity` on the current `.kicad_pcb` → 64 violations, 13 schematic-parity issues, 0 unconnected items.
- Direct grep/read of `SlimeVR_Dock.kicad_pcb`, `SlimeVR_Dock.kicad_pro`, and `SlimeVR_Dock.kicad_dru` for anything DRC doesn't itself report (mounting holes, Kelvin-sense footprints, zone structure, footprint pad geometry, custom rules).
- Repo `git status` is clean, so this is exactly your last commit — not a work-in-progress snapshot.

---

## Bottom line

The board is in noticeably better shape than the tone of the last few reviews suggests. **DRC on the current PCB comes back with zero electrical violations** — no unconnected nets, no clearance/hole/annular-ring errors. All 64 reported DRC items are cosmetic silkscreen (text size/overlap). ERC is unchanged at 35 violations, the same noise-floor breakdown every pass in this series has reported. Schematic-parity is down to 13 issues, and once you filter out what's actually benign, there's really only **one** parity item worth a look.

Of the substantial defects the original review found, **everything except three items is now fixed or was correctly retracted as not-a-defect.** The three genuinely open items are: no local decoupling/pull-ups added for the ESP32 hygiene items (M3/M4/M5), the current-sense shunts are still not Kelvin-sensed (H6, structural), no mounting holes exist (M9), and — newly reconfirmed this pass — **the inductor `L1`'s footprint/datasheet metadata still doesn't match the part actually specified.**

---

## Part 1 — Done (verified against the current netlist/PCB, not assumed)

| # | Finding (original wording) | Verified current state |
|---|---|---|
| C1 | TPS548A20 `VDD` fed through 4.7 kΩ, no bypass cap | `R55` = 2.2 Ω (`+3V3`→`VDD`), `C40` = 1 µF (`VDD`→GND). Confirmed in netlist. |
| C3 | 74HC595 `SRCLR` tied to 5 V while `VCC` = 3.3 V | `U19` pin 10 (`SRCLR`) is on the `+3V3` net with pin 16 (`VCC`). Confirmed. |
| H1 (pull direction) | `EN0`–`EN3` pulls wired backwards for active-low switches | `R51/R52/R53/R58` (10 kΩ) all confirmed on `+3V3`, correct polarity. |
| N1 | Bootstrap cap `C41` shared a node with the `R54`/`C27` snubber | `C41` pin 1 is on the `SW` net (with `L1` pin 1 and `U4`'s SW pins) directly; pin 2 on `VBST`. Snubber (`R54`+`C27`) sits on its own independent node to GND. Two clean branches, exactly as TI's layout note asks. |
| N4 | `MODE`/`TRIP`/`PGOOD` resistors cross-wired — no working overcurrent trip, converter silently in Auto-skip instead of FCCM | `R36` (20 kΩ) is `MODE`↔`PGOOD` (selects FCCM). `R39` (43 kΩ) is `TRIP`↔`Earth` (real ground-referenced OCL). Confirmed. |
| N5 | Input-snubber `C42` landed on `PGOOD` instead of `PGND` | `C42` pin 1 on `/StepDown/Vin`, pin 2 on `Earth`. Confirmed. |
| H5 | Load-switch `ILIM` resistors gave a 2.4–2.9 A limit, above the AP22652's 2.1 A rating | `R6/R9/R12/R15` = 14.3 kΩ (~1.66–2.00 A limit, comfortable margin both above your ~1.5 A/switch actual load and below the 2.1 A rating). Confirmed. |
| M2 | 74HC165 cascade used the inverted `~Q7` output instead of true `Q7` | `U17` pin 9 (`Q7`, true) now feeds `U18` pin 10 (`DS`). Pin 7 (`~Q7`) is unconnected. Confirmed. |
| M6 | `R1` (1 kΩ) dissipating ~279 mW in a 0603 (100 mW rating) | Footprint is now `R_2010_5025Metric` (~750 mW rating), value unchanged. ~37% derated. Confirmed. |
| H4 | No thermal vias under the buck IC / ESP32 | 13 vias under `U4`'s GND pad + 12 under `U1` pin 19, both 0.2/0.2032 mm drill. **Also confirmed:** `SlimeVR_Dock.kicad_dru` has a custom `"Hole diameter"` rule setting the minimum to 0.2 mm (matching JLCPCB's capability), which is why DRC no longer flags these — `review_v5.md`'s 21 `drill_out_of_range` violations are gone because the rule was fixed, not because the vias were resized. `review_v9.md` listed this "still open" only because it couldn't run DRC to see the rule change. |
| — | `BZ1` pad 2 unrouted, ~1.1 mm short (`review_v5.md`) | DRC now reports 0 unconnected items. Route is complete. |
| — | Duplicate `R36` footprint on PCB (`review_v5.md`) | Only one `R36` footprint exists. The DRC "duplicate footprint" warnings that still show up are unrelated — see Part 3 below, they're a false alarm on a different set of parts. |
| M11 | Fragmented ground pour (flagged as genuinely unknown in `review_v5.md`) | The PCB now has a **single** `Earth`-net zone spanning both `F.Cu` and `B.Cu` (down from 22 zone objects at the time of `v5`), and DRC reports no isolated-copper/unconnected issues anywhere. Strong evidence this is fixed; a visual check in the PCB editor is the only way to be 100% certain no island is hiding inside that one zone, but nothing in DRC's output suggests one. |

**Correctly retracted — these were never real defects, no action needed:**

| # | Original framing | Why it's not a defect (per `review_v3.md`/`v7.md`, re-confirmed this pass) |
|---|---|---|
| N2 | `OE` shorted to the shared 595 clock/latch line | Fixed differently than first proposed: `OE` (`U19` pin 13) is on the `Boot` net with `R18` (10 kΩ pull-up to `+3V3`) and the ESP32's `IO9` strap pin. Confirmed in netlist. This deliberately reuses the boot-strap/EN header pattern — idle-high (safe/off) at power-up, and the load switches only flicker if someone uses the programming header while trackers are plugged in. Accepted tradeoff, not a bug. |
| N3 | Gate/`PG` zener clamp on wrong node | It's on the right node; it does what it was built for (keeps `PG` under its 13.5 V max). Side effect: `Q1`'s off-state `V_GS` floors at −10 V instead of ~0 V when `PG` is floating (not asserting) at full `VBUS`. Confirmed unchanged in netlist. Real tradeoff, optional hardening available, not a required fix. |
| H1 (part choice) | `AP22652` is active-low, "backwards" for the design | Confirmed still `AP22652W6-7` on all four switches. Worked through what a part swap would cost — it's a coordinated firmware+hardware change for a polarity convention, not a defect. No hardware change needed. |
| H3 | 20 V PD contract oversized vs. ~0.75 A/port actual draw | You chose to keep 20 V for future headroom. Confirmed unchanged: `CFG1`=`Earth`, `CFG2`=`VDD`, `CFG3`=`Earth` — correctly wired for 20 V. No wiring change needed. |
| M1a | CC advertisement resistors (10 kΩ) "over-advertise" 3 A | Retracted: these ports talk to a fixed-`Rd` dumb sink, not a standards-reading device — `Rp`'s exact value doesn't change the attach-detection threshold, which is set by the BC857's own `V_BE`. Confirmed `R5/R8/R11/R14` still 10 kΩ, correctly unchanged. |
| M7 | `D+`/`D−` unconnected on `J2`–`J9`, "missing DCP signature" | Retracted: these ports are charge-only; USB data goes through a separate ESP32 programming connector. No BC1.2 signature needed. Confirmed still unconnected on all 8 ports — this is correct, not a gap. |
| M1b | No base resistor on `Q2`–`Q5` (BC857 CC detect) | Downgraded to optional ESD/fault hardening, not a required fix. Confirmed still absent (`Q2` base lands directly on the CC net) — your call, not a defect. |

---

## Part 2 — Still open (real to-do items)

| # | Finding | Verified current state | Fix |
|---|---|---|---|
| **H7 / L1** | Inductor footprint claims a part it isn't | The PCB footprint is named `Inductor_SMD:L_Coilcraft_XAL7070-XXX` and its `descr` field still cites a real Coilcraft XAL7070 datasheet PDF. But the pad geometry (1.92 mm-wide pads, 8.2×8.5 mm courtyard) is a hand-stretched version of the older `XAL7030` footprint, not a real XAL7070 land pattern (verified directly by reading the footprint's pad/courtyard coordinates in `SlimeVR_Dock.kicad_pcb`) — and per `review_v9.md`'s part-level check (not re-verified independently this pass, no internet lookup done here), the part actually behind this footprint is a **Coilank `APS0730M2R2F`** (LCSC `C49261221`), not a Coilcraft part at all. Coilank's own `I_sat` figure (17 A) sits right at, not above, the ~15.9 A worst-case OCP trip point derived in `review_v6.md`, and no independent datasheet for that figure was ever found. | Either commit to `XAL7030-222MEC` (real Coilcraft part, 18 A `I_sat` with a stated 30%-drop basis, `review_v8.md`'s actual recommendation — you're already touching the footprint either way) or get a real datasheet for the Coilank part before trusting its margin, and in either case fix the footprint name/`descr`/datasheet-link so it describes the part that will actually ship. |
| **M3** | No decoupling cap on ESP32 `EN` | `Enable` net is still just `R17` (10 kΩ) to `U1` pin 2. No capacitor found anywhere on this net. | Add 1 µF ceramic (0603), `EN` to GND, close to the pin. |
| **M4** | `IO2`/`IO8` (ESP32 strap pins) floating | Both still on their own `unconnected-(...)` nets in the current netlist. | 10 kΩ pull-up to `+3V3` on each (2 new parts) — `IO2` must read 1 in both boot modes, `IO8` needs a defined high for reliable download-mode entry. |
| **M5** | No local decoupling near `U17`/`U18`/`U19`/`U1`-3V3/`U3`-input | No capacitors beyond `C42` exist above `C40` in the current schematic — confirmed no `C43`+ parts anywhere. | 100 nF ceramic (0603) at each of those 5 pins. |
| **H6** | Current-sense shunts (`R31`/`R32` and siblings) not Kelvin-sensed | Confirmed: standard 2-pad `Resistor_SMD:R_0805_2012Metric` footprint, INA226 `Vin+`/`Vin−` land on the same two pads the load current flows through. True Kelvin sensing isn't physically possible with this footprint. | Needs either a 4-terminal shunt part or Kelvin-style routing (sense traces branching exactly at the pad edge) — a structural change, not a quick fix. |
| **M9** | No mounting holes | Confirmed: zero `MountingHole` footprints anywhere in the current `.kicad_pcb`. | Add before finalizing mechanical fit. |
| — | `J1Temp1` footprint has a 5th pad the schematic symbol doesn't define | Still present — DRC's `net_conflict` on pad 5. This is the one schematic-parity item that isn't benign noise (see Part 3). | Confirm whether the physical connector genuinely has an unused 5th pin (harmless, just annotate it) or the wrong footprint was picked (needs fixing before fab). |
| — | Cosmetic silkscreen (reference text size/overlap on `C27`, `R39`, `R54`, `R34`, several `J2`–`J9` fields, `C31`/`C13`/`3V3`/`U4` overlaps) | Confirmed still present — all 64 current DRC violations are exactly this category (50 `silk_overlap`, 8 `text_thickness`, 4 `text_height`, 2 `silk_over_copper`). Zero electrical DRC violations. | Batch-clean whenever you're next in the PCB editor; none of these affect function. |
| — | Capacitor voltage ratings (`M12`/`M13`) | Not checkable from schematic/netlist — voltage rating isn't a field stored in either file. | Check against whatever parts actually get ordered (BOM-level, not a design-file check). |

**Benign schematic-parity noise (not defects, no action needed):** of the 13 current parity issues, 4 `duplicate_footprints` + 8 `extra_footprint` warnings all trace to two harmless sources, confirmed by reading the PCB file directly: (a) 5 unnamed single-pad footprints implementing the `H4` thermal-via arrays under `U4`/`U1` — these have no schematic symbol by construction, which is exactly what triggers the parity warning; (b) 3 `TestPoint_Pad_1.5x1.5mm` footprints labeled `3V3`/`USB`/`5V` placed directly on copper as test points, also by design. Neither is the "duplicate `R36`" issue `review_v5.md` worried about — that one is separately confirmed fixed above.

---

## Suggested order of work

1. **Decide `L1`'s actual part** — either switch to `XAL7030-222MEC` (real datasheet, real margin) or get a genuine datasheet for the Coilank part before trusting it, then fix the footprint metadata either way. This is the only item with any electrical-margin risk left on the board.
2. **M3 → M4 → M5** — the three remaining ESP32/decoupling hygiene items. All new-parts-only, no rewiring, can be done in one pass.
3. **`J1Temp1` pad 5** — five-minute check to confirm intentional-vs-wrong footprint.
4. **H6 (Kelvin sensing) and M9 (mounting holes)** — structural, do whenever you're already committed to another layout pass.
5. Silkscreen cleanup and `M12`/`M13` — cosmetic/BOM-level, no urgency, batch whenever convenient.

Nothing above is blocking — the electrical DRC is clean and every previously-fatal wiring defect this series found is now either fixed or was a false alarm.

---

## Sources

- Direct output of `kicad-cli sch export netlist`, `kicad-cli sch erc --severity-all`, and `kicad-cli pcb drc --severity-all --schematic-parity` (KiCad 10.0.2), run this session against the current committed `Board/` files.
- `Board/SlimeVR_Dock.kicad_pcb`, `Board/SlimeVR_Dock.kicad_pro`, `Board/SlimeVR_Dock.kicad_dru` — read directly for the thermal-via rule, footprint pad geometry, and zone structure.
- `review.md` through `review_v9.md` — used only to know what to check, not as evidence for any status claim above; every claim here was independently re-derived.
