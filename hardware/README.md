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
- **Not yet done:** nothing here has been built or reviewed by a second person.
  See [Open items](#open-items).
- **Earlier version:** the git tag `48v-input` is the previous design, which took
  48 V on the board and made 5 V with an on-board LM5116 buck.

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
| Supply | 12.8 V battery | 5 V DC from an external Mean Well DDR-60L-5, which converts and isolates the 48 V battery. No 48 V on the board |
| 5 V rail | eFuse + AP64500 buck | The 5 V input, through a fuse and two P-channel MOSFETs (reverse polarity, on/off) |
| Module | CM4 | CM5 |
| USB | USB 2.0 hub, 2 × USB-A | The CM5's two USB 3.0 ports on a stacked USB 3.0 Type-A connector |
| USB-C | rpiboot through a mux | rpiboot, wired straight to the CM5 USB 2.0 port |

### Schematic sheets

| Sheet | File | Contents |
|---|---|---|
| Top | `cm5-servo-dish-controller.kicad_sch` | Block diagram, grounding, mounting holes |
| power-input | `power-input.kicad_sch` | 5 V terminal, fuse, reverse-polarity and on/off MOSFETs, TVS, bulk capacitors, power LED, design notes |
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

- **Supply:** a Mean Well DDR-60L-5 DIN-rail converter, mounted off the board. It
  takes the 48 V battery (18–75 V input) and gives 5 V at up to 12 A, isolated to
  4 kVdc. Its +Vo terminals (3, 4) go to J1 pin 1 and its −Vo terminals (1, 2) to
  J1 pin 2. The board's load budget is 5 A.
- **Input:** J1 screw terminal, F1 (5×20 mm fuse, T6.3A), Q8 and Q9, then the `5V`
  rail with D31 (SMCJ6.0A surge clamp), C105–C110 (224 µF bulk) and LED3.
  - Q8 (SI7137DP P-channel MOSFET, 2.5 mΩ) is the reverse-polarity protection. A
    reversed supply leaves it off: nothing reaches the board and F1 does not blow.
  - Q9 (a second SI7137DP, back to back with Q8) is the on/off switch, driven by
    SW1. SW1 switches the board only; the converter stays powered.
  - TP68 is a bare pad on `V_IN_F` between F1 and Q8, for checking the fuse. TP24
    is the `5V` rail.
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

1. **5 V setting.** The converter's output is adjustable from 4.5 V to 5.5 V. F1, Q8,
   Q9 and the cable drop about 0.2 V at 5 A. With the board running, adjust the
   converter until TP24 reads 5.10 V. The CM5 needs 4.75–5.25 V.
2. **Supply over-voltage.** The converter is rated for 18–75 V input. If the servo
   drives can pump the 48 V bus above 75 V when braking, fit brake resistors on the
   drives. The 48 V feed to the converter also needs its own fuse, off the board.
3. **Fuse.** F1 is the fuse holder; the assembler does not fit the fuse. Fit a
   Littelfuse 021806.3MXP (6.3 A time-lag, glass, LCSC C142706) by hand. It is
   named in F1's `Fuse` field. Do not fit a larger fuse: the holder is rated 6.3 A.
   The converter limits its output to 12.6–16.2 A, which is only 2.0 to 2.6 times
   the fuse rating, so into a short F1 opens slowly (within 2 minutes at 13.2 A)
   and may not open at 12.6 A. The converter's current limit is the short-circuit
   protection. Lay out the copper from J1 through F1, Q8 and Q9 for 16 A.
4. **CM5 module connectors.** J17/J18 are still the Hirose DF40C-100DS-0.4V(51)
   parts from the CM4 board. The CM5 datasheet names Amphenol 10164227-1001A1RLF
   as the mating part, tested to the CM5's higher current.
5. **Self power-cycle circuit** (`nPWR_CYCLE`, U10) is carried over from the CM4
   board and is not proven on a CM5.
6. **Console UART notes** on the connectors sheet (how to enable the Linux console
   on GPIO14/15 on a CM5) need checking against current Raspberry Pi documentation.
7. **Part stock.** All fitted parts were checked against live JLCPCB stock on
   2026-11-17. Two that were out of stock were replaced: U25 (now CA-IS3052G,
   same pinout and footprint as the ISO1042) and USB1 (now Molex 48406-0003).
   Still low: U18 TPS70933DBVR (10), U20 LTC4311ISC6 (15), U7 LM75BIMX-3 (54),
   U24 STM32G474RET6 (72), J19 FTSH-107-01-L-DV-K-A (173). Recheck stock before
   ordering.
   The 5 V input parts were checked the same day: Q8/Q9 SI7137DP-T1-GE3 (C784367)
   and D31 SMCJ6.0A/TR13 (C310065) each had over 100,000 in stock.
   The USB1 footprint comes from EasyEDA; check it against the Molex drawing
   before layout.
8. **Net classes** in the project file were updated for the new net names, but the
   track widths are still the 12 V board's. `/5V` is in the `PWR_3A` class (1.0 mm)
   and the input nets in `V_IN` (2.0 mm): neither is enough for the 16 A input
   path in item 3, which needs copper pours.
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
    `QFP50P1200X1200X160-64N` (STM32), `SON127P615X515X120-9N` (Q8/Q9),
    `SOIC127P1150X280-8N` (CA-IS3052G), `SOT95P240X110-3N` (PESD2CANFD27V),
    `RESC2012X60N`, `OSCCC320X250X80-4N` (X2) and `SOP127P835X250-4N` (SW2).
  - `SON127P615X515X120-9N` was made for the BSC098N10NS5 (TDSON-8). Q8/Q9 are in
    Vishay's PowerPAK SO-8, which has the same land pattern and pin order; its
    maximum height is 1.12 mm, not the 1.20 mm in the name.
  - Four footprints made for the 48 V stage are no longer used and are still in
    the library: `SOP65P640X120-21N` (LM5116), `CAPAE1030X1250N`,
    `INDM135125X650N` and `INDC3216X130N`.
  - Parts with no IPC-7351 package family (connectors, headers, modules, switches,
    the CAN choke, the battery, the RH-5016 test points) are named manufacturer_part,
    e.g. `Molex_48406-0003`, `HanRun_HR911130A`, `PinHeader_2x12_P2.54mm_Vertical`.
  - D31 uses the reference project's SMC pattern `DIOM7959X262N`
    (2.7 × 3.3 mm pads instead of 3.03 × 3.82 mm).
  - `Samtec_FTSH-107-01-L-DV-K-A` (J19) numbers its pads `01`–`14` to match the
    STDC14 symbol. The `MyParts` footprint numbers them `1`–`14`, which left
    J19 pins 1–9 without pads.
- **3D models:** set the KiCad path variable `MYPARTS` to `<clone>/parts` and
  `MYPARTS_DATASHEETS` to `<clone>/parts/datasheets`
  (**Preferences → Configure Paths**).

This design added these symbols to `parts/myparts.kicad_sym`, with footprints and
3D models where the library had none. The 48 V parts among them (`LM5116MHX`,
`BSC098N10NS5`, `TMPC1265HP-100MG-D`, `RLP25FEGR015`, `CL31Y225KCHVPNE`,
`EFVH100ADA101M10C0`, `0402CG331J500NT`, `SS510C`, `SMCJ58A`, `BAV21WS_C2903901`,
`BLM31PG121SN1L`) are no longer used by this board:

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
| `SI7137DP-T1-GE3` | 20 V P-channel MOSFET, 2.5 mΩ at 4.5 V, PowerPAK SO-8 | C784367 |
| `SMCJ6.0A` | 6 V TVS, 1.5 kW, SMC | C310065 |

It also renamed the footprint `SW-SMD_DSHP02TSGER` to `SOP127P835X250-4N` in
`parts/myparts.pretty` (used by the `DSHP02TSGER` symbol). Other projects that
share the library still reference the old name.

These changes are committed in `parts/`, except the last two symbols
(`SI7137DP-T1-GE3`, `SMCJ6.0A`), which are in the submodule's working tree. Commit
and push those in `parts/` first, then commit the new submodule pointer here.

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
