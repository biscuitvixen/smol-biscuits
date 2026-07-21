# Rev 2 — what's on your plate

The schematic (`pcb/smol-slime-rev2.kicad_sch`) is electrically complete
and its passive values are datasheet-verified. Everything below is the work
that remains, in roughly the order it needs doing. Part selections and
values are in [rev2-parts.md](rev2-parts.md); the net map and placement
guide are in [rev2-nets.md](rev2-nets.md). Datasheets for every rev 2 part
are PDFs in this folder.

## 1. The antenna (entirely yours)

The schematic deliberately stops at a net called `RF_ANT` — there is no
antenna component anywhere. The chain is:

```
nRF ANT pin ── C26 0.8pF ┬─ L4 4.7nH ── C27 0.5pF ┬─ C28 (DNP) ┬─ RF_ANT ── your copper
              (shunt)    │  (series)    (shunt)   │  (shunt)   │
                        GND                      GND          GND
```

- C26/L4/C27 are Nordic's reference match for a 50 Ω antenna
  (nRF52840.pdf, PS v1.11 §7.3.5). Treat them as starting values — the
  real values come out of tuning against *your* geometry.
- C28 is an unpopulated 0402 pad at the feed for that tuning, mirroring
  the N.C. pad Nordic keeps in their reference.
- Design references: Nordic's reference layouts (nRF52840 product page),
  and the classic printed 2.4 GHz antenna app notes (TI DN035 swivel/IFA,
  Cypress AN91445) for meander/IFA geometry.
- Layout rules that make or break it: antenna area keeps copper and
  pour clearance on **all** layers, solid ground under the feed/match
  region up to the antenna edge, nothing routed under the RF path.
- Tuning: iterate the three match values (plus C28) with a VNA if you
  can get one, otherwise by RSSI/range testing. Expect at least one
  board spin for the antenna.

## 2. Review my hand-derived work before layout

- nPM1300 symbol pin map vs nPM1300.pdf (PS v1.1) Table 35 (p. 151)
- LSM6DSV16X symbol pin map vs LSM6DSV16X.pdf Figure 5 (p. 9)
- nPM1300 QFN exposed pad: stock EP3.45×3.45 footprint placed — check
  against PS §9.2.1 mechanical drawing
- TS-1088 and LGA-14 footprints vs their datasheet land patterns
- nRF GPIO assignments (I2C on P0.26/P0.27, IMU INTs on P0.06/P0.08,
  nPM interrupt on P0.07) — remux freely, nothing is timing-critical
- The 1.8 V single-rail assumption (BUCK1, VSET1 = 47 kΩ): nRF52840,
  LSM6DSV16X, QMC6309 are all fine at 1.8 V per datasheets — confirm
  you're happy before layout locks it in
- Expected ERC findings (enumerated in the sheet note): VOUT2→VSYS
  Nordic strap, VBUSOUT power flag, four USB nets awaiting the
  connector, IMU SDO/SDX ground straps

## 3. Shopping list (Digikey/Mouser-friendly)

Since JLCPCB assembly is optional now, the Basic/Extended games stop
mattering and genuine parts are back on the table:

| Part | Spec | Notes |
|---|---|---|
| U1 nPM1300-QEAA-R7 | QFN32 5×5 | Scarce at LCSC; Nordic distributors (Digikey/Mouser) carry it — buy early |
| U4 nRF52840-QIAA | aQFN73 | Plentiful everywhere |
| U2 LSM6DSV16XTR | LGA-14 | Genuine ST from anywhere |
| U3 QMC6309 | carried over | LCSC-territory part |
| D1 TVS | PESD5V0S1BA,115 (Nexperia) | Genuine, plentiful at Digikey/Mouser; the HXY clone was only for JLC Basic |
| SW1 switch | SPST-NO, ~3.9×3×2 mm, 160–260 gf | XUNPU TS-1088 is LCSC-only. Western equivalent (e.g. C&K KMR2 class) means swapping the footprint — decide before layout |
| X1 crystal | 32 MHz, **CL = 8 pF**, ≤±40 ppm total | 3225 footprint placed; Nordic suggests 2016 — either works, footprint follows the part |
| L1 | 2.2 µH, DCR < 400 mΩ, 2016 size | Murata DFE201612E class (nPM1300 buck) |
| L3 | 10 µH, I_DC ≥ 50 mA, 0603 | nRF REG1 filter |
| L2 / L4 | 15 nH ±10% / 4.7 nH ±5%, 0402 HF chip inductor | Murata LQG15HS class; L4 is in the RF path — no substituting wirewound for multilayer casually |
| Caps & resistors | values + ratings in [rev2-parts.md](rev2-parts.md) | all commodity: sub-µF 0402, bulk ≥1 µF 0603, NP0/C0G for the RF match and crystal loads, 1 % for the 47 kΩ VSET1 |
| TH1 | 10 kΩ NTC, B≈3380 (0603) | Confirm beta against nPM1300 NTC config you'll use |
| LED, J1 JST PH 2-pin, USB connector | — | Your pick |

Generate the full BOM from the schematic when ready (`kicad-cli sch
export bom` or the Fabrication Toolkit if you do end up at JLC).

## 4. USB connector — layout note

The connector (mid-mount HRO TYPE-C-31-M-17) is placed and wired; part
detail is in [rev2-parts.md](rev2-parts.md). Layout consequence: mid-mount
means the footprint needs its **board edge cutout** — the stock footprint
carries the outline and the connector hangs in it, roughly halving the
height above board.

## 5. Layout notes

- aQFN73 fanout realistically wants 4 layers; assembly stays single-sided
- nPM1300: PVSS1/PVSS2 via straight to ground per PS Fig 54, tight buck
  loops (L1 + 10 µF right at SW/VOUT), exposed pads need via stitching
- nRF: keep the DEC4/DEC6 parts (1 µF + 47 nF) and the 15 nH at the
  pins; Nordic publishes reference layouts worth copying for the fanout
- Button and LED placement are enclosure-facing decisions

## 6. Hand-assembly heads-up

If you solder it yourself: the aQFN73 (0.5 mm pitch, thermal pad) and
both QFN/LGA parts are stencil + paste + hot air or hotplate territory,
not iron work. Order a stencil with the boards. The 0402 NP0 RF parts
are fiddly but fine with tweezers. Leave C28 empty until tuning.

## 7. Firmware (in your SlimeVR-Tracker-nRF fork)

- Board definition for the new hardware
- Sensor: `LSM6DSV` driver (+ QMC6309 mag as before)
- Low-frequency clock: LFRC — there is no 32.768 kHz crystal
- NFC pins as GPIO (`CONFIG_NFCT_PINS_AS_GPIOS`) if you ever use them
- Ship mode / one-button behaviour per `samples/pmic/native/npm13xx_one_button`
- nPM1300 fuel gauge integration comes free with the nRF Connect SDK

## 8. One honest note

Own antenna = own radio. The pre-certified module shortcut is gone, so
range/performance is yours to validate, and if this ever becomes more
than a personal project, FCC/CE intentional-radiator testing is on this
design, not a module grant. For personal DIY use this is the normal
SlimeVR-community situation.
