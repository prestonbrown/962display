# 962display

A CAN-fed steering wheel display for a Porsche 962.

The engine runs on its original motorsport Motronic, which has no CAN. All data
comes from an **AiM PDM32** and the redundant sensor set hung off it — six EGT
probes, boost, oil and fuel pressure, temperatures, and the PDM's own per-output
current sensing.

The PDM does the logging. This is a pure real-time viewport: no SD card, no
filesystem, no persistence. It shows you what is happening right now, and it
gets out of the way.

## Shape

- **Panel:** 2.86" IPS, 376x960, ST7701S. Mounted landscape (960x376).
- **MCU:** STM32H7 (LTDC + DMA2D + FDCAN). See `docs/SPEC.md` for why.
- **Shift light:** discrete LED strip above the panel, driven from the CAN
  handler rather than the render loop.
- **Input:** two buttons for page next/prev, long-press to acknowledge alarms.

Everything crossing the slip ring is four wires: 12V, GND, CANH, CANL. The MCU
lives in the wheel.

## Layout

```
config/channels.yaml     Source of truth for the CAN contract
tools/gen_channels.py    -> C table + decoder, and the RaceStudio sheet
src/app/generated/       Generated. Do not edit.
src/app/                 Channel store, alarm state machine, page registry
src/ui/                  LVGL page implementations
src/hal/                 MCU-specific: LTDC, FDCAN, buttons, LED driver
sim/                     SDL simulator build — where the UI actually gets built
docs/SPEC.md             Design decisions and rationale
docs/RACESTUDIO.md       Generated. What to type into RaceStudio 3.
```

## Working on it

The UI is developed in the **SDL simulator on a desktop**, not on hardware. LVGL
abstracts the display down to a flush callback and a tick, so the MCU is a
porting detail rather than a blocking decision. Iterate on layout at 60fps with
a mouse instead of reflashing a board bolted inside a steering wheel.

```sh
tools/gen_channels.py          # regenerate after editing channels.yaml
tools/gen_channels.py --check  # CI gate: fails if generated files are stale
```

## Non-negotiables

Three rules that exist because the failure mode is worse than a blank screen:

1. **Stale data never renders as live data.** Every channel carries a timeout.
   Past it, the field goes to dashes. A confident frozen oil pressure reading is
   worse than no reading.
2. **Critical alarms latch and take over the screen.** A fault on page 3 must be
   visible from page 1. Latched alarms survive the condition clearing, and a
   single button press cannot dismiss them.
3. **The dash never transmits on CAN.** Listen-only, always. A firmware bug must
   not be able to error-frame the bus the PDM is running the car on.
