# SlimeVR_Dock — Design Review v5 (first PCB re-import + layout pass)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-08-28
**Reviewing:** the **current uncommitted working tree** (`git status` shows `Board/SlimeVR_Dock.kicad_pcb`, `Board/SlimeVR_Dock.kicad_pro`, `Board/SlimeVR_Dock.kicad_sch`, `Board/DUSBSS.kicad_sch`, `Board/StepDown.kicad_sch` all modified since `HEAD` = `239885e`, the commit that captured the N4/N5 fix from `review_v4.md`).

---

## Read this part first

This is the first pass since `review.md` (v1) where the **`.kicad_pcb` file itself has actually changed** — it was frozen at 2026-05-20 through v2, v3, and v4; it's now dated **today**, with a ~36,000-line diff. That means "Update PCB from Schematic" was finally run and real layout work happened, which reopens the entire PCB-side half of the review that's been sitting untouched for three and a half months.

**The good news first:** schematic connectivity is byte-for-byte unchanged from `review_v4.md` — I diffed the two netlists node-for-node and there is zero difference. N1, N2, N4, and N5 are all still exactly as fixed. On top of that, three things happened that line up directly with recommendations from this review series:

- **`C24`/`C25` were swapped from 100 µF electrolytic to 47 µF ceramic (0805)** — exactly the "replace like-for-like with more of what's already proven on the board" option discussed, not just a single smaller cap dropped in.
- **`C42` moved from an 0805 to an 0402 footprint** — matching TI's layout guideline text precisely (*"Place one more small capacitor (2.2 nF, 0402 size)..."*), which up to now had only been satisfied in value, not package size.
- **Every previously footprint-less part (`D2`, `R51`–`R53`, `R58`–`R60`, `C40`, `C41`) now has a real footprint assigned** — this is what allowed the PCB re-import to actually place them; schematic-parity's "missing footprint" count is now **zero**, down from 7.

**What needs attention from this pass:** the layout work introduced a handful of new, PCB-side issues that weren't there before — a stray unrouted buzzer pad, a duplicated `R36` footprint, and 21 DRC violations from thermal vias sized below the board's configured minimum drill. None of these are subtle netlist-tracing catches like N1–N5 were; they're straightforward DRC output, listed below.

**Everything in Part 3 (H1, H3, H5, H7, M1, M2, M3, M4, M5, M6, M7)** is still schematic-unchanged, confirmed by the same zero-diff netlist comparison — this pass was entirely about the PCB and a couple of targeted capacitor swaps, not a sweep through the rest of the open items.

---

## How this pass was done

- `kicad-cli sch export netlist` on the current schematic, diffed net-for-net (and component-value-for-value) against the netlist captured in `review_v4.md` — not re-derived from scratch, since a zero-diff result is itself the strongest possible confirmation nothing regressed.
- `kicad-cli sch erc --severity-all` → **35 violations**, identical count/category breakdown to every prior pass — still just the noise floor.
- `kicad-cli pcb drc --severity-all --schematic-parity` → **27 DRC violations + 1 unconnected item + 18 schematic-parity issues** — a large improvement from v4's 32 + 0 + 58/60, reflecting the actual re-import and re-route.
- Structural checks against the raw `.kicad_pcb` (zone/mounting-hole/footprint grep) for the PCB-side findings that DRC doesn't directly report (Kelvin sensing, mounting holes) — flagged clearly below where this is a structural confirmation vs. something that would need a visual/GUI check to fully settle.

---

## Part 1 — Confirmed unchanged and correct (N1, N2, N4, N5)

Netlist diff against `review_v4.md`'s captured state: **zero connectivity changes anywhere in the schematic.** N1 (bootstrap/snubber), N2 (`OE`/`GET` decoupling), N4 (`MODE`/`TRIP`/`PGOOD`), and N5 (`C42` on `PGND`) all remain exactly as verified fixed. Not re-derived in detail here since there is nothing to re-derive — nothing moved.

## Part 1b — New, positive changes this pass

| Change | Detail |
|---|---|
| `C24`, `C25`: **100 µF electrolytic → 47 µF ceramic (0805)** | Matches `C26`/`C31`, which were already 47 µF/0805. Output bank on `5VStepDown` is now fully ceramic: `C2`+`C4`+`C6`+`C8`+`C10` (5×10 µF) + `C24`+`C25`+`C26`+`C31` (4×47 µF) ≈ **238 µF total**, no polarity risk, no electrolytic wear-out exposure. |
| `C42`: **0805 → 0402 footprint** (value unchanged, 2.2 nF) | Now matches TI's layout guideline exactly on package size, not just value and net. |
| `D2`, `R51`, `R52`, `R53`, `R58`, `R59`, `R60`, `C40`, `C41`: **all now have assigned footprints** | Previously these had no `(footprint ...)` at all in the schematic (only an empty `Footprint` field) — that's specifically what caused v2/v3/v4's "7 missing footprints" parity finding. All 7 are now real, placeable parts. |
| **First PCB re-import since 2026-05-20** | Schematic-parity dropped from 58 (v4) to **18** issues; DRC's non-parity violation count dropped from 32 to 27 despite new via-related rule checks being added by the layout work. |
| **Thermal vias now present under `U4` and `U1`** | Confirmed via the `drill_out_of_range` violations below — this is `H4` from the original review (`.kicad_pcb` thermal vias for the buck IC) actually being addressed, for the first time. See Part 2 for the caveat. |

---

## Part 2 — New issues from this layout pass

### Unrouted buzzer pad (new, error severity)

`kicad-cli pcb drc` reports **1 unconnected item**: `BZ1` pad 2 (`Net-(BZ1--)`) has a routed copper segment ending **1.096 mm short** of the pad. This is a straightforward incomplete route, not a design decision — finish that trace.

### Duplicate `R36` footprint on the PCB (new)

Schematic-parity reports `duplicate_footprints` for `R36` — there are **two physical `R36` footprints** on the board. This is a known artifact of re-importing after a symbol's footprint assignment changes (KiCad can leave the old footprint behind instead of cleanly replacing it). Given `R36` is the resistor at the center of the now-fixed N4 finding, it's worth being extra careful here: **confirm which of the two footprints is actually connected into the `MODE`/`PGOOD` copper**, delete the stray one, and re-run DRC. Four more `duplicate_footprints` entries exist for footprints with no schematic reference (`<bez reference schematu>` — likely fiducials/test points placed directly on the PCB); worth a quick look but lower priority since they're not tied to a specific schematic component.

### Thermal via drill size below the board's configured minimum (new, 21 violations)

All 21 `drill_out_of_range` violations trace to two places:

- **13 vias at `U4`'s `[Earth]`/thermal pad** (0.2032 mm actual drill)
- **12 vias at `U1` (ESP32-C3) pin 19 `[Earth]`** (0.2000 mm actual drill)

Both are thermal/ground via-stitching arrays — exactly the kind of thing `H4` asked for under the TPS548A20. The problem is purely a **rule-vs-part mismatch**: the board's Design Rules currently specify a 0.3 mm minimum hole, but these vias were placed at 0.2/0.2032 mm (both very standard small-thermal-via sizes chosen specifically to avoid solder-paste wicking through during reflow). This isn't necessarily wrong — plenty of fabs support 0.2 mm (8 mil) drills — but it needs a deliberate decision, not a silent DRC failure:

- If your fab supports 0.2 mm drills: update **Board Setup → Design Rules → Constraints → Minimum hole size** to match, so this stops flagging.
- If your fab's minimum is actually 0.3 mm: the thermal vias need to grow to ≥0.3 mm, which changes the achievable via density under the pad (more spacing needed) — worth checking your fab's capability table before deciding which way to go.

### Silkscreen issues (new, minor/cosmetic)

- `C27`'s reference designator has **0 mm text height** (`text_height` violation) — likely a degenerate/invisible text object from the layout edit, worth just re-setting its size.
- `C31`/`C13` reference fields overlap each other (`silk_overlap`).
- An `R2` silkscreen segment overlaps a silkscreen polygon on `B.Silkscreen`.
- `Q1`'s silkscreen is clipped by solder mask over `R4`'s pad (`silk_over_copper`) — cosmetic now, but silkscreen ink bleeding onto an exposed pad is worth avoiding for solderability.
- `R7`'s reference field sits over two of `U6`'s pads (`SDA`, `Alert00`) on `B.Cu` — same category, same fix (move the ref designator text).

None of these are electrical defects — they're the kind of thing that shows up after moving parts around during a layout pass and is quick to clean up.

### `J1Temp1` footprint/schematic pin-count mismatch (new)

Schematic-parity flags a `net_conflict` on `J1Temp1` pad 5 — the PCB footprint has a 5th pad that the schematic symbol (which only defines pins 1–4: `+3V3`, `SCL`, `Earth`, `SDA`) doesn't know about. Worth checking whether the physical connector footprint chosen has an extra unused pin (e.g., a 5-pin header where only 4 are wired) — if so this is harmless but worth annotating; if the footprint is simply wrong for the part, that needs fixing before fab.

---

## Part 3 — Still open (schematic side, confirmed unchanged this pass)

Confirmed via the same zero-diff netlist comparison plus direct value checks — nothing here moved:

- **H1:** `U5`/`U8`/`U11`/`U14` still `AP22652W6-7` (active-low).
- **H5:** `R6`/`R9`/`R12`/`R15` still 10 kΩ (`ILIM` unchanged).
- **M1:** `R5`/`R8`/`R11`/`R14` still 10 kΩ (3 A Rp code). Confirmed directly: `Q5`'s base (pin 1) sits straight on the port's `CC_Line` net with no series resistor — still no base resistor on the BC857 detect transistors.
- **M2:** `U17` pin 7 (`~{Q7}`) still feeds `U18`'s `DS`; pin 9 (`Q7`, true) still its own unconnected net.
- **M3:** `Enable` net still just `R17` (10 kΩ) to `U1` `EN` — no capacitor.
- **M4:** `U1` `IO2` and `IO8` both still confirmed on their own `unconnected-(...)` nets.
- **M5:** no new decoupling caps appeared near `U17`/`U18`/`U19`/`U1`'s 3V3 pin/`U3`'s input — the only cap changes this pass were `C24`/`C25`/`C42`, all already-existing positions.
- **M6:** `R1` unchanged (1 kΩ, 0603 footprint).
- **M7:** spot-checked `J2` again — `D+`/`D−` (`A6`/`A7`/`B6`/`B7`) still each on their own unconnected net.
- **H3:** `U2` (CH224K) `CFG1`/`CFG3` still `Earth`, `CFG2` still tied to `VDD` — contract configuration unchanged.
- **H7:** `L1` still 2.2 µH. This is now derivable in principle (N4 gives a working `TRIP` threshold), but hasn't been re-derived this pass — that's schematic analysis work, not something this layout-focused round touched.

---

## PCB-side findings — first real re-assessment since v1

These were frozen since the `.kicad_pcb` hadn't changed; now that it has, here's what's actually checkable:

- **H4 (thermal vias):** ✅ **now present** under `U4` and `U1` — see Part 2's drill-size caveat above; the vias exist, they just need a rule/size reconciliation before fab.
- **H6 (non-Kelvin shunts):** confirmed still **not Kelvin-sensed**. Traced the INA226 current-sense network: `R31`/`R32` (and presumably their per-port siblings) are 50 mΩ shunts in a standard **2-pad `Resistor_SMD:R_0603_1608Metric`** footprint, with the INA226's `Vin+`/`Vin−` landing on the same two pads the load current flows through. True 4-terminal Kelvin sensing isn't physically possible with a 2-pad part — this would need either a purpose-built 4-terminal shunt component or Kelvin-style PCB routing (separate sense traces branching exactly at the pad edge). Unchanged from the original finding.
- **M9 (mounting holes):** confirmed still **absent** — zero `MountingHole` footprints found anywhere in the current `.kicad_pcb`.
- **M11 (fragmented ground pour):** **not conclusively re-checked this pass.** The board now has 22 zone objects across both copper layers — that's not unreasonable on its own (multiple named nets/voltage domains legitimately get separate pours), but confirming whether the ground pour specifically is still fragmented vs. now properly stitched needs either a visual check in the PCB editor or a full copper-connectivity trace, which wasn't done here. Don't treat this as either confirmed-fixed or confirmed-still-broken — it's genuinely unknown from this pass.
- **M12/M13 (capacitor voltage ratings):** not independently re-verifiable from the schematic/netlist — voltage rating isn't a field either file stores here (confirmed when checking `C24`/`C25` earlier in this conversation). This needs a BOM-level check against whatever parts actually get ordered, not a schematic-side check.
- **L1–L15:** not re-visited this pass; nothing in the diff touched layout details these would depend on beyond what's captured above.

---

## Suggested order of work

1. **Finish the `BZ1` route** — 1 mm of missing copper, trivial fix, currently a hard DRC error.
2. **Resolve the duplicate `R36` footprint** — confirm which one is live, delete the other, re-run DRC.
3. **Decide the thermal-via drill-size question** (0.2 mm vias vs. 0.3 mm board rule) against your actual fab's capability, then either resize the vias or relax the rule — 21 violations resolve at once either way.
4. Clean up the 6 silkscreen violations (quick, cosmetic, but easy to batch while already in the editor).
5. Check the `J1Temp1` footprint's 5th pad — confirm intentional-and-unused vs. wrong footprint.
6. Once the above are clear, re-run `kicad-cli pcb drc --severity-all --schematic-parity` to confirm a clean sheet before ordering.
7. The remaining Part 3 items (H5 → M1 → M2 → M3/M4 → M5, plus H7's now-derivable overcurrent math) are unaffected by any of this and can be picked up independently whenever you're back in the schematic rather than the layout.

---

## Sources

- `review_v4.md` (2026-08-28) — baseline for N1/N2/N4/N5, confirmed via zero-diff netlist comparison this pass rather than re-derived.
- No new datasheet fetches this pass — `C42`'s 0402 footprint change was checked directly against the TPS548A20 layout-guideline text already extracted and quoted earlier in this review series.
