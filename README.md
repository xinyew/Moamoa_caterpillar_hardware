# Caterpillar — Hardware

KiCad project for the **caterpillar** board: a custom nRF54L15-based motor-control + IMU
board (vibration robot). Firmware lives in a sibling repo:
`Moamoa_caterpillar_firmware` (Zephyr / nRF Connect SDK v3.3.0, board target
`caterpillar/nrf54l15/cpuapp`).

## Board overview

| Subsystem | IC | Interface | Purpose |
|---|---|---|---|
| MCU + BLE | Raytac AN54LQ-P15 (nRF54L15) | — | Application core + BLE remote control |
| Motor driver | DRV8212P | PWM + GPIO | Dual-channel motor control |
| Digital pot | MAX5419LETA | I2C `0x28` | Adjusts motor supply voltage via STBB1-APUR |
| Buck-boost | STBB1-APUR | GPIO enable | Motor supply rail |
| Buck-boost | ADP2503 (3.3 V) | — | Logic supply rail |
| IMU | ASM330LHHTR | I2C `0x6A` | 6-axis accel/gyro |
| Inductor | TDK VLCF4020T-2R2N1R7 | — | 2.2 µH power inductor |

## Pin map (must stay in sync with firmware)

| Signal | Pin | Dir | Notes |
|---|---|---|---|
| I2C SDA | P1.02 | bidir | Shared bus: IMU + digipot (`i2c20`) |
| I2C SCL | P1.03 | out | Shared bus |
| IMU INT1 | P1.04 | in | ASM330LHHTR data-ready |
| IMU INT2 | P1.05 | in | ASM330LHHTR secondary interrupt |
| DRV8212 ~SLEEP | P1.06 | out | Active-low sleep (LOW = sleep) |
| PWM IN1 | P1.07 | out | Motor driver channel 1 (`pwm20`) |
| PWM IN2 | P1.08 | out | Motor driver channel 2 |
| DCDC EN | P2.03 | out | STBB1-APUR enable, active-high |

If you change any of these nets, update the devicetree in the firmware repo
(`boards/kamoamoa/caterpillar/`) and this table.

## Project files

```
caterpillar.kicad_pro   project settings
caterpillar.kicad_sch   schematic
caterpillar.kicad_pcb   PCB layout
caterpillar.csv         BOM export
sym-lib-table           project symbol libraries
fp-lib-table            project footprint libraries
library/                custom components (see rules below)
```

`caterpillar.kicad_prl` and `fp-info-cache` are per-user/cache files and are gitignored.

## Library component organization — the rules

Every custom component lives in its own folder under `library/`, laid out exactly like this:

```
library/<PART>/
  <PART>.kicad_sym          symbol library (named after the part, no dated filenames)
  footprints.pretty/        footprint library (one .kicad_mod per footprint variant)
  <model>.step              3D model (optional)
```

Conventions:

1. **Every footprint links its 3D model.** If the component has a STEP file,
   *each* `.kicad_mod` in `footprints.pretty/` (including -L/-M density variants)
   must contain a `(model ...)` clause pointing to it — otherwise the part renders
   as bare pads in the 3D viewer. Vendor downloads often ship the `.kicad_mod`
   without this clause; add it when importing.
2. **3D model paths are project-relative** — inside a footprint use
   `(model "library/<PART>/<model>.step")`, not `${KIPRJMOD}/...` and never an
   absolute path.
3. **The symbol's Footprint property** must be fully qualified:
   `<fp-lib-nickname>:<footprint-name>`, so placing the symbol auto-assigns
   the footprint.
4. **Register both tables** — add the `.kicad_sym` to `sym-lib-table` and the
   `footprints.pretty/` folder to `fp-lib-table`, both via `${KIPRJMOD}` URIs.
5. **Keep only what KiCad needs.** From vendor downloads (SamacSys/Mouser,
   UltraLibrarian, CSE) copy only the `.kicad_sym`, the `.kicad_mod`(s), and the
   STEP model. Do not commit legacy `.lib`/`.dcm`/`.mod` files, other-CAD folders,
   ImportGuides/readme junk, or the original .zip.

### Importing a new vendor component, step by step

1. `mkdir library/<PART>/footprints.pretty`
2. Copy `<vendor>.kicad_sym` → `library/<PART>/<PART>.kicad_sym`
3. Copy `*.kicad_mod` → `library/<PART>/footprints.pretty/`
4. Copy the `.step`/`.stp` → `library/<PART>/`
5. In the footprint(s), set the model path to `library/<PART>/<model>.step`
6. In the symbol, set the Footprint property to `<PART>:<footprint-name>`
7. Add one entry each to `sym-lib-table` and `fp-lib-table`
8. Verify in KiCad: symbol chooser shows the part, footprint 3D viewer shows the model

## Notes

- The AN54LQ-P15 module's symbol and footprint are **embedded** in the design files
  (no standalone library source in `library/`); to re-place or edit it, export it
  from the board via the Footprint Editor first.
- `Logos:kamoamoa` and `Kicad_library:Personal_small` footprints come from global
  (machine-level) libraries, not this repo; they are embedded in the board file.
- Fab history: boards ordered via PCBWay (production zips not tracked in git).
