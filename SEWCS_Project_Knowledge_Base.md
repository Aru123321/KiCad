# SEWCS — NIR Recyclability Sensor · Project Knowledge Base

> Living design document. Update this as decisions change so any future session (or teammate) has full context.
> **Last updated:** 2026-06-29

---

## 1. Concept

A **near-infrared (NIR) reflectance sensor** that identifies an item's material — and therefore whether it's recyclable — at the moment of disposal. Mounted at a bin's rim, an item is scanned before it's thrown away, reducing recycling-stream **contamination** (wrong materials in the recycling bin).

**Why NIR works:** Common plastics have distinct C–H / aromatic absorption features in the ~1000–1700 nm band:
- **PET** — strong band near ~1660 nm
- **HDPE / LDPE / PP** (polyolefins) — C–H overtones near ~1200 / 1400 / 1700 nm
- **PS** — aromatic feature near ~1680 nm

By shining several discrete NIR wavelengths at the item one at a time and measuring how much each reflects, we build a small spectral signature and classify the material.

**Architecture principle:** sequential LEDs (one wavelength at a time) + a single photodiode. Cheaper and simpler than a true spectrometer, and sufficient to separate the main plastics.

---

## 1a. Files in this folder

| File | Role |
|------|------|
| `SEWCS.kicad_*` | **Prototype/MVP** schematic — Pi 5 HAT + Thorlabs lab parts. Pristine reference; do not gut. |
| `SEWCS_Commercial_v1.kicad_*` | **Commercial board** — standalone ESP32, SMD, JLCPCB parts. Active development. |
| `prototype_backup/` | Safety copy of the original prototype files. |
| `SEWCS_Commercial.*` | Orphaned empty project files (backend-locked, ignore/delete). |
| `SEWCS_Project_Knowledge_Base.md` | This document. |

> Decision (2026-06-28): keep the original `SEWCS` intact for prototyping/MVP; build the commercial design in a **separate** `SEWCS_Commercial_v1` project.

## 2. Two design tracks

| Track | Purpose | Compute | Optics | Board |
|-------|---------|---------|--------|-------|
| **A — Prototype** | Prove the concept + classifier | Raspberry Pi 5 + breadboard | Thorlabs **lab-grade** (TO-can) | No PCB needed |
| **B — Commercial** | Mass-production product | **Onboard ESP32** (no Pi per unit) | **Commercial** (sourced, off-board head) | Custom 2-layer PCB, JLCPCB-assembled |

> Decision (2026-06-28): **Skip the prototype PCB.** Prototype on breadboard + Pi. The next board *is* the commercial one, designed around cheap sourceable parts — not the Thorlabs lab parts.

---

## 3. Prototype (Track A) — parts already ordered

| Ref | Part | Spec | Pkg | ~Cost |
|-----|------|------|-----|-------|
| D1 | Thorlabs **LED910L** | 910 nm, 10 mW | TO-18 | ~$10 |
| D2 | Thorlabs **LED1085L** | 1085 nm, 5 mW | TO-18 | ~$30 |
| D3 | Thorlabs **LED1300L** | 1300 nm, 3.5 mW | TO-18 | ~$31 |
| D4 | Thorlabs **LED1600L** | 1600 nm, 2 mW | TO-18 | ~$30 |
| D5 | Thorlabs **FDG05** | Ge photodiode, 800–1800 nm, Ø5 mm | TO-8 | ~$310 |
| U? | **OPA380AID** | Transimpedance amp | SOIC-8 | ~$6 |
| — | **ADS1115** | 16-bit I²C ADC | — | ~$2 |
| — | **Raspberry Pi 5 (4GB)** + PSU | Host compute | — | ~$125 |

**Wavelength rationale:** 910 / 1085 / 1300 / 1600 nm is a good 4-point spread; 1600 nm in particular gives strong PET/aromatic discrimination. The Ge photodiode (800–1800 nm) covers all four — a proper NIR pairing.

> The existing KiCad project `SEWCS.kicad_sch` is built around these lab parts. It now serves as the **breadboard reference schematic**, not the production design.

**Note on lab optics:** FDG05 leads are epoxied (don't solder with high heat). For a reflectance probe, all LEDs + photodiode must face the item with the photodiode centered and **optical isolation** blocking direct LED→detector glare.

---

## 4. KEY CONSTRAINT — NIR optics are not fab-assemblable

Checked JLCPCB's live catalog (616,593 parts, 2026-06-28):
- **No InGaAs photodiodes. No Ge photodiodes. No NIR LEDs above ~1100 nm.**
- Their optical catalog stops at **silicon photodiodes (~1100 nm)** and **940 nm IR emitters**.

**Implication:** The true-NIR optics (1300/1600 nm LEDs + InGaAs/Ge detector) **cannot be machine-placed** by the fab. They must be sourced separately and hand-mounted.

**Design response:** Put the optics on an **off-board connectorized optical head**. The main PCB carries all electronics (fab-assembled); the optics plug in via a header. Benefits:
1. Fab assembles everything it can; you hand-mount only the specialty optics.
2. The optical head can be aimed at the item (mechanical geometry, not flat-PCB).
3. Swapping the production detector (e.g., lab Ge → commercial InGaAs) needs **no board respin**.

**Commercial optics sourcing (off-board, ~$15–80 total vs ~$450 lab):**
- Detector: commercial **InGaAs photodiode** (~900–1700 nm), ~$5–40/vol — Marktech / Everlight / Hamamatsu.
- LEDs: commercial NIR LEDs — 910 nm is <$0.50 (Osram/Vishay); 1300/1600 nm ~$2–15/vol (Marktech/Roithner).
- **Strategy:** use the prototype to find *which* wavelengths actually carry the discriminating signal, then production buys only those.

---

## 5. Commercial board (Track B) — architecture

Standalone, USB-C powered, ESP32-based. 2-layer, SMD, JLCPCB PCBA.

```
USB-C (5V) ──► AMS1117-3.3 LDO ──► 3V3 (digital)
                         └─(ferrite)─► 3V3A (clean analog rail)
USB D+/D- ──► CH340C USB-UART ──► ESP32 TXD0/RXD0
                         └─ auto-reset (2× transistor) ──► EN / IO0

ESP32-WROOM-32E
   ├─ GPIO ×4 ──► gate R ──► 2N7002 ──► [LED+ header]   (4 NIR LED driver channels)
   ├─ I2C (SDA/SCL) ──► ADS1115 (16-bit ADC)
   └─ buttons: BOOT (IO0), RESET (EN); status LED

Optical head connector (off-board):
   ├─ 4× LED drive returns (+ shared anode to 3V3/5V)
   └─ photodiode anode/cathode ──► OPA380 TIA ──► ADS1115 AIN0

OPA380 TIA: photodiode current → voltage; feedback R + small C; biased at mid-rail for single-supply.
```

### Commercial BOM (assemblable — JLCPCB part numbers)

| Block | Part | JLCPCB | Pkg | ~$/ea |
|-------|------|--------|-----|-------|
| MCU | ESP32-WROOM-32E module (ESP32-32E-N4) | C19949066 | SMD module | 2.90 |
| TIA | OPA380AIDGKR | C2057355 | VSSOP-8 | 2.83 |
| ADC | ADS1115IRUGR | C2651328 | X2QFN-10 | 1.78 |
| USB-UART | CH340C | C84681 | SOP-16 | 0.53 |
| LED driver ×4 | 2N7002 | C16338 | SOT-23 | 0.03 |
| LDO | AMS1117-3.3 | C18906017 | SOT-223 | 0.03 |
| USB-C | (16-pin Type-C receptacle — TBD) | TBD | SMD | ~0.10 |
| Passives | R/C 0402–0805, I²C pull-ups, decoupling | various | SMD | — |

> **Off-board (NOT on PCBA):** 4× NIR LEDs, 1× InGaAs/Ge photodiode → mount on optical head, wire to header.

### Estimated cost (worked, 2026-06-28 — validate by uploading Gerbers)

**Per-board electronics parts (JLCPCB):**

| Item | $/board |
|------|---------|
| ESP32-WROOM-32E | 2.90 |
| OPA380 TIA | 2.83 |
| ADS1115 ADC | 1.78 |
| CH340C | 0.53 |
| AMS1117-3.3 | 0.03 |
| 4× 2N7002 | 0.12 |
| USB-C conn | 0.10 |
| ~22 passives (R/C 0805) | 0.45 |
| Indicator LED, ferrite, 2 buttons, ESD diode | 0.30 |
| **Parts subtotal** | **≈ $9.0** |

**One-time:** ~7 extended parts × $3 setup = **$21**; SMT stencil/setup ≈ **$8–10**.
**Bare 2-layer PCB (~50×60 mm):** ~$2 for 5 pcs; ~$15–25 for 100 pcs (+ shipping ~$6–25).

**Assembled-board scenarios (electronics only, excl. optics):**

| Qty | ≈ $/board (electronics) |
|-----|--------------------------|
| 5 | ~$18 (setup dominates) |
| 100 | ~$9–10 |
| 1000 | ~$7.5 |

**Off-board optics (sourced + hand-mounted), per board:**
- 4 NIR LEDs: 910 nm ~$0.50; 1085/1300/1600 nm ~$2–15 ea → ~$10–45 low vol.
- InGaAs photodiode: ~$5–40.
- → ~$15–85/board low vol; ~$10–30 at volume (and cheaper if feature-selection drops channels).

**Per-unit at ~1000 qty: ≈ $7.5 electronics + ~$15 optics ≈ $22/unit** (vs ~$80+ if every unit needed a Pi). The optics dominate cost at scale → prioritize wavelength feature-selection on the prototype.

---

## 6. Decision log

- **2026-06-28** — Concept confirmed: sequential NIR LEDs + single photodiode, bin-rim mount, reduce contamination.
- **2026-06-28** — Prototype = breadboard + Pi 5 + Thorlabs parts. No prototype PCB.
- **2026-06-28** — Commercial board = **standalone ESP32** (not a Pi HAT); a Pi per unit is too expensive at scale.
- **2026-06-28** — Parts chosen for **JLCPCB assembly** so the fab quote reflects a fully-built board.
- **2026-06-28** — NIR optics **not fab-assemblable** → off-board connectorized optical head; main board fab-assembled.
- **2026-06-28** — Keep dedicated **ADS1115** (16-bit) rather than ESP32's noisy internal ADC, for signal quality.
- **2026-06-29** — **Optics belong on a SEPARATE optical head, not the main board.** Main board should expose a ~10-pin connector (4 LED drives + shared anode, photodiode ±, GND) instead of carrying the optic footprints. See §9.
- **2026-06-29** — **Freeze `v1` as the cost-estimate milestone.** Next board revision = a *new* `v1.1` file (don't edit v1 in place). Hold v1.1 until after bench validation so all informed changes batch into one revision.

## 7. Open questions / TODO
- Confirm USB-C connector part + whether USB-C is power-only or power+data (programming).
- Confirm LED drive rail (3V3 vs 5V) — depends on chosen commercial NIR LED forward voltages.
- Pick optical-head connector pinout (≥ 4 LED drives + PD ±, plus power/GND).
- Decide ADC reference / TIA mid-rail bias scheme.
- After prototype: feature-select wavelengths → finalize production LED set.

## 8. Status — COMMERCIAL BOARD COMPLETE (updated 2026-06-29)

The commercial board `SEWCS_Commercial_v1` is **schematic-complete, placed, routed, DRC-clean, and fab-exported.**
- Schematic: ERC 0 errors.
- PCB: 115×80 mm, 2-layer, 50 footprints, all 37 nets routed (375 trace segments), GND pours both layers.
- DRC: **1 advisory** only (ESP32 antenna keep-out grazes one hand-mounted LED courtyard; that LED is off-board on the optical head in the real product). No shorts, no unrouted nets. Clearance 0.15 mm / drill 0.2 mm — within JLCPCB capability.
- Routed with Freerouting 2.2.4 (needs Java 21+; installed OpenJDK 26).

**Fab package** (in `fab/`): `SEWCS_Commercial_v1-Gerbers.zip` (Gerbers + drill), `-CPL.csv` (placement),
`-BOM.csv`, `-board.svg` (render). Upload the Gerber zip + BOM + CPL to JLCPCB for an exact quote.

> Note: layout is functional but **loose** (115×80 mm). A compact re-place could shrink it well under
> 100×100 mm for a cheaper bare-board tier — a worthwhile optimization before production.

## 9. Optics architecture & roadmap (decided 2026-06-29)

### Decision: optics go on a separate optical head, not the main PCB
The reflectance-probe geometry — photodiode centred, LEDs ringed around it and tilted inward,
a **baffle wall** blocking direct LED→detector glare, and an **ambient-light hood** — is
fundamentally **mechanical/3D, not a flat-PCB problem**. Conclusions:
- The main electronics board should carry a **single connector** (~10 pins: 4 LED drives +
  shared LED anode rail, photodiode anode/cathode, GND) — **not** the optic footprints.
- The optics mount on a small dedicated **optical head** (tiny PCB or 3D-printed carrier) with
  the baffle + hood, cabled back to the main board.

**Why:** (1) the geometry is mechanical and best iterated as a 3D-printed head; (2) a bin-rim
sensor wants the electronics sealed in an enclosure while only the head faces the (wet, dirty)
bin throat — the standard sensor-head/controller split; (3) the head can be respun cheaply
without touching the ~$9 electronics board; (4) it's cleaner than the current board.

### Current-state caveat (important)
`SEWCS_Commercial_v1` currently has the **5 optic footprints scattered as loose pads** across the
board. That was a stopgap: the intended photodiode-centred ring caused a real DRC **short**
(an LED cathode bridging GND) plus courtyard overlaps because the placeholder footprints
overlapped, so they were spread out to get a clean route. The result is **neither a ring nor a
connector** — it routes and quotes fine, but it is **not** the production shape. Production shape
= replace those 5 footprints with the head connector (per the decision above).

### Roadmap & sequencing (do NOT respin the PCB next)
1. **Freeze `v1`** as the cost-estimate artifact (committed). Don't edit in place.
2. **Validate on the breadboard first** (Pi + Thorlabs parts): wire the front-end, then collect
   spectral data on target materials. This determines **which wavelengths actually separate the
   materials**, plus the real **spacing / working distance / baffle** geometry. *Everything
   downstream depends on these numbers.* (Linear PRA-12, PRA-14.)
3. **Then create `v1.1` once**, batching all bench-informed changes together: optics→connector,
   compact re-layout (shrink < 100×100 mm for a cheaper JLCPCB tier), final wavelength set, and
   any commercial-part swaps. One clean revision beats 2–3 incremental respins.
4. **Design the optical head** (small PCB + 3D-printed baffle/hood) around the validated geometry.

> Rationale for *not* doing v1.1 now: the concept isn't bench-validated yet, and several changes
> (which wavelengths to keep, head pinout) are gated on data we don't have. Polishing the board
> before the breadboard proves material separation is premature. Validate → batch → respin once.

### Older status log (2026-06-28, ~1:30pm)
- [x] Design tracks defined; commercial architecture set.
- [x] JLCPCB parts feasibility checked.
- [x] Original `SEWCS` restored to pristine prototype; commercial work isolated in `SEWCS_Commercial_v1`.
- [x] **Commercial schematic — COMPLETE & ERC-CLEAN (0 errors).** All sections wired & verified.
- [ ] PCB layout + routing — **next**.
- [ ] Export Gerbers/BOM/CPL for fab cost quote.

> ERC note: 0 errors. ~200 remaining warnings are cosmetic only (symbol-vs-library cache
> mismatch, off-grid pins from mm placement, NC-flag notices) and do not affect the netlist
> or fab. A tooling quirk (connect_to_net + move created spurious cross-wires) was cleaned up
> via script; net connectivity is by pin-snapped labels.

### Resume notes — where wiring left off
Work file: `SEWCS_Commercial_v1.kicad_sch`. Reused analog/LED/ADC block keeps its prototype
net labels (GPIO17/27/22/23 for LED gates, I2C_SDA/SCL, PD_SIG→TIA_OUT, +3V3/-3V3/GND).
New blocks added: U3 ESP32-WROOM-32E, U4 CH340C, J3 USB-C, U5 AMS1117-3.3, U6 ICL7660 (−3V3
charge pump), plus C5–C15, R12–R16, SW1/SW2, D6.

**Done wiring:** J3 (USB-C: VBUS→+5V, GND, CC1/CC2 via R14/R15, D±→USB_DP/DM), U5 (AMS1117
pins), and the first snap-to-pin labels (R14/1, U6/8, U3/2).

**Tooling gotchas for next session:**
- `batch_add_components` IGNORES x/y → parts land at (0,0); fix with `move_schematic_component`
  (takes `position:{x,y}`).
- `connect_to_net` has a stale index for batch-added parts → use
  `add_schematic_net_label` with `componentRef`+`pinNumber`+`netName` (snaps to pin, reliable).

**Remaining wiring (net = pin):**
- U6 ICL7660: 3→GND, 5→-3V3, 6→GND(LV), 2→CP_PLUS, 4→CP_MINUS; C8 1→CP_PLUS/2→CP_MINUS;
  C9 1→-3V3/2→GND.
- AMS caps: C5 1→+5V/2→GND; C6,C7 1→+3V3/2→GND.
- USB-C CC: R14 2→GND; R15 1→CC2/2→GND. Power LED: R16 1→+3V3/2→PWR_LED; D6 2→PWR_LED/1→GND.
- ESP32 U3: 1→GND, 3→EN, 25→IO0, 35→ESP_TX, 34→ESP_RX, 28→GPIO17, 12→GPIO27, 36→GPIO22,
  37→GPIO23, 33→I2C_SDA, 31→I2C_SCL. Decoupling C10/C11/C12 1→+3V3/2→GND.
  EN: R12 1→+3V3/2→EN, C13 1→EN/2→GND, SW1 1→EN/2→GND. IO0: R13 1→+3V3/2→IO0, SW2 1→IO0/2→GND.
- CH340 U4: 16→+5V, 1→GND, 4→V3_CH340, 5→USB_DP, 6→USB_DM, 2→ESP_RX, 3→ESP_TX;
  C14 1→V3_CH340/2→GND, C15 1→+5V/2→GND.
- Then: ERC → sync_schematic_to_board → board outline (~50×60mm) + optics ring region →
  place → route → DRC → export Gerbers/BOM/CPL.

**Design notes / TODO to verify:** auto-reset transistors omitted (manual BOOT+RESET flashing
for v1). NIR optics (D1–D5) are hand-mounted, not in JLCPCB PCBA. TIA uses ±3V3 (−3V3 from U6).
