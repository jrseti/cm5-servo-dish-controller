# CM5 Servo Dish Controller

Servo controller for a dish antenna, built around the Raspberry Pi Compute Module 5 (CM5).

This repository contains:

- **Hardware** – KiCad schematic and PCB design files
- **Software** – Python control code

## Repository layout

```
.
├── hardware/   # KiCad project (schematic, PCB, libraries, fabrication outputs)
└── software/   # Python controller code
```

## Hardware

Open the KiCad project in `hardware/` with KiCad.

<!-- TODO: KiCad version, board overview, fabrication notes -->

## Software

### Setup

```sh
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

<!-- TODO: how to run the controller, configuration, hardware interfaces -->

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
