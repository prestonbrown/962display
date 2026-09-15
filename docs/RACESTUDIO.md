# RaceStudio 3 — PDM32 custom CAN output stream

GENERATED from `config/channels.yaml`. Do not hand-edit; regenerate with
`tools/gen_channels.py`.

- **Bitrate:** 500000 bit/s
- **ID format:** standard 11-bit
- **Byte order:** little-endian unless a frame says otherwise
- **All signals:** 16-bit signed, byte-aligned, no multiplexing

The dash is configured listen-only and never transmits, so nothing here
needs an inbound path on the PDM.

## Reserved ID blocks

| Block | Range | Nominal rate | Contents |
|-------|-------|--------------|----------|
| high | 0x600–0x60F | 50 Hz | RPM, boost, pressures |
| mid | 0x610–0x61F | 20 Hz | EGT, lambda |
| low | 0x620–0x63F | 5 Hz | temps, electrical, PDM currents |

## Frames

### 0x600 — ENGINE_FAST @ 50 Hz

| Byte | Channel | Label | Unit | Scale | Decimals |
|------|---------|-------|------|-------|----------|
| 0–1 | `rpm` | RPM | — | 1.0 | 0 |
| 2–3 | `boost` | BOOST | bar | 0.01 | 2 |
| 4–5 | `oil_press` | OIL P | bar | 0.01 | 1 |
| 6–7 | `fuel_press` | FUEL P | bar | 0.01 | 1 |

### 0x601 — DRIVER_FAST @ 50 Hz

| Byte | Channel | Label | Unit | Scale | Decimals |
|------|---------|-------|------|-------|----------|
| 0–1 | `tps` | TPS | % | 0.1 | 0 |
| 2–3 | `gear` | GEAR | — | 1.0 | 0 |

### 0x610 — EGT_A @ 20 Hz

| Byte | Channel | Label | Unit | Scale | Decimals |
|------|---------|-------|------|-------|----------|
| 0–1 | `egt_1` | EGT 1 | C | 1.0 | 0 |
| 2–3 | `egt_2` | EGT 2 | C | 1.0 | 0 |
| 4–5 | `egt_3` | EGT 3 | C | 1.0 | 0 |
| 6–7 | `egt_4` | EGT 4 | C | 1.0 | 0 |

### 0x611 — EGT_B @ 20 Hz

| Byte | Channel | Label | Unit | Scale | Decimals |
|------|---------|-------|------|-------|----------|
| 0–1 | `egt_5` | EGT 5 | C | 1.0 | 0 |
| 2–3 | `egt_6` | EGT 6 | C | 1.0 | 0 |
| 4–5 | `lambda_1` | LAM L | — | 0.001 | 2 |
| 6–7 | `lambda_2` | LAM R | — | 0.001 | 2 |

### 0x620 — TEMPS @ 5 Hz

| Byte | Channel | Label | Unit | Scale | Decimals |
|------|---------|-------|------|-------|----------|
| 0–1 | `water_temp` | WATER | C | 0.1 | 0 |
| 2–3 | `oil_temp` | OIL T | C | 0.1 | 0 |
| 4–5 | `air_temp` | IAT | C | 0.1 | 0 |
| 6–7 | `fuel_temp` | FUEL T | C | 0.1 | 0 |

### 0x621 — ELECTRICAL @ 5 Hz

| Byte | Channel | Label | Unit | Scale | Decimals |
|------|---------|-------|------|-------|----------|
| 0–1 | `battery_v` | BATT | V | 0.01 | 1 |
| 2–3 | `pdm_i_fan_l` | FAN L | A | 0.1 | 1 |
| 4–5 | `pdm_i_fan_r` | FAN R | A | 0.1 | 1 |
| 6–7 | `pdm_i_fuelpmp` | F PUMP | A | 0.1 | 1 |

### 0x622 — ELECTRICAL_2 @ 5 Hz

| Byte | Channel | Label | Unit | Scale | Decimals |
|------|---------|-------|------|-------|----------|
| 0–1 | `pdm_i_waterpmp` | W PUMP | A | 0.1 | 1 |

## Bus budget

Worst case (full 8-byte frames, maximum bit stuffing): **~4.0% of 500 kbit/s**.

Plenty of headroom. Add channels in the appropriate ID block rather than
repacking existing frames.
