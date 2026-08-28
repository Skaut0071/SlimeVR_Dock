# SlimeVR_Dock — Design Review v4 (N4/N5 fix check)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-08-28
**Reviewing:** the **current uncommitted working tree** (`git status` shows `Board/StepDown.kicad_sch` and `Board/SlimeVR_Dock.kicad_pro` modified since `HEAD` = `07e5e92 Fixing from Review`) — i.e. the fixes made since `review_v3.md`, not yet committed.

---

## Result: both of v3's open findings are fixed, and nothing else changed

Re-exported the netlist and diffed it node-for-node against the netlist from v3. The entire connectivity delta on the whole board is these four moves:

```
Net-(U4-MODE):   removed R39 pin2   added R36 pin2
Net-(U4-TRIP):   removed R36 pin2   added R39 pin2
Net-(U4-PGOOD):  removed C42 pin2
Earth:                              added C42 pin2
```

That's it — no other net on the board changed. `R36`, `R39`, `C42` all still their original values (20 kΩ, 43 kΩ, 2.2 nF respectively), only their far-end connections moved. This is exactly, precisely the N4 and N5 fix from `review_v3.md`, applied cleanly and nothing else touched. (The 876-line raw diff in `StepDown.kicad_sch` is just KiCad rewriting wire-segment coordinates/UUIDs around that one crowded corner of the schematic when the two endpoints moved — not a sign of broader changes; the `.kicad_pro`'s `used_designators` bookkeeping field independently confirms only `R36` and `C42` symbols were touched.)

### N4 — TPS548A20 `MODE`/`TRIP`/`PGOOD` — ✅ fixed

| Component | Now wired | Matches TI's recommendation |
|---|---|---|
| `R36` (20 kΩ) | `MODE` (`U4` pin 21) — `PGOOD` (`U4` pin 2) | ✅ selects FCCM |
| `R39` (43 kΩ) | `TRIP` (`U4` pin 25) — `Earth`/GND | ✅ inside the valid 20–65 kΩ window, ground-referenced |

`TRIP` now has a real path to GND, so the converter has a working cycle-by-cycle overcurrent limit again. `MODE` is back on `PGOOD`, so the converter is back in FCCM as originally intended.

### N5 — input snubber cap `C42` — ✅ fixed

`C42` (2.2 nF) pin 2 is now on `Earth`, pin 1 on `/StepDown/Vin` — a clean `VIN`→`PGND` bypass right at the IC, as TI's layout guidance asks for.

---

## What's still open

Nothing from `review_v3.md`'s Part 3 was touched by this change (expected — it was a two-net surgical fix, not a broader pass). Still outstanding, unchanged:

- **H1** (AP22652 active-low part choice), **H5** (`R_ILIM` still 10 kΩ), **M1** (CC advertisement still 10 kΩ / 3 A code), **M2** (165 cascade `Q7`/`~{Q7}` pin), **M3** (no ESP32 `EN` cap), **M4** (`IO2`/`IO8` floating), **M5** (decoupling gaps), **M6** (`R1` still 0603), **M7** (D+/D− unconnected), **H3** (PD contract still 20 V-shaped config).
- **PCB/schematic parity**: `.kicad_pcb` is still the same **2026-05-20** file — now even further behind. Schematic-parity count moved slightly (58 vs. 60, from the renamed/rewired nets around `U4`) but is not materially different — still needs "Update PCB from Schematic" + a DRC re-run before any layout work.
- **H7 (inductor/OCP mismatch) is now actually derivable** — it was explicitly blocked on N4 in both v2 and v3 ("no working trip threshold to compare against"). Now that `TRIP` is properly grounded through 43 kΩ, the real overcurrent trip point can be computed from the TPS548A20's `I_TRIP`/current-sense equations and compared against `L1`'s 2.2 µH saturation current — that wasn't meaningful to do before this fix and hasn't been re-derived yet in this pass.

---

## Suggested next step

1. Re-import netlist into the PCB editor ("Update PCB from Schematic"), place the still-orphaned footprints (`D2`, `R58`, `R59`, `R60`, `C40`, `C41`, `C42`), remove the orphaned `R50` footprint, re-run DRC.
2. If you want the full picture on `H7` now that `TRIP` is real, that's the next worthwhile derivation — otherwise the priority order from `review_v3.md` (H5 → M1 → M2 → M3/M4 → M5) still stands.

---

## Sources

- `review_v3.md` (2026-08-28) — baseline for N4/N5; this pass re-traced only the nets those findings touched, via `kicad-cli sch export netlist` diffed programmatically against v3's parsed netlist (not visual inspection).
