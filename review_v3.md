# SlimeVR_Dock — Design Review v3 (Follow-up on the v2 fixes)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-08-28
**Reviewing:** the committed working tree at `HEAD` (`07e5e92 Fixing from Review`) — `git status` is clean, so this is exactly what's checked in, not an in-progress edit. This commit is the one that followed the 2026-08-09 v2 review (`review.md`) and both applied further schematic fixes *and* checked `review.md` itself into the repo as documentation.

---

## Read this part first

Two of v2's five wiring defects are now genuinely fixed, and fixed correctly. **N1** (bootstrap cap sharing a node with the snubber) is resolved exactly as recommended — `C41` now runs straight `VBST`→`SW`, and the `R54`/`C27` snubber is its own independent branch to ground. **N2** (`OE` shorted to the shared shift-register clock line) is also resolved: `R50` (the 0 Ω jumper that caused the short) has been deleted outright, and `OE` now gets its own line — a 10 kΩ pull-up to `+3V3` (`R18`) with the ESP32's `IO9` able to pull it low, decoupled from `GET`/`RCLK`. That's the "dedicate a spare GPIO" option v2 suggested. It comes with a new wrinkle worth understanding (see N2 below), not a wiring bug.

**But N4 and N5 — the two defects flagged as most urgent in v2 — were not touched.** `R36`/`R39` around the TPS548A20's `MODE`/`TRIP`/`PGOOD` pins are wired exactly the same broken way v2 described: **there is still no working ground-referenced overcurrent trip on the buck converter**, and the converter is still silently in Auto-skip mode instead of the intended FCCM. `C42` is still landed on `PGOOD` instead of `PGND`. Netlist tracing (not visual inspection — same method v2 used) confirms both are byte-for-byte the "currently wired" (broken) state from the v2 report, not the "should be" state. v2's suggested order of work put these two first; they're still first.

Nothing else changed. Every item in v2's Part 3 ("still open") — H1, H3, H5, H7, M1–M7 — is unchanged, and the `.kicad_pcb` is still the same **May 20** file, now **19 weeks** stale against the schematics (last schematic edit **Aug 9**). PCB schematic-parity is effectively unchanged: 60 issues now vs. 63 in v2 (the drop is just `R50`'s footprint moving from "missing" to "extra" — it's an orphan on the board now, not absent).

---

## How this pass was done

- `kicad-cli sch export netlist` on `SlimeVR_Dock.kicad_sch` (v10.0.2), parsed programmatically into a net→pin map and queried per-net — not read visually from the `.kicad_sch` coordinates, same rationale v2 used: symbol/pin geometry doesn't reliably reveal connectivity without resolving it.
- `kicad-cli sch erc --severity-all` → **35 violations**, same categories and same count as v2 (19 `pin_to_pin`, 8 `power_pin_not_driven`, 6 `lib_symbol_mismatch`, 1 `multiple_net_names`, 1 `footprint_link_issues`) — still just the noise floor, nothing new in it.
- `kicad-cli pcb drc --severity-all --schematic-parity` → **32 DRC violations** (identical count to v2 — expected, the `.kicad_pcb` file hasn't changed) **+ 60 schematic-parity issues** (7 missing footprints, 9 extra, 21 net conflicts, 21 footprint/symbol mismatches, 4 duplicates).
- Component values/footprints cross-checked directly against the netlist's `(value ...)` / `(footprint ...)` fields for every part named in v2's findings, to catch value or designator swaps that pure connectivity tracing wouldn't show.

---

## Part 1 — What's fixed since v2 (confirmed correct)

| # | v2 finding | Status | Evidence |
|---|---|---|---|
| **N1** | `C41` bootstrap cap shared a node with the `R54`/`C27` snubber instead of going straight to `SW` | ✅ **Fixed correctly** | `C41` pin 1 is now on `Net-(C27-Pad2)` — the same net as `L1` pin 1 and `U4`'s `SW` pins directly, not through the snubber. `C41` pin 2 is on `Net-(U4-VBST)`. The snubber (`R54` pin 1 + `C27` pin 1) sits alone on its own node (`Net-(C27-Pad1)`), touching neither `C41` terminal. Two fully independent branches now, exactly as TI's layout note asks. |
| **N2** | `R50` (0 Ω) shorted `OE` directly to `GET`, so the 595 outputs were only briefly enabled during the register-load pulse | ✅ **Fixed, with a new wrinkle (see below)** | `R50` no longer exists in the schematic (confirmed absent from the netlist's component list; its footprint now shows up as an **orphan/"extra footprint"** on the stale PCB, which is consistent with a schematic-side deletion). `OE` (`U19` pin 13) is now on its own net, `Boot`, pulled to `+3V3` through a new 10 kΩ (`R18`), with `U1` (ESP32-C3) `IO9` on the same net able to pull it low. `GET` (`U1` `IO10` → `~{PL}`/`RCLK`) is untouched and no longer shares anything with `OE`. |

Both are clean, minimal fixes that match what v2 recommended.

### N2 — worth knowing: `OE` now shares a net with the ESP32-C3's boot-mode strap pin

`IO9` on the ESP32-C3-WROOM-02 is a **strapping pin** — its level is sampled by the boot ROM at reset to choose SPI boot (normal firmware) vs. UART download mode; it has an internal weak pull-up, so pulling it low at reset forces download mode. The new `Boot` net ties three things together: `R18` (10 kΩ to `+3V3`), `U1` `IO9`, `U19`'s `OE`, and a 3-pin external header (`J1_Control1`, whose other two pins are `GND` and `EN`/reset) — i.e. this looks like a deliberate reuse of the classic "GPIO-strap + EN" auto-program header pattern, just built on `IO9` instead of the classic `IO0`, which is correct for this chip.

This is a reasonable design choice, not a wiring defect — but two consequences follow from choosing to reuse a strap pin for `OE` rather than a genuinely spare GPIO:

1. **Normal operation is fine and arguably better than a plain spare-GPIO fix would have been**: at reset, before firmware runs, `IO9`'s internal pull-up plus the new external `R18` hold `OE` high (595 tri-stated, all ports off) — which is the same safe default the H1 fix already established, now reinforced through power-up/reset, not just after the 595's own reset.
2. **The external `Boot` header now also toggles port power.** Anyone using `J1_Control1` to force the ESP32 into download mode (grounding `IO9` while cycling `EN`) is, for that same instant, also driving `OE` low and enabling all four load switches (through whatever the 595 last latched, which is undefined mid-reset). Harmless electrically, but worth knowing if that header is ever used with trackers plugged in — flashing firmware will flicker port power.

No action required unless that flicker is undesirable, in which case decoupling `OE` from `IO9` (e.g., driving it from a plain GPIO like the originally-suggested "spare pin," with `IO9` left dedicated to programming) would remove the coupling entirely.

---

## Part 2 — Still broken (v2's most urgent findings, unaddressed)

v2 ranked these **first and second** in its suggested order of work. Netlist tracing shows both are wired exactly the way v2 described as broken — not touched since.

### N4 — TPS548A20 `MODE`/`TRIP`/`PGOOD` resistors are still cross-wired (unfixed)

Current netlist:

| Component | Currently wired | Should be (per TI SLUSC78A) |
|---|---|---|
| `R36` (20 kΩ) | `PGOOD` (`U4` pin 2, `Net-(U4-PGOOD)`) — `TRIP` (`U4` pin 25, `Net-(U4-TRIP)`) | `MODE` (`U4` pin 21) — `PGOOD` |
| `R39` (43 kΩ) | `Earth`/GND — `MODE` (`U4` pin 21, `Net-(U4-MODE)`) | `Earth`/GND — `TRIP` |

Same consequences v2 described, still true:

- **`TRIP` has no path to GND at all** — it only reaches `PGOOD` (~5 V via `R56`, 100 kΩ, when the converter is healthy) through `R36`. The datasheet's OCL-setting equation assumes `TRIP` is grounded through `R_TRIP`; as wired, there is **no functioning cycle-by-cycle overcurrent limit**.
- **`MODE` is pulled to GND through 43 kΩ**, landing in an undocumented gap in Table 3's Auto-skip bins (40 kΩ/50 kΩ) *and* silently selecting **Auto-skip mode instead of FCCM** — a different converter behavior than what was validated.

**Fix (unchanged from v2):** swap the two resistors' second-leg connections — `R36` between `MODE` and `PGOOD`; `R39` between `TRIP` and `GND`.

### N5 — Input-snubber cap `C42` still lands on `PGOOD`, not `PGND` (unfixed)

`C42` (2.2 nF) pin 2 is still on `Net-(U4-PGOOD)`, the same node `R36` and `R56` sit on — not on `Earth`. TI's recommended `2.2 nF 0402 between VIN and PGND` placement is unchanged from v2's finding.

**Fix (unchanged from v2):** move `C42` pin 2 to the `Earth` net.

Given both N4 and N5 touch the same crowded pin cluster around `U4`, and neither moved between v2 and this commit, it's worth checking whether this area of `StepDown.kicad_sch` was actually opened during the "Fixing from Review" pass at all — the fixes that *did* land (N1, N2) are both in different parts of the schematic (bootstrap/snubber, and the 595/165 logic), while this MODE/TRIP/PGOOD/PGOOD-snubber cluster looks untouched.

---

## Part 3 — Still open (unchanged since v2, confirmed by netlist this pass)

All confirmed unchanged by direct netlist/value inspection this pass, not carried forward blind:

- **H1 (part choice):** `U5/U8/U11/U14` are still `AP22652W6-7` (active-low). Still safe by default (H1's pull-direction fix from v1 holds), still worth the AP22653 swap on a respin.
- **H5 (load-switch `ILIM`):** `R6/R9/R12/R15` still 10 kΩ → current limit still 2.4–2.9 A against the AP22652's 2.1 A continuous rating. Still worth raising to ~15–18 kΩ.
- **H7 (inductor/OCP mismatch):** `L1` still 2.2 µH. Still blocked on N4 above — there's no working `TRIP` threshold to compare it against yet.
- **M1 (CC advertisement):** `R5/R8/R11/R14` still 10 kΩ (3 A Rp code) against the measured ~0.75 A/port actual draw. `BC857` (`Q5`) detect transistors still have no base resistor.
- **M2 (165 cascade):** `U17` pin 7 (`~{Q7}`, inverted) still feeds `U18` pin 10 (`DS`/`SER`); `U17` pin 9 (`Q7`, true) confirmed still on its own `unconnected-(U17-Q7-Pad9)` net.
- **M3 (ESP32 `EN` cap):** `Enable` net is still just `R17` (10 kΩ) to `U1` pin 2 (`EN`) — no capacitor.
- **M4 (floating strap pins):** `U1` `IO2` (pin 16) and `IO8` (pin 7) confirmed still each on their own `unconnected-(...)` net. (Note: `IO9`, the *other* strap pin, is no longer floating — see N2 above — but that was a deliberate, separate change.)
- **M5 (decoupling gaps):** no new decoupling caps found near `U17`, `U18`, `U19`, `U1`'s 3V3 pin, or `U3`'s input; the only new caps in this commit (`C40`/`C41`/`C42`) are all in the buck circuit, as before.
- **M6:** `R1` confirmed still `Resistor_SMD:R_0603_1608Metric` — unchanged.
- **M7 (D+/D−):** spot-checked `J2`; `A6`/`A7`/`B6`/`B7` (`D+`/`D−`) each confirmed on their own `unconnected-(J2-...)` net — still no DCP signature on downstream ports.
- **H3 (PD contract):** `U2` (CH224K) config pins unchanged — `CFG1` (pin 9) and `CFG3` (pin 3) still tied to `Earth`, `CFG2` (pin 2) still tied to `VDD`. The 15 V-contract recommendation from v2 (based on the measured ~0.75 A/port real load) still stands as an open option, not re-derived this pass since nothing here changed.
- **All PCB/layout findings** (H4 thermal vias, H6 non-Kelvin shunts, M9 mounting holes, M11 fragmented ground pour, M12/M13 capacitor voltage ratings, L1–L15) are unchanged — the `.kicad_pcb` file's timestamp (**2026-05-20**) hasn't moved.

---

## PCB/schematic parity — still needs a re-import before layout work

`kicad-cli pcb drc --schematic-parity` against the current schematic and the still-May-20 PCB: **60 issues** (7 missing footprints, 9 extra, 21 net conflicts, 21 footprint/symbol mismatches, 4 duplicates) — essentially the same shape as v2's 63, not better or worse in substance:

- **Missing footprints (7):** `D2`, `R60`, `R58`, `R59`, `C40`, `C41`, `C42` — all still without a home on the board.
- **New this pass:** `R50`'s footprint now shows up as an **extra footprint** (present on the PCB, absent from the schematic) — the direct fingerprint of N2's fix deleting `R50` from the schematic. Nothing to do about this except re-import; it's not a defect, it's confirmation the deletion took effect.

As before: this isn't a design defect on its own, it's that "Update PCB from Schematic" hasn't been run since the Aug 9 edits. But it does mean the PCB currently reflects **none** of Part 1 or Part 2 above.

---

## Suggested order of work

1. **N4** — swap `R36`/`R39` second legs back (`MODE`–`PGOOD` via 20 kΩ, `TRIP`–`GND` via 43 kΩ). Still the single highest-priority item: there is currently no working overcurrent trip on the buck.
2. **N5** — move `C42` pin 2 from `PGOOD` to `Earth`.
3. Re-import netlist into the PCB editor, place the now-7 orphaned footprints, remove/replace the orphaned `R50` footprint, re-run DRC + ERC.
4. Decide whether N2's `OE`/`IO9`/programming-header coupling is acceptable (it's benign, just worth a conscious yes/no) — no urgency.
5. Work back through Part 3 in roughly the same order v2 suggested: H5's `R_ILIM`, M1's CC network, M2's cascade pin, M3/M4's remaining ESP32 hygiene, M5's decoupling — then the PCB-side items (H4, H6, M9, M11), which are unaffected by any schematic work and can proceed in parallel once the board is re-imported.

---

## Sources

- `review.md` (2026-08-09, "Design Review v2") — baseline for every finding re-checked this pass; not re-derived where confirmed unchanged.
- This pass did not re-fetch the TPS548A20 or CH224K datasheets — N4's fix requirement and H3's contract math are unchanged from v2's sourced analysis, and nothing new emerged that needed new datasheet text. If N4 gets fixed and H7 needs re-deriving against a working `TRIP` threshold, that's the point to re-pull TPS548A20 §7.3.7 again.
