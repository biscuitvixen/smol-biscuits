# Rev 2 plan

Status report for the revision 2 redesign. Branch: `rev2`. Part-level
sourcing detail lives in [rev2-parts.md](rev2-parts.md).

## Direction

Rev 1 is a XIAO nRF52840 module carrier. Rev 2 moves to a custom
nRF52840 + Nordic nPM1300 design: the PMIC brings proper ship mode,
single-button power control, and USB charging into a wearable-sized,
single-sided, SMD-only board assembled by JLCPCB (Basic parts preferred).

## One-button power control

The nPM1300's SHPHLD pin does all the work; the button is the only
external part in the control path.

| Action | Result | Mechanism |
|---|---|---|
| Short press | Wake from ship mode | SHPHLD hardware |
| ~3 s press | Enter ship mode | Firmware, via I2C |
| ~10 s press | Hardware reset | nPM1300 long-press reset |
| USB attach | Exit ship mode | nPM1300 VBUS detect |

Reference: `samples/pmic/native/npm13xx_one_button` in sdk-nrf, and the
nPM1300-EK schematic.

### Button circuit

```
SHPHLD ──┬────── SW (TS-1088-AR02016) ────── GND
         │
         └────── TVS (PESD5V0S1BA) ───────── GND
```

- TVS placed close to the switch, short return path to GND
- Deliberately absent (nPM1300 provides internally): pull-up, debounce
  capacitor, series resistor

## Selected parts

| Part | LCSC | JLC tier | Notes |
|---|---|---|---|
| XUNPU TS-1088-AR02016 switch | C720477 | Basic | 3.9×3.0×2.0 mm, 160 gf |
| PESD5V0S1BA TVS (HXY) | C5261083 | Basic | SOD-323, bidirectional |
| ST LSM6DSV16X IMU | C5267406 | Extended | replaces unsourceable ICM-45686 |
| QMC6309 magnetometer | — | — | carried over from rev 1 |

## Done so far (on `rev2`)

- Custom libraries consolidated into `pcb/libs/smol-biscuits.kicad_sym` +
  `smol-biscuits.pretty` + shared `3dmodels/`; dead lib-table entries and
  vendor cruft removed
- Project converted to KiCad 10, verified (ERC/DRC, sch↔pcb sync, 3D
  models) with no copper changes
- Switch and ST LGA-14 footprints imported from official KiCad libraries;
  LSM6DSV16X symbol drawn from datasheet DS13510 Rev 4 Figure 5
  (pin mapping pending review before layout)

## Schematic draft (`pcb/smol-slime-rev2.kicad_sch`)

New KiCad project alongside rev 1, sharing `libs/`. Follows the nPM1300
Product Spec Configuration 2 (minimal): BUCK1 = 1.8 V system rail (VSET1
47 kΩ), VDDIO on 1.8 V, BUCK2 unused (VOUT2 strapped to VSYS per Nordic —
this is the one expected ERC error), LOADSW1 gates a switchable `VDD_SNS`
1.8 V rail for the IMU + magnetometer so firmware can power-cycle sensors.
Button + TVS on SHPHLD as specified. I2C bus (SDA/SCL, 4.7 kΩ pulls to
+1V8) carries nPM1300, LSM6DSV16X (SA0 low, CS high, aux SPI disabled),
and QMC6309. One status LED on LED0 from VSYS. NTC pin gets an on-board
10 kΩ thermistor. Connectivity is stub+label style — placement is a draft
for rearranging in KiCad. ERC: 3 expected errors (VOUT2/VSYS tie, CC1/CC2
awaiting USB connector), isolated-label warnings for the nets that wait on
the nRF stage.

## Remaining work

1. **nRF52840 stage:** bare chip vs. a certified module is the open
   decision (antenna design, FCC/CE burden, board area); its symbol,
   USB connector, and SWD wiring follow from it
2. **Review the draft:** nPM1300/LSM6DSV16X symbol pin maps against
   datasheets, QFN EP pad size, buck inductor selection, rail voltage
   choice (1.8 V assumed — check every peripheral is happy with it)
3. **Layout:** single-sided, wearable outline; button placement is
   enclosure-facing
4. **Firmware:** board definition + `LSM6DSV` sensor selection in
   SlimeVR-Tracker-nRF; ship-mode long-press handling per the
   one-button sample
5. **Ordering:** LSM6DSV16X stock moderate (2,883), nPM1300 thin (168,
   Global Sourcing) — order both early
