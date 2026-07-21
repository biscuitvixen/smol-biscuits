# Rev 2 support-part ownership map

Which passive belongs to which IC in `pcb/smol-slime-rev2.kicad_sch`.
This is also the layout placement guide: each group sits tight against
its IC on the board — especially C16/C17/L2/L3 at the nRF's DCC/DEC4
pins, C6 + L1 in a tight buck loop at the nPM1300, and the RF match
parts in line between the ANT ball and the antenna feed.

## U1 — nPM1300 (PMIC)

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
| R1, R2 | 4.7 kΩ | SDA/SCL pull-ups — **the only pair on the bus** |

R1/R2 are the single pull-up pair for the whole I2C bus (nRF master +
nPM1300 + IMU + magnetometer all share `SDA`/`SCL`). I2C wants one pair
per line, not one per device, so the IMU and magnetometer sections below
list no resistors of their own — do not add any. They're filed here only
because the nPM1300's SDA/SCL pins are a convenient anchor; on the board
place them at one end of the bus. Full net map: `docs/rev2-nets.md`.

## U2 — LSM6DSV16X (IMU)

| Part | Value | Role |
|---|---|---|
| C9 | 100 nF | VDD |
| C10 | 100 nF | VDDIO |
| C8 | 1 µF | VDD_SNS bulk (sensor rail — equally at home by the nPM's LSOUT1 pin) |

## U3 — QMC6309 (magnetometer)

| Part | Value | Role |
|---|---|---|
| C11 | 100 nF | VDD |

## U4 — nRF52840 (MCU)

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

## J4 — USB-C

Nothing: no CC resistors (the nPM1300 does Type-C detection), and ESD
protection currently covers only the button line (D1), not the USB data
pins.
