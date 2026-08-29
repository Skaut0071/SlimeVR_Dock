# SlimeVR_Dock — Design Review v7 (corrected for actual CC/data-line design intent)

**Board:** SlimeVR_Dock (Ver. 3), 8-port USB-C charging dock for SlimeVR trackers
**Reviewed:** 2026-08-28
**Purpose of this pass:** two of `review_v6.md`'s items were built on a wrong assumption about how this board's CC detection and data lines are actually meant to work. Once corrected against your actual design intent, they're **not defects** — retracted below, not just amended. Also: a genuinely hard push on the TPS548A20 RF-pin frequency table, with an honest account of where that search actually landed. Everything else from `review_v6.md` (H5, M2, M3, M4, M5, M6, H3, H1) is unchanged and still stands — not repeated in full here except where it interacts with the corrections.

---

## Retracted: M1a (CC advertisement resistor value)

**What I had wrong:** I treated `R5`/`R8`/`R11`/`R14` (10 kΩ) as a USB-C `Rp` current-advertisement code and flagged it as over-advertising 3 A against your real ~0.75 A/port delivery.

**What's actually going on, per your description:** the target device (the SlimeVR tracker) is a "dumb" sink — it just presents a fixed 5.1 kΩ `Rd` to GND on the CC line. It never reads your `Rp` value to negotiate current; it just expects 5 V once the port notices it's attached. This circuit's real job is **attach detection**, not standards-compliant current advertisement.

**Checked whether 10 kΩ still does that job well:** `Q2`–`Q5` (BC857, PNP) have their emitter on `+3V3` and base directly on `CC_Line`, with `R5` etc. as the pull-up to `+3V3`. Worked the actual bias point:

- **Unattached:** nothing loads `CC_Line`, so it sits at `+3.3 V` via `R5`. Base = Emitter → `V_BE` = 0 → transistor OFF. Clean, unambiguous "not present" state.
- **Attached (5.1 kΩ `Rd` to GND):** the base-emitter junction itself clamps the node to `~3.3 V − 0.65 V ≈ 2.65 V` once it starts conducting (this dominates over the simple resistor-divider math, since the transistor draws its own base current once forward-biased). At that point: `R5` supplies 65 µA, `Rd` sinks 520 µA, the transistor's own base current makes up the ~455 µA difference. Comfortably, unambiguously ON.

**Whether `Rp` = 10 kΩ vs 22 kΩ changes anything here:** reran the same numbers at 22 kΩ — the base current shifts from ~455 µA to ~490 µA, but the node still clamps to the same ~2.65 V either way. **The detection threshold is set by the transistor's own `V_BE`, not by `Rp`'s exact value**, as long as `Rp` is "large enough" relative to `Rd` to not overpower it (both 10 kΩ and 22 kΩ qualify by a wide margin against 5.1 kΩ). There's no functional reason to change this resistor for your actual use case.

**Decision: retracted. `R5`/`R8`/`R11`/`R14` stay at 10 kΩ, no change.**

---

## Kept, but re-justified: M1b (base resistor on Q2–Q5)

Checked whether the "no base resistor" wiring is actually a problem for *this* use case, separate from the USB-compliance framing I originally used it for. Worked the base current in normal attach operation (above): **~455–490 µA**, nowhere near the BC857's continuous base current rating (~200 mA) — not a component-stress issue in normal operation, and not something that breaks the detection function either (the circuit works fine electrically as analyzed above).

The only remaining reason to add it: **fault/ESD hardening on an exposed connector pin.** `CC` is a pin on a connector people plug things into — a resistor there is cheap insurance against a hot-plug transient or ESD event driving the base outside its safe range, something `R5` alone doesn't fully cover since it's in parallel with the base path, not in series with whatever might drive `CC` from outside. This is genuinely optional now, not a fix for a broken function.

**Decision: keep the 10 kΩ base resistors on `Q2`–`Q5` if you're doing a layout pass anyway (cheap, harmless, adds ESD margin) — but this is hardening, not a required fix. Your call, not flagged as a problem either way.**

---

## Retracted: M7 (D+/D− shorting for DCP signature)

**What I had wrong:** I assumed these 8 charging ports needed a BC1.2 "Dedicated Charging Port" signature (`D+`/`D−` shorted) so that BC1.2-aware devices would recognize them as charge-capable.

**What's actually going on, per your description:** these ports are charge-only by design — the trackers don't use USB data at all, and whatever device *does* need real USB data (programming, serial monitoring) connects through a **separate** connector wired directly to the ESP32-C3's own D+/D− pins (`J1_Prog1`, already confirmed wired to `U1` `IO18`/`IO19` earlier in this review series), not through any of `J2`–`J9`. Your charging-detection path is the CC/`Rd` circuit above, not BC1.2 — so there's nothing for a DCP signature to do here, and no legacy device is expected to check for one.

**Decision: retracted. Leave `D+`/`D−` unconnected on all 8 ports — this was already correct, not a defect.**

---

## H7 — the RF-pin frequency table search, done properly this time

You asked me to try harder, including non-TI sources, and accept it taking a while. Here's exactly what that turned up:

**What I did:**
1. Re-confirmed the retail PDF's broken cross-reference and TI's official HTML datasheet viewer independently (done in `review_v6.md`).
2. Searched TI's E2E support forums for TPS548A20/TPS548A28/TPS548D22 threads discussing RF pin configuration — several exist, but the forum blocks automated fetching (403) and search snippets didn't surface the table content.
3. Found and downloaded TI's own **TPS549A20EVM-737 evaluation module user's guide** (SLUUBE0A) — the TPS549A20 is confirmed pin/register-compatible with our TPS548A20 (the guide's own variant table shows the only differences are pins 26–28, which add PMBus/telemetry features not present on our plain 548A20 — `RF`, `MODE`, `TRIP`, `PGOOD` etc. are identical between the two parts).
4. That EVM's schematic uses the **exact same topology** our board does: a high-side resistor from `VREG` and a low-side resistor to `AGND` feeding the `RF`/`ADDR` pin — confirmed by checking which of our `R40`/`R41` is which (`R40` = 240 kΩ to `VREG`, `R41` = 75 kΩ to `Earth` — matches the EVM's `R3`(high)/`R4`(low) pattern exactly).
5. The EVM guide's **Table 6-1** does document a resistor-divider-ratio lookup table for this exact pin — but on the TPS549A20, that same physical pin/ADC mechanism is quantized into **15 bins for a 7-bit PMBus address**, not the plain TPS548A20's simpler **8 bins for switching frequency**. The two parts share the pin and the underlying resistor-divider-to-ADC mechanism, but not necessarily the same bin boundaries, since a 7-bit address table and a 3-bit frequency table are different lookup tables. Using the address table's thresholds to infer our frequency bin would be presenting a guess as a measurement — I'm not willing to do that.

**What I can honestly tell you:** our board's ratio, `R41/(R40+R41) = 75/315 ≈ 0.238`, sits noticeably lower than the EVM's own reference choice (`R4/(R3+R4) = 300/301 ≈ 0.997`, at the extreme top of that pin's range) — meaning whoever designed this board picked a meaningfully different operating point than TI's own reference design, consistent with a mid-range rather than extreme switching frequency, but **I cannot tell you which specific one of the 8 frequencies, or even confirm the direction (higher ratio → higher or lower frequency) without the actual 548A20-specific table**, which does not appear to exist in any TI-published document I can access — retail datasheet, official HTML viewer, or the closest available EVM guide.

**This is the honest ceiling of what's findable here.** The `I_OCP` range from `review_v6.md` stands unchanged: **~10.3–15.9 A** (at the 20 V contract), bounding the answer across the device's entire published frequency range rather than guessing a single point. The recommendation is also unchanged: **verify your actual `L1` part's `I_sat` is ≥ ~17 A.**

---

## What's unchanged from `review_v6.md` (not repeated in full)

- **H5:** `R6`/`R9`/`R12`/`R15` → 14.3 kΩ. Still stands — this was AP22652 current-limit math, unrelated to CC detection.
- **M2:** move `U17` pin 7's wire to pin 9. Still stands — 165 cascade wiring, unrelated.
- **M3, M4, M5:** new decoupling/pull-up parts. Still stand — ESP32 hygiene, unrelated to CC/data-line questions.
- **M6:** `R1` → 2010 footprint. Still stands.
- **H3:** staying at 20 V contract, no wiring change. Still stands.
- **H1:** firmware-only polarity note, no hardware change. Still stands.

---

## Updated action list

| # | Component(s) | Change | Status |
|---|---|---|---|
| H5 | `R6`,`R9`,`R12`,`R15` | 10k → 14.3k | stands |
| M1a | `R5`,`R8`,`R11`,`R14` | ~~10k → 22k~~ | **retracted — no change** |
| M1b | base resistors on `Q2`–`Q5` | 10k, new parts | **optional hardening only, not required** |
| M2 | `U17` pin 7 wire | move to pin 9 | stands |
| M3 | `U1` EN cap | new 1µF | stands |
| M4 | `U1` IO2/IO8 pull-ups | new 10k ×2 | stands |
| M5 | decoupling caps | new 100nF ×5 | stands |
| M6 | `R1` | 1kΩ → 2010 footprint | stands |
| M7 | `D+`/`D−` on `J2`–`J9` | ~~short per port~~ | **retracted — leave unconnected, was already correct** |
| H3 | PD contract | stay at 20V | stands, no action |
| H1 | load switch polarity | firmware note only | stands, no action |
| H7 | `L1` | verify `I_sat` ≥ 17A | stands, BOM check |

Net effect of this pass: two items came off the list entirely (M1a, M7), one downgraded from required to optional (M1b), nothing new added.

---

## Sources

- `review_v6.md` — baseline for everything not revisited here.
- TPS549A20EVM-737 User's Guide (SLUUBE0A, Texas Instruments) — schematic and Table 6-1, used to confirm the RF-pin resistor-divider topology and to establish (honestly) the limits of what's determinable about the frequency table.
- BC857 datasheet — base current absolute maximum rating, used to confirm the no-base-resistor bias point (~455–490 µA) is not a component-stress concern.
