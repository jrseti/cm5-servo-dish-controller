# Hardware

Hardware design files for the CM5 Servo Dish Controller.

## Layout

```
hardware/
├── kicad/   # KiCad project: schematic (PCB layout not started)
└── stm32/   # STM32 firmware / CubeMX files (empty so far)
```

## Status

- **Schematic:** first pass complete. KiCad's electrical rules check reports 0 violations.
- **PCB:** not started.
- **Not yet done:** the 48 V to 5 V converter has not been simulated or built, and
  nothing here has been reviewed by a second person. See [Open items](#open-items).

Schematic PDF: [kicad/cm5-servo-dish-controller.pdf](kicad/cm5-servo-dish-controller.pdf)

## Board overview

A carrier board for a Raspberry Pi Compute Module 5 (CM5) with an STM32G474RET6 that
drives the dish's servo motors over CANopen. It is derived from the
[roboclaw-cm4-dc-dish-controller](https://github.com/jrseti/roboclaw-cm4-dc-dish-controller)
board. The CM5, Ethernet, GPS, I2C and header circuits are carried over. These parts changed:

| Area | RoboClaw CM4 board | This board |
|---|---|---|
| Motor control | RoboClaw 2x15A, brushed DC motors | Removed. An STM32G474RET6 is the CANopen master for external servo drives (e.g. DSY-400C) |
| Motor power | On the board (PGND, 15 A fuse) | None. The drives are wired straight to the 48 V supply |
| Encoders, limit switches, E-stop | On the board | Removed. They will be separate CANopen nodes (future boards) |
| Control loop | In the RoboClaw | In the STM32: it commands drive velocity and reads position over CAN |
| Module to controller link | UART, CM4 to RoboClaw | UART, CM5 to STM32 (same CM pins, GPIO12/13) |
| Supply | 12.8 V battery | 48 V DC, working range 34–58 V |
| 5 V rail | eFuse + AP64500 buck | LM5116 controller + two 100 V MOSFETs, 5.09 V at 5 A |
| Module | CM4 | CM5 |
| USB | USB 2.0 hub, 2 × USB-A | The CM5's two USB 3.0 ports on a stacked USB 3.0 Type-A connector |
| USB-C | rpiboot through a mux | rpiboot, wired straight to the CM5 USB 2.0 port |

### Schematic sheets

| Sheet | File | Contents |
|---|---|---|
| Top | `cm5-servo-dish-controller.kicad_sch` | Block diagram, grounding, mounting holes |
| power-input | `power-input.kicad_sch` | 48 V terminal, fuse, reverse-polarity diode, TVS, bulk capacitor, filter |
| power_5v | `power_5v.kicad_sch` | 48 V to 5 V buck, on/off switch, design notes |
| STM32 | `stm32.kicad_sch` | STM32G474RET6, its 3.3 V regulator, crystal, SWD, UART buffers, reset/boot control |
| STM32 / CAN | `can.kicad_sch` | Isolated CAN transceiver and supply, choke, termination, RJ45 |
| CM5 | `cm5.kicad_sch` | CM5 module |
| CM5 / usb | `usb.kicad_sch` | USB 3.0 ports, port power switch, ESD |
| CM5 / usb / usb2 | `usb2.kicad_sch` | USB-C rpiboot port |
| CM5 / ethernet | `ethernet.kicad_sch` | Gigabit Ethernet (unchanged) |
| CM5 / CM5_Connectors | `cm5_connectors.kicad_sch` | GPIO header, boot jumpers, console UART |
| CM5 / I2C | `i2c.kicad_sch` | Temperature sensor, I2C connector (unchanged) |
| GPS | `gps.kicad_sch` | GPS receiver and 1PPS (unchanged) |

Each sheet carries notes explaining its part values.

### Power

- **Input:** J1 screw terminal, F1 (5×20 mm fuse, T2A), D30 series Schottky for reverse
  polarity, D31 SMCJ58A surge clamp, C90 100 µF bulk, FB3 bead. TP68 is a bare
  pad on `V_IN_F` between F1 and D30, for checking the fuse.
- **5 V:** LM5116 synchronous buck at 259 kHz with a 10 µH inductor. Current limit
  7.3 A peak, input UVLO at 33 V. SW1 shuts the converter down (5 V rail only; the
  48 V side stays live). Test points TP55/TP63 (CS) and TP64/TP65 (GND) are for a
  four-wire measurement of the 15 mΩ sense resistor R112: place all four at its pads.
- **3.3 V:** the CM5 supplies `3V3_CM5` as before. The STM32 has its own regulator
  (`3V3_MCU`), so it keeps running while the CM5 is off or rebooting.

### STM32 and CAN

- **CAN:** one CANopen bus, galvanically isolated (CA-IS3052G + B0505S isolated supply).
  Classic CAN up to 1 Mbit/s; the transceiver does not support CAN FD, which CANopen
  does not use.
  The RJ45 uses the CiA 303-1 pinout, the same as the servo drives' CN1/CN2 ports:
  pin 1 CAN_H, pin 2 CAN_L, pin 3 CAN_GND. SW2 switches in a 120 Ω split termination.
- **UART to the CM5:** USART1 through two buffers powered from `3V3_CM5`, so the STM32
  never drives a pin of an unpowered CM5.
- **Firmware update from the CM5:** CM5 GPIO23 resets the STM32 and GPIO24 selects its
  system bootloader, which listens on the same USART1. No SWD probe is needed.
- **Debug:** STDC14 connector (STLINK-V3) with SWD, SWO and a virtual COM port.

STM32 pin use:

| Pin | Function | Connects to |
|---|---|---|
| PA9 / PA10 | USART1 TX / RX | CM5 GPIO13 / GPIO12 |
| PA12 / PA11 | FDCAN1 TX / RX | CA-IS3052G |
| PA13 / PA14 / PB3 | SWDIO / SWCLK / SWO | STDC14 |
| PA2 / PA3 | USART2 (or LPUART1) TX / RX | STDC14 virtual COM port |
| PC6 / PC7 | CANopen RUN (green) / ERR (red) LED | LED5 / LED6 |
| PF0 / PF1 | 24 MHz crystal | X2 |
| PG10 | NRST | CM5 GPIO23 (through Q7), STDC14 |
| PB8 | BOOT0 | CM5 GPIO24 |

All other pins are unconnected.

CM5 GPIO changes from the CM4 board: GPIO23 `MCU_RST`, GPIO24 `MCU_BOOT0`, GPIO25
`USB_nFAULT`. GPIO24/25 were the eFuse fault and power-good inputs.

## Open items

Check these before layout or ordering:

1. **Converter review.** The 48 V to 5 V stage follows the LM5116 datasheet procedure
   but has not been simulated or bench-tested. The reference board's power stages
   were simulated; this one should be too.
2. **5 V tolerance.** Output is 5.09 V nominal, 4.93–5.25 V worst case with the 1 %
   feedback resistors. The CM5 limit is 5.25 V. Fit 0.1 % resistors for margin.
3. **Supply over-voltage.** D31 starts to conduct at about 64 V. If the servo drives
   can pump the 48 V bus above that when braking, D31 overheats and F1 blows. Fit
   brake resistors on the drives, or choose a higher-voltage TVS and re-check the
   100 V ratings.
4. **Fuse.** F1 is the fuse holder; the assembler does not fit the fuse. Fit a
   Littelfuse 0477002.MXP (2 A time-lag, ceramic, 400 VDC, LCSC C187634) by hand.
   It is named in F1's `Fuse` field. A substitute must be ceramic with a DC rating
   of 60 V or more: ordinary glass fuses are rated for AC only.
5. **CM5 module connectors.** J17/J18 are still the Hirose DF40C-100DS-0.4V(51)
   parts from the CM4 board. The CM5 datasheet names Amphenol 10164227-1001A1RLF
   as the mating part, tested to the CM5's higher current.
6. **Self power-cycle circuit** (`nPWR_CYCLE`, U10) is carried over from the CM4
   board and is not proven on a CM5.
7. **Console UART notes** on the connectors sheet (how to enable the Linux console
   on GPIO14/15 on a CM5) need checking against current Raspberry Pi documentation.
8. **Part stock.** All fitted parts were checked against live JLCPCB stock on
   2026-11-17. The three that were out of stock were replaced: FB3 (now
   BLM31PG121SN1L), U25 (now CA-IS3052G, same pinout and footprint as the ISO1042)
   and USB1 (now Molex 48406-0003). Still low: U18 TPS70933DBVR (10), U20
   LTC4311ISC6 (15), U7 LM75BIMX-3 (54), U24 STM32G474RET6 (72), J19
   FTSH-107-01-L-DV-K-A (173). Recheck stock before ordering.
   U23 LM5116MHX/NOPB is listed by JLCPCB as Standard-assembly only, which moves
   the whole order from Economic to Standard assembly.
   The USB1 footprint comes from EasyEDA; check it against the Molex drawing
   before layout.
9. **Net classes** in the project file were updated for the new net names, but the
   track widths and clearances are still the 12 V board's. Review them for 48 V.
   The Ethernet and USB pairs also carry net class flags on the CM5 sheet
   (`ETH_100_OHM`, `USB_90_OHM`), so the target impedance shows on the drawing.

## Libraries

- **Symbols and footprints:** the `MyParts` library in the `parts/` submodule
  ([jrseti/parts](https://github.com/jrseti/parts)), referenced from the project
  library tables as `${KIPRJMOD}/../../parts/...`.
- **IPC-named footprints:** every part on the board uses a footprint from
  `kicad/myparts-ipc.pretty` (`MyPartsIPC`), except the bare test pads
  (`MyParts:TestPoint_Pad_D0.8mm`) and the mounting holes. The schematic's
  Footprint fields point there; the `MyParts` library symbols keep their original
  footprints, so other projects are unaffected.
  - Standard packages have IPC-7351B names. Most come from the reference project.
    The ones added here are named from datasheet maximum heights:
    `SOP65P640X120-21N` (LM5116), `QFP50P1200X1200X160-64N` (STM32),
    `SON127P615X515X120-9N` (BSC098N10NS5), `SOIC127P1150X280-8N` (CA-IS3052G),
    `SOT95P240X110-3N` (PESD2CANFD27V), `CAPAE1030X1250N` (C90),
    `INDM135125X650N` (L3), `RESC2012X60N`, `OSCCC320X250X80-4N` (X2) and
    `SOP127P835X250-4N` (SW2).
  - Parts with no IPC-7351 package family (connectors, headers, modules, switches,
    the CAN choke, the battery, the RH-5016 test points) are named manufacturer_part,
    e.g. `Molex_48406-0003`, `HanRun_HR911130A`, `PinHeader_2x12_P2.54mm_Vertical`.
  - D30/D31 now use the reference project's SMC pattern `DIOM7959X262N`
    (2.7 × 3.3 mm pads instead of 3.03 × 3.82 mm).
  - `Samtec_FTSH-107-01-L-DV-K-A` (J19) numbers its pads `01`–`14` to match the
    STDC14 symbol. The `MyParts` footprint numbers them `1`–`14`, which left
    J19 pins 1–9 without pads.
- **3D models:** set the KiCad path variable `MYPARTS` to `<clone>/parts` and
  `MYPARTS_DATASHEETS` to `<clone>/parts/datasheets`
  (**Preferences → Configure Paths**).

This design added these symbols to `parts/myparts.kicad_sym`, with footprints and
3D models where the library had none:

| Symbol | Part | LCSC |
|---|---|---|
| `ComputeModule5-CM5` | CM5 (the CM4 symbol with CM5 pin names) | – |
| `LM5116MHX` | 100 V buck controller | C13755 |
| `BSC098N10NS5` | 100 V MOSFET | C534398 |
| `TMPC1265HP-100MG-D` | 10 µH inductor | C305223 |
| `RLP25FEGR015` | 15 mΩ sense resistor | C459682 |
| `CL31Y225KCHVPNE` | 2.2 µF 100 V capacitor | C2803390 |
| `EFVH100ADA101M10C0` | 100 µF 100 V electrolytic | C22387799 |
| `0402CG331J500NT` | 330 pF C0G capacitor | C190579 |
| `SS510C` | 100 V Schottky | C123944 |
| `SMCJ58A` | TVS | C151899 |
| `BAV21WS_C2903901` | 250 V switching diode | C2903901 |
| `XC-7_5x20_FuseHolder` | 5×20 mm fuse holder | C3131 |
| `ACT45B-510-2P-TL003` | CAN common-mode choke | C76584 |
| `484060003` | Molex stacked USB 3.0 Type-A connector | C565298 |
| `CA-IS3052G` | Isolated CAN transceiver (the ISO1042 symbol, re-labelled) | C528764 |
| `BLM31PG121SN1L` | 120 Ω 3.5 A ferrite bead | C85841 |

It also renamed the footprint `SW-SMD_DSHP02TSGER` to `SOP127P835X250-4N` in
`parts/myparts.pretty` (used by the `DSHP02TSGER` symbol). Other projects that
share the library still reference the old name.

These changes are in the submodule's working tree and are not committed. Commit
and push them in `parts/` first, then commit the new submodule pointer here.

## Checking the schematic

```sh
cd hardware/kicad
/Applications/KiCad/KiCad.app/Contents/MacOS/kicad-cli sch erc --severity-all cm5-servo-dish-controller.kicad_sch
```

Expected result: `Found 0 violations`.

## Fabrication

Not started. The reference board is a 4-layer, 150 × 90 mm PCB assembled by JLCPCB
with LCSC part numbers; this board is expected to follow it.

## STM32

`stm32/` is for the STM32G474RET6 firmware and its CubeMX `.ioc` file. Use the pin
table above when creating the `.ioc`.
