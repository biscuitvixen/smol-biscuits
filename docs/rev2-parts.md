# Rev 2 part selections: power button + IMU

Sourcing decisions for the rev 2 (nRF52840 + nPM1300) redesign. Prices and
stock checked 2026-07-19 against LCSC retail and JLCPCB assembly inventory
(separate pools — assembly stock is what matters for PCBA orders).

## Power button (nPM1300 SHPHLD)

One momentary switch between SHPHLD and GND. Short press wakes from ship
mode, ~3 s long press enters ship mode via firmware over I2C, ~10 s is the
hardware reset bailout. USB attach also exits ship mode.

### Tactile switch — XUNPU TS-1088-AR02016 — LCSC C720477 — Basic

- SPST-NO SMD, 3.9 x 3.0 mm body, 2.0 mm tall, 160 gf, 50 mA / 12 V DC
- JLCPCB assembly stock ~484k, LCSC retail ~730k — no supply concern
- LCSC price: $0.056 @ 10, $0.045 @ 100
- Basic part: no extended-part setup fee
- KiCad: symbol `Switch:SW_Push` (stock), footprint
  `smol-biscuits:SW_SPST_TS-1088-xR020` (copied from KiCad's official
  Button_Switch_SMD library; exact match for this part's AR020 height code)

### ESD protection — PESD5V0S1BA (HXY MOSFET) — LCSC C5261083 — Basic

TVS diode chosen over the RC-cap option: the button is finger-accessible on
a body-worn device, which is the textbook contact-ESD scenario, and the TVS
costs ~2 cents. SHPHLD gets a real clamp instead of a charge divider.

- SOD-323, 5 V standoff, bidirectional, <1 ns response, ~200 W peak pulse
- JLCPCB stock ~121k–166k, ~$0.016 @ 50 (tiers start at 50)
- Basic part
- Note: the sourcing brief asked for a *unidirectional* TVS; the PESD…BA
  family is bidirectional. That is fine (arguably preferable) for a
  switch-to-GND line, and the unidirectional UA variant has no Basic-part
  option at JLCPCB.
- Genuine Nexperia (C19224) is Extended with only ~340 units at LCSC —
  flagged as the out-of-stock risk; the HXY second-source is the pick.
  Low risk for a clamp diode (unlike precision parts).
- KiCad: symbol `Device:D_TVS` (stock), footprint `Diode_SMD:D_SOD-323`
  (stock)

### Deliberately NOT included

The nPM1300 handles all of this internally — do not add:

- pull-up on SHPHLD (internal pull-up)
- debounce capacitor (internal debounce)
- series resistor on the switch line

## IMU — ST LSM6DSV16X — LCSC C5267406 — Extended

Replaces the ICM-45686, which is unsourceable (1 unit in JLCPCB stock).

Selection logic: firmware (SlimeVR-Tracker-nRF) has drivers for BMI270,
ICM-42688, ICM-45686, ISM330BX, LSM6DSM, LSM6DSO, LSM6DSV. The SlimeVR IMU
comparison puts only the LSM6DSV family in the same "superior" tier as the
ICM-45686; ICM-42688 (temperature drift) and BMI270 (poor drift) are
explicitly not recommended, and ISM330BX has 10 units in stock. The only
high-stock ICM-42688P is an HXY-branded clone — rejected: a cloned MEMS die
on the precision part of a tracker is a false economy.

- LGA-14, 2.5 x 3.0 x 0.83 mm, I2C/SPI/I3C, embedded SFLP sensor fusion
- Genuine STMicroelectronics
- JLCPCB assembly stock 2,883 (LCSC retail shows 175 — different pool)
- $3.67 @ 10, $3.10 @ 100; Extended part (~$3 one-time feeder fee)
- **Stock risk: moderate.** 2,883 units is workable but not deep for a
  popular part — order sooner rather than later.
- KiCad: symbol `smol-biscuits:LSM6DSV16X` (new, drawn from datasheet
  DS13510 Rev 4 Figure 5 — pin mapping needs review before layout),
  footprint `smol-biscuits:LGA-14_3x2.5mm_P0.5mm_LayoutBorder3x4y` (copied
  from KiCad's official Package_LGA library, ST's standard LGA-14 land
  pattern used across the LSM6DS family)
- QMC6309 magnetometer stays unchanged

## PMIC — Nordic nPM1300-QEAA — LCSC C7501206 — Global Sourcing

Fixed by the rev 2 architecture (not a selection), but stock-checked:

- QFN-32-EP 5×5 mm; $2.36 @ 10, $1.83 @ 100
- **Stock risk: high.** 168 units at JLC via "Global Sourcing" (longer lead
  time), LCSC retail shows out of stock. Plan the buy early, or source the
  reel elsewhere (Nordic distributors) and consign.
- KiCad: symbol `smol-biscuits:nPM1300` (new, drawn from Product Spec v1.1
  Table 35), footprint `Package_DFN_QFN:QFN-32-1EP_5x5mm_P0.5mm_EP3.45x3.45mm`
  (stock; **verify the EP size** against PS §9.2.1 before layout)

## MCU — Nordic nRF52840-QIAA (bare chip) — LCSC C190794 — Basic

Module survey result: no JLCPCB-stocked nRF52840 module exposes RF for a
custom on-PCB antenna (Ebyte E73-S1C has its ceramic antenna on-module,
E73-S1CX is IPEX-only; Raytac/Minew/Holyiot not stocked). Since rev 2
wants its own printed antenna, the bare aQFN73 chip is the route.

- **JLCPCB Basic**, 1,178 units — the deepest nRF stock available
- Circuit per nRF52840 PS v1.11 §7.3.5 (Config 5): VDD = 1.8 V from
  nPM1300 BUCK1, VDDH shorted to VDD, REG1 DC/DC enabled
  (DCC → 15 nH → 10 µH → DEC4/6), USB enabled, NFC unused, 32.768 kHz
  crystal omitted (firmware uses LFRC)
- RF match per PS v1.1 values: 0.8 pF shunt, 4.7 nH series, 0.5 pF shunt
  → `RF_ANT` net → printed antenna (layout geometry + tuning TBD)
- KiCad: stock `MCU_Nordic:nRF52840` symbol +
  `Package_DFN_QFN:Nordic_AQFN-73-1EP_7x7mm_P0.5mm` footprint
- Consequences owned: aQFN73 fanout (plan 4-layer, still single-sided
  assembly), antenna tuning, and RF certification burden

## Still to select

- Buck inductor L1: 2.2 µH, DCR < 400 mΩ, 2016-metric (Nordic BOM);
  footprint placeholder is `Inductor_SMD:L_Murata_DFE201610P`
- 32 MHz crystal: CL = 8 pF, ±40 ppm; 3225 footprint placed (project has
  it), Nordic BOM suggests 2016 size
- nRF DC/DC and RF inductors (15 nH / 10 µH / 4.7 nH) and the NP0 RF
  caps — 0402, values fixed by Nordic, parts unpicked
- Status LED, 10 kΩ NTC thermistor, and part numbers for the passives
  (Nordic-spec sizes: sub-µF caps and resistors 0402, bulk ≥1 µF 0603,
  RF parts 0402 NP0 — all commodity values)
- Antenna geometry (printed IFA/meander) — layout work, plus final match
  values after tuning

## USB connector — Korean Hroparts TYPE-C-31-M-17 — LCSC C283540

Mid-mount (sink type, board cutout) 16-pin USB 2.0 Type-C, ~$0.09,
chosen for the low-profile wearable stack. KiCad stock footprint
`Connector_USB:USB_C_Receptacle_HRO_TYPE-C-31-M-17` (includes the board
cutout). No CC resistors needed — the nPM1300 handles Type-C detection.
LCSC-sourced; the Digikey-side mid-mount alternative (JAE
DX07S016JA1R1500) is backordered and uses a different footprint.

## BOM integration

The LSM6DSV16X symbol carries an `LCSC` field (`C5267406`), which the
Fabrication Toolkit picks up for the JLCPCB BOM export. Give the switch and
TVS schematic instances the same field when they are placed:
`C720477` (switch), `C5261083` (TVS).
