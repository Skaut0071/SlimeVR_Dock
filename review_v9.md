# SlimeVR_Dock — Design Review v9 (inductor swap checked + full open-items sweep)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-09-18
**Purpose of this pass:** two things you asked for — (1) check the inductor swap you actually made in the working tree against `review_v8.md`'s recommendation, and (2) sweep every review in this series (`review.md` through `v8`) to build one consolidated status list, since the H7 sub-thread alone spans four passes (`v6`→`v9`) and it's easy to lose track of what's actually done vs. still open.

**Tooling note:** `kicad-cli` is not available in this environment, so nothing below that would normally come from `pcb drc`/`sch erc` was re-run — those items are marked "unverified this pass" rather than confirmed either way. Everything else was checked directly against the current `.kicad_sch`/`.kicad_pcb` text.

---

## H7 — the inductor swap needs a second look before you commit to it

**What `v8` recommended:** `XAL7030-222MEC` (real Coilcraft part, 18 A `I_sat`, 12.9 A `I_rms`, verified against Coilcraft's own datasheet) as the primary pick, with `IHLP-2525CZ-01` (14 A, zero footprint change) as an explicit fallback if you didn't want to touch the footprint.

**What's actually in the working tree right now:** neither of those. The footprint field was changed to `Inductor_SMD:L_Coilcraft_XAL7070-XXX`, and its `descr` still cites a real Coilcraft XAL7070 datasheet URL — but the part you linked (LCSC `C49261221`) is:

| Field | Value |
|---|---|
| Part number | `APS0730M2R2F` |
| **Manufacturer** | **Coilank** — not Coilcraft |
| Inductance | 2.2 µH |
| DCR | 13.7 mΩ |
| Rated current | 13 A (temp-rise basis not stated) |
| `I_sat` | 17 A (drop % not stated) |
| Body | 7.8×7.6 mm SMD |

Three problems, worth fixing before this goes to fab:

1. **Footprint/metadata mismatch.** The schematic and PCB still say "Coilcraft XAL7070" (name, `descr`, even the datasheet link) but the part you're actually using is a Coilank APS0730 — different manufacturer, different case (7.8×7.6 mm vs. Coilcraft's real 7.7×8.0×7.0 mm XAL7070), different land pattern. The land pattern currently on the PCB isn't even a real XAL7070 footprint either — it's the old `XAL7030` footprint with the pads manually stretched (1.78→1.92 mm wide, courtyard nudged 4.25→4.1 mm). That's very unlikely to match APS0730's actual pad geometry. Before ordering: pull the real Coilank APS0730 land pattern (or measure the part) and rebuild the footprint from that, and fix the `Footprint`/`descr`/datasheet fields so they describe the part that's actually there.
2. **Margin is thinner than it looks.** Your own bar was "`I_sat` > 17 A," set against the `~15.9 A` worst-case OCP trip calculated in `review_v6.md`. This part's `I_sat` is *exactly* 17 A — it doesn't clear your own bar, it sits on it, leaving under 7% headroom over the 15.9 A trip point. `XAL7030-222MEC` had 18 A against the same trip point (~13% headroom) with a datasheet-stated 30%-drop definition; Coilank doesn't state what drop % its 17 A figure uses, so the real margin could be smaller still.
3. **No independently-checkable datasheet.** I couldn't find a Coilank APS0730 datasheet PDF anywhere — not on Coilcraft's site (different company), not via general web search. The 17 A/13 A numbers above come only from the LCSC listing page, not a manufacturer datasheet with a stated test methodology. That's a real gap on a part whose whole justification is a saturation-current number sitting right at your margin.

**Recommendation — your call, not made for you:** either (a) go with `v8`'s actual recommendation, `XAL7030-222MEC` — real Coilcraft datasheet, 18 A with defined 30%-drop basis, and you're already redoing the footprint anyway so it's not extra layout work — or (b) if you want to keep the Coilank part for cost/availability reasons, get an actual datasheet from Coilank/the LCSC listing before committing, and rebuild the footprint from that part's real land pattern rather than a stretched XAL7030 pattern. Either way, fix the footprint name/`descr`/datasheet-link metadata to match whichever part actually ships.

---

## Consolidated status — every open item, `review.md` through `v8`

| # | Item | Last verdict | Status this pass |
|---|---|---|---|
| H1 | `AP22652` active-low EN | firmware-only note, no HW change (`v6`) | done — nothing to check in hardware |
| H3 | PD contract stays at 20 V | no wiring change needed (`v6`) | done — confirmed `CFG1`/`CFG2`/`CFG3` unchanged since `v6` |
| H4 | Thermal vias under `U4`/`U1` | present, but 0.2/0.2032 mm drill vs. 0.3 mm board rule (`v5`) | **still open** — confirmed: `min_through_hole_diameter` in `SlimeVR_Dock.kicad_pro` is still `0.3`, vias at `0.2`/`0.2032` mm still present. Decide fab capability, then resize vias or relax the rule. |
| H5 | `R6`/`R9`/`R12`/`R15` → 14.3 kΩ | fixed (`v6`) | **confirmed done** — all four (one hierarchical symbol, 4 sheet instances) read 14.3 kΩ |
| H6 | Shunt resistors not Kelvin-sensed | structural, needs new part or Kelvin routing (`v5`) | not re-verified this pass — no footprint change would fix this without deliberate action, worth a visual check if you haven't touched that area |
| H7 | `L1` inductor `I_sat` | in progress | see full section above — **not resolved**, footprint/part mismatch found |
| M1a | `R5`/`R8`/`R11`/`R14` Rp value | retracted, stay 10 kΩ (`v7`) | **confirmed correct** — all four still 10 kΩ, matches the retraction (not a defect) |
| M1b | Base resistors on `Q2`–`Q5` | optional hardening only (`v7`) | not verified — optional either way, your call |
| M2 | `U17` pin 7→9 wire move | stands (`v6`) | **not verified this pass** — worth a quick check next time you're in the schematic |
| M3 | 1 µF cap on `U1` `EN` | stands (`v6`) | **not verified this pass** |
| M4 | 10 kΩ pull-ups on `IO2`/`IO8` | stands (`v6`) | **not verified this pass** |
| M5 | 5× 100 nF decoupling caps | stands (`v6`) | **not verified this pass** |
| M6 | `R1` → 2010 footprint | stands (`v6`) | **confirmed done** — `R1` is `Resistor_SMD:R_2010_5025Metric`, value unchanged at 1k |
| M7 | `D+`/`D−` short on `J2`–`J9` | retracted, leave unconnected (`v7`) | **confirmed correct** — no `D+`/`D-` net naming found in `DUSBSS.kicad_sch`, consistent with staying unconnected |
| M9 | Mounting holes | absent (`v5`) | **confirmed still absent** — zero `MountingHole` footprints in `SlimeVR_Dock.kicad_pcb` |
| M11 | Fragmented ground pour | genuinely unknown, needs visual check (`v5`) | not re-checked — needs the PCB editor open, not a text-level check |
| M12/M13 | Capacitor voltage ratings | needs BOM-level check (`v5`) | not checkable from schematic/netlist; check against whatever parts actually get ordered |
| — | `BZ1` unrouted pad | ~1.1 mm short of pad (`v5`) | **not independently re-verifiable without DRC** — re-run `pcb drc` next time KiCad's available |
| — | Duplicate `R36` footprint | two copies on PCB (`v5`) | **confirmed fixed** — only one `R36` footprint found now |
| — | Silkscreen issues (`C27`, `C31`/`C13`, `R2`, `Q1`, `R7`) | cosmetic (`v5`) | not re-verified — needs DRC or visual check |
| — | `J1Temp1` pad 5 mismatch | needs confirming intentional-vs-wrong footprint (`v5`) | not re-verified this pass |

**Bottom line:** since `v5`/`v6`, four items are confirmed genuinely done (H5, M6, and the two retractions M1a/M7 holding correctly), one PCB cleanup item (duplicate `R36`) is confirmed fixed, and one (mounting holes, M9) plus one (thermal via drill size, H4) are confirmed still open with hard evidence either way. Everything else in the M2–M5 wiring/new-parts batch and the PCB cosmetic/DRC items hasn't been touched since whichever pass last found it — not because anything's wrong, just because nothing in this pass's diff (`SlimeVR_Dock.kicad_pcb`, `StepDown.kicad_sch`) touched that part of the board. Worth a `kicad-cli pcb drc --severity-all --schematic-parity` + `sch erc` pass next time you're at a machine with KiCad installed, to settle the "not verified this pass" rows in one shot.

---

## Sources

- Current working-tree `Board/StepDown.kicad_sch`, `Board/SlimeVR_Dock.kicad_sch`, `Board/DUSBSS.kicad_sch`, `Board/SlimeVR_Dock.kicad_pcb`, `Board/SlimeVR_Dock.kicad_pro` — read directly, not re-derived from prior passes' summaries.
- LCSC listing for `C49261221` (`APS0730M2R2F`, manufacturer Coilank) — only source found for this part's specs; no manufacturer datasheet PDF located.
- `review_v8.md` — baseline for the `XAL7030-222MEC` recommendation this pass checks the actual change against.
- `review_v6.md` — source of the ~15.9 A worst-case OCP trip point used as the margin reference above.
