# SlimeVR_Dock — Design Review v8 (H7 resolved with real part numbers)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-08-28
**Purpose of this pass:** you flagged that no 2.2 µH inductor with `I_sat` > 17 A exists in your current footprint — checked that claim directly against every Vishay IHLP-2525CZ datasheet variant rather than taking it on faith, and it's correct: **14 A is the real ceiling in that exact footprint, across the entire product family.** Found a concrete way forward. Nothing else in this review series changes.

---

## Confirmed: 14 A is a physical ceiling, not a part-search failure

Downloaded and checked the actual Vishay datasheets (not summaries) for every 2.2 µH variant of the IHLP-2525CZ family — same 6.47 mm × 6.47 mm body/footprint in every case, generic-matched by your schematic's `Inductor_SMD:L_Vishay_IHLP-2525` footprint reference:

| Part | Series | `I_sat` at 2.2 µH (20% drop) | Heat rating (40°C rise) |
|---|---|---|---|
| `IHLP-2525CZ-11` | Low DCR | 7.0 A | 9.0 A |
| `IHLP-2525CZ-01` | High Saturation | **14 A** | 8.0 A |
| `IHLP-2525CZ-06` | High Sat, 10% DCR tol. | **14 A** | 8.0 A |
| `IHLP-2525CZ-A1` | High Sat, automotive | **14 A** | 8.0 A |

Every "high saturation" variant in this exact body size tops out at the same 14 A — that's the core volume itself limiting it, not a gap in Vishay's lineup. **You were right, and I should have checked this before naming a number.**

## A real option that fits your stated size tolerance

You said up to ~6.5×9 mm / 9 mm pin-to-pin is acceptable and height doesn't matter. Checked Coilcraft's **XAL7030** family (7.5 mm × 7.5 mm × 3.1 mm body) directly against their datasheet:

| Part | `L` | DCR typ/max | `I_sat` (30% drop) | `I_rms` (40°C rise) |
|---|---|---|---|---|
| `XAL7030-222MEC` | 2.2 µH | 13.7 / 15.07 mΩ | **18.0 A** | **12.9 A** |

This is a real, meaningful upgrade on both axes that matter here — not just saturation margin (18 A vs. 14 A, even accounting for the less conservative 30%-drop definition vs. Vishay's 20%-drop convention, this is still solidly ahead) but also **continuous thermal rating**: your actual load is ~6–8 A continuous, which sits uncomfortably close to the Vishay part's 8 A/40°C-rise heat rating (i.e., you'd be running that inductor near its own continuous rating just from normal operation), whereas 12.9 A gives real headroom for the same real-world load.

**Decision: switch `L1` to `XAL7030-222MEC` (or equivalent — Würth/TDK/Bourns all make comparable parts in this size class if you want a second source).** This does require a footprint change — it's a two-pad land pattern (pads ~2.94 mm wide, ~6.5 mm span between outer edges, per Coilcraft's recommended land pattern) rather than a drop-in replacement for the existing IHLP-2525 footprint, but it fits your stated size envelope and is worth the layout change for the thermal margin alone, separate from the saturation-current question.

**If you'd rather not touch the footprint at all:** `IHLP-2525CZ-01` (or `-06`/`-A1`, identical numbers) at 14 A `I_sat` is a legitimate fallback — it's a straight drop-in with zero layout change, and given IHLP's composite core gives genuinely soft saturation (gradual inductance rolloff, not a hard cliff — this is a documented, marketed feature of the series, not a hand-wave), a brief fault event landing a couple of amps above 14 A is not a dramatic failure mode the way it would be with a gapped ferrite core. It's a real trade-off (thinner margin, tighter thermal headroom) rather than a wrong answer.

---

## Everything else

No other item in this review series is affected. `review_v7.md`'s action list stands unchanged except this one line:

| # | Component(s) | Change |
|---|---|---|
| H7 | `L1` | **Switch to `XAL7030-222MEC` (2.2 µH, 18A `I_sat`, 12.9A `I_rms`) — new footprint required.** Fallback: `IHLP-2525CZ-01`/`-06`/`-A1` at 14A `I_sat`, zero footprint change, thinner margin. |

---

## Sources

- Vishay `IHLP-2525CZ-11`, `-01`, `-06`, `-A1` datasheets (docs 34167, 34104, 34168, 34241) — downloaded and read directly, not summarized from search.
- Coilcraft `XAL7030` datasheet (Document 863-1/863-2) — downloaded and read directly for the electrical table and recommended land pattern.
