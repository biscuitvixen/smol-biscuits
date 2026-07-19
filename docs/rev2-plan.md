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
10 kΩ thermistor.

The nRF52840 stage is wired per its PS v1.11 §7.3.5 Config 5 (decision:
bare chip — no stocked module exposes RF for a custom antenna): bare
nRF52840-QIAA, VDDH shorted to the 1.8 V rail, REG1 DC/DC with the
15 nH + 10 µH chain, full DEC decoupling, 32 MHz crystal (12 pF loads),
USB D+/D− to labels, RF match (0.8 pF/4.7 nH/0.5 pF) into the `RF_ANT`
net for the printed antenna, SWD + VTref/reset programming pads on the
existing PinPads footprints. I2C on P0.26/P0.27; IMU interrupts on
P0.06/P0.08; nPM1300 GPIO0→RESET, GPIO1→P0.07.

Connectivity is stub+label style — placement is a draft for rearranging
in KiCad. ERC findings are all expected and enumerated in the sheet note
(VOUT2/VSYS tie, VBUSOUT flag, USB nets awaiting a connector, IMU
GND straps).

## Remaining work

1. **Review the draft:** nPM1300/LSM6DSV16X symbol pin maps against
   datasheets, nRF GPIO pin choices, QFN EP pad size, rail voltage
   choice (1.8 V assumed — check every peripheral is happy with it)
2. **USB connector selection** and wiring (last missing schematic piece)
3. **Layout:** wearable outline, single-sided assembly, likely 4-layer
   for the aQFN73 fanout; printed antenna geometry + matching network
   tuning; button placement is enclosure-facing
4. **Firmware:** board definition + `LSM6DSV` sensor selection in
   SlimeVR-Tracker-nRF; ship-mode long-press handling per the
   one-button sample; LFRC (no 32.768 kHz crystal); NFC pins as GPIO
5. **Ordering:** LSM6DSV16X stock moderate (2,883), nPM1300 thin (168,
   Global Sourcing) — order both early; nRF52840-QIAA is Basic with
   deep stock
