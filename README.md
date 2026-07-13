# 🍪 Smol Biscuits SlimeVR Trackers

![Build Status](https://img.shields.io/badge/build-passing-brightgreen)
![License](https://img.shields.io/badge/Hardware-CC--BY--NC--SA--4.0-blue)
![License](https://img.shields.io/badge/Firmware-SlimeVR--Terms-green)
![Platform](https://img.shields.io/badge/platform-nRF52840-orange)

My own SlimeVR-compatible trackers, built because I wanted something
smaller. Like butterflies but before they were out! Custom PCB on a
Seeed Studio XIAO nRF52840, custom firmware fork, and a two-piece 3D
printed case with a flexible link between the battery and PCB sections.
They fit great and I love them!

## Table of Contents

- [Hardware](#hardware)
- [3D Printed Case](#3d-printed-case)
- [Firmware](#firmware)
- [Assembly](#assembly)
- [Setup & Calibration](#setup--calibration)

## Hardware

### PCB

Designed in KiCad, files in [`pcb/`](./pcb). A minimal carrier board for
the XIAO nRF52840, with sensors mounted on the opposite side.

### Sensors

| Component | Model | Function |
|-----------|-------|----------|
| IMU | ICM-45686 | 6-axis motion sensing |
| Magnetometer | QMC-6309 | Magnetic field detection |

### Battery

The board takes a LiPo. The included case is sized around a 150mAh cell -
go bigger if your case design has room for it.

## 3D Printed Case

Two-piece design: separate battery and PCB compartments connected by a
flexible TPU link. The joint lets the tracker bend naturally against your
body, which makes a real difference for all-day wear.

Print the flex link in TPU. Base plates should be PETG - they take real
strain from the strap and PLA will eventually fail on you. The top cover
and switch piece can be PLA or PETG, whichever you have to hand.

## Firmware

Firmware lives in [`firmware/`](./firmware) as a git submodule - a fork of
the SlimeVR firmware with specific support for this PCB.

Pre-compiled firmware is available in the [releases section](../../releases).

### Building from source

1. Clone with submodules:
   ```bash
   git clone --recurse-submodules https://github.com/biscuitvixen/smol-biscuits.git
   ```

2. Follow the [SlimeVR compilation guide](https://docs.slimevr.dev/smol-slimes/firmware/smol-compiling-firmware.html),
   then build with:
   ```bash
   west build -p auto -b xiao_ble -d build . -- -DNCS_TOOLCHAIN_VERSION=NONE -DBOARD_ROOT=.
   ```

## Assembly

### Soldering

The components on this board are small. Take your time.

1. Solder the sensor components onto the PCB. Leave the XIAO for now.
2. Pre-tin the battery terminals on the XIAO and the matching footprint
   on the PCB.
3. Use the solder jig to position the XIAO with battery terminals facing up.
4. Apply flux, place the sensor PCB on top aligning the battery terminals,
   and heat through the square pass-through pads while pressing the boards
   flat. Let it cool fully before moving it.
5. Inspect all joints and check for shorts with a multimeter before
   continuing.
6. Solder the castellated pins around the PCB edge, then solder the battery
   to the back of the sensor board.

### Programming

**Tracker:**

1. Connect via USB-C and double-tap the reset button. The device should
   appear as `XIAO-SENSE`.
2. Copy `update-slimenrf_xiao_sense_bootloader-0.9.2-SlimeVR.7_nosd.uf2`
   to the drive. It will remount as `SLIMENRF_XIAO_SENSE`.
3. Copy `firmware_tracker.uf2` to the drive. The tracker reboots
   automatically.

**Receiver:**

The Holyiot receiver is programmed via Nordic nRF Connect Programmer - flash
`firmware_receiver.hex`. For other hardware, follow the
[SlimeVR receiver docs](https://docs.slimevr.dev).

### Case assembly

1. Slide both bases onto a 30mm strap with the notches facing each other.
2. Place the flex link between the bases in its slot.
3. Insert the battery spacer, then lay the battery on top.
4. Hold the battery in place by its wires and snap the battery base down.
5. Place the PCB into the PCB base, aligning the switch with the cutout.
6. Slide the switch slider over the PCB switch.
7. Snap the top cover into place and adjust the strap to fit.

## Setup & Calibration

Follow the [SlimeVR pairing and calibration guide](https://docs.slimevr.dev/smol-slimes/firmware/smol-pairing-and-calibration.html).

## License

### Hardware

PCB files, 3D models, schematics, and case designs are licensed under
[CC BY-NC-SA 4.0](LICENSE-CC-BY-NC-SA). Personal and educational use is
free - just credit me and share improvements under the same terms.
Commercial use requires a separate arrangement; get in touch at
onlybiscuit7@gmail.com.

### Firmware

Firmware retains the original SlimeVR license terms:
[MIT](LICENSE-MIT) | [Apache 2.0](LICENSE-APACHE). See the firmware
submodule for the full details.

## Contributing

Pull requests and issues welcome.

## Support

- [SlimeVR Documentation](https://docs.slimevr.dev)
- SlimeVR Discord
- GitHub Issues