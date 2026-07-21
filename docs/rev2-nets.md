# Rev 2 design reference — nets, power tree, and placement

Reference for hand-wiring `pcb/smol-slime-rev2.kicad_sch` in KiCad: the power
tree, the full net map, which support part belongs to which IC, and how to
organize the sheet. Part selections and values are in
[rev2-parts.md](rev2-parts.md); remaining work is in
[rev2-todo.md](rev2-todo.md).

The net list here was pulled from `kicad-cli sch export netlist`; regenerate it
that way after edits to re-check. 90 nets total, but 54 are single-pin
`unconnected-(...)` spares, so the design is really 35 nets.

Most of the unfamiliar "voltage" nets are the nRF52840's internal
power-management pins. On the rev 1 XIAO module these were sealed inside the
module. Going bare exposes every one — you don't design with them, you just
hang Nordic's exact reference part on each, to GND, per PS v1.11 §7.3.5
(Circuit configuration no. 5).

## Power tree — the mental model

Everything flows through the nPM1300:

```
USB VBUS_IN ─┐
             ├─► nPM VSYS ──► BUCK1 ──► +1V8 ──► nPM LDO1 ──► VDD_SNS
battery VBAT ─┘   (5.5V)      (buck)   (main)   (load sw)    (sensor rail)
```

The one thing to internalise: `+1V8` and `VDD_SNS` are **two different rails**.
The nRF runs off `+1V8` directly. The sensors run off `VDD_SNS`, which is
`+1V8` passed through the nPM's LDO1 / load switch so firmware can cut sensor
power independently (e.g. in ship mode).

| Net | What it is | Key nodes |
|---|---|---|
| `VBUS_IN` | 5 V from USB-C into the PMIC | J4 → U1.21, C1 |
| `VBAT` | LiPo cell | J1 → U1.19, C5 |
| `VSYS` | PMIC system rail (higher of USB/batt) | U1.4/20/32, C2/C3, D2 anode |
| `+1V8` | **Main rail — BUCK1 output.** nRF, nPM VDDIO, LDO1 input, I2C pull-ups | U1.1/12/28, all U4 VDD balls, R1/R2, C6/C7/C21–C25 |
| `VDD_SNS` | **Switchable sensor rail** off nPM LDO1 (U1.29) | U2 (IMU), U3 (mag), C8/C9/C10/C11 |
| `VBUSOUT` | PMIC protected 5 V pass-through to the nRF USB pins | U1.22 → U4.AD2, C4/C20 |
| `VSET1` | Resistor-programmed BUCK1 voltage (47 kΩ = 1.8 V) | R3 → U1.17 |
| `NTC` | Battery thermistor sense | TH1 → U1.18 |

## nRF52840 internal decoupling nets — "just follow the reference"

Each of these gets one prescribed part to GND (or the inductor chain). None is
a signal you route anywhere. This is the price of going bare-chip.

| Net | Part | Pin function |
|---|---|---|
| `DEC1` | C14 100 nF | Core LDO decoupling |
| `DEC3` | C15 100 pF | Internal regulator decoupling |
| `DEC5` | C18 820 pF | Internal regulator decoupling |
| `DECUSB` | C19 4.7 µF | USB PHY 3.3 V regulator |
| `DCC` → `DCC_F` → `DEC4_6` | L2 15 nH → L3 10 µH → C16 1 µF + C17 47 nF | REG1 DC/DC LC filter. `DCC` is the switch node, `DCC_F` is just the midpoint between the two inductors, `DEC4`/`DEC6` are the filtered output |

`DEC2` (U4.A18) and `DCCH` (U4.AB2) are correctly left **unconnected** in this
configuration (VDDH shorted to VDD, high-voltage DC/DC unused).

## Clock and RF

| Net | What |
|---|---|
| `XC1` / `XC2` | 32 MHz crystal X1 + 12 pF load caps C12/C13 |
| `BUCK1_SW` | nPM buck switch node → L1. High di/dt — keep the loop tight in layout |
| `ANT` → `RF_ANT` | RF match chain C26/L4/C27/C28 out to the printed antenna |

## Signals

- `SDA` / `SCL` — shared I2C bus (see pull-up note below)
- `IMU_INT1` / `IMU_INT2` — IMU interrupts to the nRF (U2.4→U4.L1, U2.9→U4.N1)
- `NPM_GPIO1_INT` — PMIC interrupt to the nRF (U1.8 → U4.M2)
- `NPM_GPIO0_RESET` — PMIC reset, tied to the nRF and reset pad J3 (U1.7, U4.AC13, J3.2)
- `LED0` — status LED, sunk by the PMIC (D2.1 → U1.25)
- `SHPHLD` — power-button line, switch SW1 + TVS D1 (→ U1.15)
- `USB_DP` / `USB_DM` — USB data to the nRF; `USB_CC1` / `USB_CC2` — Type-C detect, to the PMIC
- `SWDIO` / `SWDCLK` — debug, to header J2

## I2C bus and pull-ups

`SDA`/`SCL` are a **single shared bus**: nRF master (U4) + nPM1300 (U1) + IMU
(U2) + magnetometer (U3), all on the same two nets.

- `SCL`: R2, U1.14, U2.13, U3.A2, U4.H2
- `SDA`: R1, U1.13, U2.14, U3.B2, U4.G1

R1/R2 (4.7 kΩ) are the **only** pull-up pair on the bus. I2C wants one pair per
line, not one per device — so the IMU and magnetometer have no pull-ups of
their own, and you should not add any. In the ownership map below R1/R2 are
filed under U1 only because the nPM's SDA/SCL pins are a convenient anchor; on
the board place them at one end of the bus (the nRF master or the mag),
wherever reads as the top of the bus.

## GND and the unconnected spares

`GND` is the big net — 56 pins including the nRF EP, the nPM PVSS/EP pads, and
every cap's return. The remaining 54 single-pin nets are all
`unconnected-(...)`: spare nRF GPIOs (P0.13–P0.25, P1.xx), NFC and analog-in
pins, unused nPM GPIOs, the IMU's aux-SPI pins (SDO_AUX/OCS_AUX), and USB SBU.
These are deliberate — pins you are not using. Flag them no-connect so ERC
stays quiet.

## Schematic organization — five blocks

Labels are not a sin. On a board this dense, wiring every rail and bus makes
worse spaghetti than clean labels. The split:

- **Draw as local wires** (topology is the point): the button, the buck loop,
  the RF match (all already drawn), plus the crystal loop (X1+C12/C13), the USB
  D±/CC/shield from J4, and the SWD header.
- **Keep as labels** (wiring them is noise): global rails `GND`, `VSYS`,
  `+1V8`, `VBAT`, `VBUSOUT`, `VBUS_IN`, `VDD_SNS`, and the `SDA`/`SCL` bus.
  Decoupling caps stay as a short stub from a rail label to a GND stub, planted
  right at the pin they serve.

Frame each block with a graphic box + title so the intent reads at a glance:

| Block | Anchor | Contents |
|---|---|---|
| 1. Power / PMIC | U1 nPM1300 | VBUS in (C1) → VSYS (C2/C3) → BUCK1 loop (SW1·L1·C6·R3) → +1V8; VBAT/J1/TH1 battery corner; VBUSOUT (C4); VDDIO (C7); LED D2; SHPHLD button (D1+SW1) |
| 2. MCU core | U4 nRF52840 | DEC/DECUSB decoupling column (C14–C25), DCC filter L2→L3, crystal X1+C12/C13 |
| 3. RF | off U4 ANT | C26·L4·C27·C28 → RF_ANT (own box — tuning-sensitive) |
| 4. Sensors (I2C) | U2 + U3 | IMU U2 (+C8/C9/C10), mag U3 (+C11), R1/R2 pull-up pair, INT lines to the nRF |
| 5. USB / debug | J4 + J2/J3 | J4 Type-C (D± to nRF, CC1/CC2 to nPM, shield to GND), SWD J2, VTref/reset J3 |

Battery corner (block 1): keep J1, C5, TH1 together so the "plugs in from
outside" parts sit in one spot.

## Support-part ownership

The full per-designator version of the block table above: which passive sits
against which IC. This is the placement guide — each group hugs its IC,
especially C16/C17/L2/L3 at the nRF's DCC/DEC4 pins, C6 + L1 in a tight buck
loop at the nPM1300, and the RF match parts in line between the ANT ball and
the antenna feed. Values are the authority in [rev2-parts.md](rev2-parts.md).

**U1 — nPM1300 (PMIC)**

| Part | Value | Pin it serves |
|---|---|---|
| C1 | 1 µF | VBUS (USB input) |
| C2, C3 | 10 µF | VSYS |
| C4 | 1 µF | VBUSOUT |
| C5 | 2.2 µF | VBAT |
| C6 | 10 µF | VOUT1 (+1V8 buck output) |
| C7 | 100 nF | VDDIO |
| L1 | 2.2 µH | SW1 pin (buck inductor) |
| D1 (TVS) + SW1 (button) | — | SHPHLD |
| TH1 | 10 kΩ NTC | NTC |
| D2 | status LED | LED0 (sinks from VSYS) |
| J1 | battery connector | VBAT |
| R3 | 47 kΩ | VSET1 (sets 1.8 V) |
| R1, R2 | 4.7 kΩ | SDA/SCL pull-ups — **the only pair on the bus** (see I2C note above) |

**U2 — LSM6DSV16X (IMU)**

| Part | Value | Role |
|---|---|---|
| C9 | 100 nF | VDD |
| C10 | 100 nF | VDDIO |
| C8 | 1 µF | VDD_SNS bulk (sensor rail — equally at home by the nPM's LSOUT1 pin) |

**U3 — QMC6309 (magnetometer)**

| Part | Value | Role |
|---|---|---|
| C11 | 2.2 µF | VDD reservoir (QMC6309 §4.3.3 / Fig. 6, low-ESR ceramic) |

**U4 — nRF52840 (MCU)**

| Part | Value | Pin it serves |
|---|---|---|
| X1 + C12, C13 | 32 MHz + 12 pF ×2 | XC1/XC2 |
| C14 | 100 nF | DEC1 |
| C15 | 100 pF | DEC3 |
| C16 + C17 | 1 µF + 47 nF | DEC4/DEC6 (tied) |
| C18 | 820 pF | DEC5 |
| C19 | 4.7 µF | DECUSB |
| C20 | 4.7 µF | VBUS (on VBUSOUT net) |
| C21, C22, C23–C25 | 4.7 µF, 1 µF, 100 nF ×3 | VDD/VDDH (+1V8, one per VDD pin) |
| L2 → L3 | 15 nH → 10 µH | DCC (REG1 DC/DC filter, chip side first) |
| C26, L4, C27, C28 | 0.8 pF, 4.7 nH, 0.5 pF, DNP | ANT → antenna feed (RF match) |
| J2, J3 | SWD pads, VTref/reset pads | debug |

**J4 — USB-C:** no support parts. No CC resistors (the nPM1300 does Type-C
detection), and ESD protection currently covers only the button line (D1),
not the USB data pins.

## Working notes

- The schematic is now **hand-edited in KiCad**. The generator that produced it
  (`gen_rev2.py`, session scratchpad only) is retired — re-running it would
  overwrite hand edits.
- Before layout, give SW1/D1/U2 their `LCSC` fields (`C720477`, `C5261083`,
  `C5267406`) so the BOM export is complete.
- Expected ERC after wiring: the four documented straps (VOUT2→VSYS,
  VBUSOUT power flag, IMU SDO/SDX ground straps). Nothing new.
