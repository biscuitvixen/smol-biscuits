# Rev 2 parts, sourcing, and values

Everything about *what* goes on the rev 2 board and *what value* each part
is. The design map (net list, power tree, placement) is in
[rev2-nets.md](rev2-nets.md); the remaining build work is in
[rev2-todo.md](rev2-todo.md). Datasheets for every part are PDFs in this
folder.

Prices and stock checked 2026-07-19 against LCSC retail and JLCPCB assembly
inventory (separate pools — assembly stock is what matters for a PCBA
order). JLCPCB assembly is now optional: parts can be bought from
Digikey/Mouser and hand-soldered, so the Basic/Extended tiers below only
matter if you go back to JLC assembly.

## Direction

Rev 1 was a XIAO nRF52840 module carrier. Rev 2 is a custom nRF52840 +
Nordic nPM1300 design: the PMIC brings proper ship mode, single-button
power control, and USB charging into a wearable-sized, single-sided,
SMD-only board.

## Power button (nPM1300 SHPHLD)

One momentary switch between SHPHLD and GND does all power control; the
nPM1300 handles the rest internally. Short press wakes from ship mode,
~3 s enters ship mode via firmware over I2C, ~10 s is the hardware reset
bailout, USB attach also exits ship mode.

| Action | Result | Mechanism |
|---|---|---|
| Short press | Wake from ship mode | SHPHLD hardware |
| ~3 s press | Enter ship mode | Firmware, via I2C |
| ~10 s press | Hardware reset | nPM1300 long-press reset |
| USB attach | Exit ship mode | nPM1300 VBUS detect |

```
SHPHLD ──┬────── SW (TS-1088-AR02016) ────── GND
         │
         └────── TVS (PESD5V0S1BA) ───────── GND
```

Reference: `samples/pmic/native/npm13xx_one_button` in sdk-nrf, and the
nPM1300-EK schematic.

**Deliberately NOT included** — the nPM1300 handles these internally, do
not add: pull-up on SHPHLD, debounce capacitor, series resistor on the
switch line.

### Tactile switch — XUNPU TS-1088-AR02016 — LCSC C720477 — Basic

- SPST-NO SMD, 3.9 x 3.0 mm body, 2.0 mm tall, 160 gf, 50 mA / 12 V DC
- JLCPCB assembly stock ~484k, LCSC retail ~730k — no supply concern
- LCSC price: $0.056 @ 10, $0.045 @ 100; Basic part
- KiCad: symbol `Switch:SW_Push` (stock), footprint
  `smol-biscuits:SW_SPST_TS-1088-xR020` (copied from KiCad's official
  Button_Switch_SMD library; exact match for this part's AR020 height code)

### ESD protection — PESD5V0S1BA (HXY MOSFET) — LCSC C5261083 — Basic

TVS diode chosen over the RC-cap option: the button is finger-accessible on
a body-worn device, the textbook contact-ESD scenario, and the TVS costs
~2 cents. SHPHLD gets a real clamp instead of a charge divider.

- SOD-323, 5 V standoff, bidirectional, <1 ns response, ~200 W peak pulse
- JLCPCB stock ~121k–166k, ~$0.016 @ 50 (tiers start at 50); Basic part
- The sourcing brief asked for a *unidirectional* TVS; the PESD…BA family
  is bidirectional. That is fine (arguably preferable) for a switch-to-GND
  line, and the unidirectional UA variant has no Basic-part option.
- Genuine Nexperia (C19224) is Extended with only ~340 units at LCSC — the
  out-of-stock risk; the HXY second-source is the pick (low risk for a
  clamp diode). For a Digikey/Mouser build, the genuine
  `PESD5V0S1BA,115` is plentiful.
- KiCad: symbol `Device:D_TVS` (stock), footprint `Diode_SMD:D_SOD-323`

## IMU — ST LSM6DSV16X — LCSC C5267406 — Extended

Replaces the ICM-45686, which is unsourceable (1 unit in JLCPCB stock).

Selection logic: firmware (SlimeVR-Tracker-nRF) has drivers for BMI270,
ICM-42688, ICM-45686, ISM330BX, LSM6DSM, LSM6DSO, LSM6DSV. The SlimeVR IMU
comparison puts only the LSM6DSV family in the same "superior" tier as the
ICM-45686; ICM-42688 (temperature drift) and BMI270 (poor drift) are not
recommended, and ISM330BX has 10 units in stock. The only high-stock
ICM-42688P is an HXY-branded clone — rejected: a cloned MEMS die on the
precision part of a tracker is a false economy.

- LGA-14, 2.5 x 3.0 x 0.83 mm, I2C/SPI/I3C, embedded SFLP sensor fusion
- Genuine STMicroelectronics; JLCPCB assembly stock 2,883 (LCSC retail 175)
- $3.67 @ 10, $3.10 @ 100; Extended (~$3 one-time feeder fee)
- **Stock risk: moderate** — order sooner rather than later
- KiCad: symbol `smol-biscuits:LSM6DSV16X` (new, drawn from DS13510 Rev 4
  Figure 5 — pin mapping needs review before layout), footprint
  `smol-biscuits:LGA-14_3x2.5mm_P0.5mm_LayoutBorder3x4y` (KiCad's official
  ST LGA-14 land pattern)

## Magnetometer — QMC6309 (carried over from rev 1)

Unchanged part-wise from rev 1. One decoupling detail to get right: the
QMC6309 datasheet (§4.3.3 and Figure 6) specifies a **2.2 µF** low-ESR
ceramic reservoir cap on VDD (C11), not the generic 100 nF — it supplies
the set/reset pulses the sensor fires on every measurement. 0603 for the
low ESR. (This was caught and fixed in the value audit below.)

## PMIC — Nordic nPM1300-QEAA — LCSC C7501206 — Global Sourcing

Fixed by the rev 2 architecture (not a selection), but stock-checked:

- QFN-32-EP 5×5 mm; $2.36 @ 10, $1.83 @ 100
- **Stock risk: high.** 168 units at JLC via Global Sourcing (long lead),
  LCSC retail out of stock. Plan the buy early, or source the reel from a
  Nordic distributor and consign.
- KiCad: symbol `smol-biscuits:nPM1300` (new, from Product Spec v1.1
  Table 35), footprint `Package_DFN_QFN:QFN-32-1EP_5x5mm_P0.5mm_EP3.45x3.45mm`
  (stock; verify the EP size against PS §9.2.1 before layout)

## MCU — Nordic nRF52840-QIAA (bare chip) — LCSC C190794 — Basic

Module survey result: no JLCPCB-stocked nRF52840 module exposes RF for a
custom on-PCB antenna (Ebyte E73-S1C has its ceramic antenna on-module,
E73-S1CX is IPEX-only; Raytac/Minew/Holyiot not stocked). Since rev 2 wants
its own printed antenna, the bare aQFN73 chip is the route.

- **JLCPCB Basic**, 1,178 units — the deepest nRF stock available
- Circuit per nRF52840 PS v1.11 §7.3.5 (Config 5): VDD = 1.8 V from nPM1300
  BUCK1, VDDH shorted to VDD, REG1 DC/DC enabled (DCC → 15 nH → 10 µH →
  DEC4/6), USB enabled, NFC unused, 32.768 kHz crystal omitted (firmware
  uses LFRC)
- RF match per PS v1.1: 0.8 pF shunt, 4.7 nH series, 0.5 pF shunt → `RF_ANT`
  net → printed antenna (geometry + tuning are layout work, see todo)
- KiCad: stock `MCU_Nordic:nRF52840` symbol +
  `Package_DFN_QFN:Nordic_AQFN-73-1EP_7x7mm_P0.5mm` footprint
- Consequences owned: aQFN73 fanout (plan 4-layer, single-sided assembly),
  antenna tuning, and RF certification burden

## USB connector — Korean Hroparts TYPE-C-31-M-17 — LCSC C283540

Mid-mount (sink type, board cutout) 16-pin USB 2.0 Type-C, ~$0.09, chosen
for the low-profile wearable stack. KiCad stock footprint
`Connector_USB:USB_C_Receptacle_HRO_TYPE-C-31-M-17` (includes the board
cutout; the connector hangs in it, roughly halving height above board). No
CC resistors — the nPM1300 does Type-C detection (CC1/CC2 wired to it). SBU
pins no-connect, shell grounded. LCSC-sourced; the Digikey mid-mount
alternative (JAE DX07S016JA1R1500) is backordered and needs a different
footprint.

## Values verified against datasheets

Every passive value was cross-checked against the reference circuit in each
IC's datasheet (2026-07-20). All match by function; the only error found
was C11 (2.2 nF where the QMC6309 needs 2.2 µF — fixed). Reference designs:

- **nRF52840** — PS v1.11 §7.3.5 Config 5 (Table 79 + Figure 222): DEC1
  100 nF, DEC3 100 pF, DEC4/6 1 µF + 47 nF, DEC5 820 pF, DECUSB 4.7 µF,
  VBUS 4.7 µF, VDD 4.7 µF + 1 µF + 3×100 nF, crystal 12 pF ×2, DC/DC filter
  15 nH + 10 µH, RF match 0.8 pF / 4.7 nH / 0.5 pF. (DEC1 is 100 nF, not
  1 µF — the 1 µF nearby is on VDD; verified against the schematic figure.)
- **nPM1300** — PS v1.1 §9.3.2 Configuration 2 (Table 40 + Figure 56): VBUS
  1 µF, VSYS 2×10 µF, VBUSOUT 1 µF, VBAT 2.2 µF, VOUT1 10 µF, VDDIO 100 nF,
  sensor rail (LSOUT1) 1 µF, buck inductor 2.2 µH, VSET1 47 kΩ → 1.8 V.
- **LSM6DSV16X** — DS13510 §7.1 Figure 28: 100 nF on VDD + 100 nF on VDD_IO.
- **QMC6309** — §4.3.3 Figure 6: 2.2 µF reservoir on VDD.

### Ratings and tolerances that matter when you buy

Easy to miss because they're spec, not value:

| Part | Requirement | Source |
|---|---|---|
| R3 — VSET1 47 kΩ | **1 % tolerance** (it sets the 1.8 V rail) | nPM1300 Table 18/39 |
| C6 — VOUT1 10 µF | rate **≥10 V**, ESR ≤50 mΩ | nPM1300 Table 21 |
| C2/C3 — VSYS 10 µF | Nordic specs **25 V X5R** for margin (16 V works) | nPM1300 Table 40 |
| C18 — DEC5 820 pF | "not required for build code Fxx and later" — DNP-able on current nRF silicon | nRF52840 Table 79 note |
| R1/R2 — I2C 4.7 kΩ | keep 4.7 kΩ; ST suggests 10 kΩ but 4.7 kΩ suits a shared 4-device bus | ST Fig 28 |
| C26/C27 RF, C12/C13 loads | NP0/C0G dielectric, tight tolerance | Nordic BOMs |

Passive sizing follows the Nordic reference BOMs: sub-µF caps and all
resistors 0402, bulk ≥1 µF 0603 (DC-bias derating), RF parts 0402 NP0. Net
result 20×0402 + 16×0603 after the C11 fix.

## BOM integration

Symbols carry an `LCSC` field for the Fabrication Toolkit's JLCPCB BOM
export. The LSM6DSV16X already has `C5267406`; give the switch and TVS the
same field when placed: `C720477` (switch), `C5261083` (TVS).
