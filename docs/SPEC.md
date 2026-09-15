# 962display — design spec

Status: draft. Nothing here has been near the car yet.

---

## 1. What this is

A wheel-mounted real-time instrument for a Porsche 962. It reads a CAN stream
broadcast by an AiM PDM32, renders it on a 376x960 panel mounted landscape, and
drives a shift light strip.

It is deliberately *not* a logger, a config tool, or a second ECU display. The
PDM32 logs. This shows.

### Division of labour

| Concern | Owner |
|---|---|
| Logging, history, analysis | PDM32 |
| Warnings of record | PDM32 |
| Real-time glance instrument | this |
| Shift light (wheel) | this |
| Shift light (dash) | existing, unchanged |

Because the PDM owns persistence, this device holds **no state across power
cycles**. No SD card (a liability in a race car, not a feature), no flash wear,
no "did it flush before the master switch opened" question. Stint min/max on the
fluids page lives in RAM and dies with the ignition, which is correct — the
version that matters is in the PDM.

---

## 2. Hardware

### Panel

2.86" IPS, 376x960, **ST7701S** driver IC, 16-bit parallel RGB (RGB565).

The governing constraint: **ST7701S has no GRAM.** It is a video-mode
controller, not an ST7789. You cannot push a framebuffer over SPI and walk away
— the host clocks out pixels continuously at ~26 MHz, forever. The board's
SDA/SCL/CS pins are only for the ~50-register init sequence (bank-switched
BK0/BK1 gamma and power rails), and those values are panel-specific. Get them
from the vendor's demo code; they are not guessable.

Rotated to 960x376 this is close to the classic race-dash proportion: full-width
rpm bar, big centre number, flanking values.

Two notes on the dev board as supplied:

- The 0.1" header and the U1 ZIF are hobby grade. On a steering wheel the FPC
  *will* walk out of that connector. The production build needs a carrier PCB
  with the panel bonded and the FPC retained.
- Verify panel brightness before committing. A generic IPS is 300-400 nits;
  automotive wants 800-1000. A 962 cockpit is enclosed and the wheel sits in
  shadow, so it may be fine — but this cannot be fixed in software.

Unverified: the silkscreen claims 376x960, which exceeds the ST7701S's
documented 480x864. Either the silkscreen is optimistic or this is a
non-standard configuration. Confirm against the vendor init sequence.

### MCU — STM32H7

Chosen for thermal margin and memory headroom. Not for rendering capability —
several cheaper parts can drive this panel fine.

#### The framebuffer decides everything

376x960 = 360,960 pixels.

| Depth | Framebuffer |
|---|---|
| RGB565 (16 bpp) | 705 KB |
| L8 indexed (8 bpp) | 352 KB |
| L4 indexed (4 bpp) | 176 KB |

Memory traffic is active pixels only — the LCD DMA does not fetch during
blanking, so the pixel clock overstates it:

| Refresh | Sustained read |
|---|---|
| 60 Hz | ~43 MB/s |
| 50 Hz | ~36 MB/s |

Every candidate below has a display controller. Memory is what filters them.

#### Candidates

| Part | Core | SRAM | LTDC/RGB | Verdict |
|---|---|---|---|---|
| STM32L073RZ | M0+ 32 MHz | 20 KB | none | Different species. No display controller, no CAN, and 20 KB is 6% of the smallest useful framebuffer. |
| STM32F746ZG | M7 216 MHz | 320 KB | LTDC | Has the controller, cannot feed it. 320 KB total (~240 KB contiguous) is short of even L8. Needs external SDRAM — which is exactly why the F746G-DISCO carries 8 MB of it. |
| STM32F767ZI | M7 216 MHz | 512 KB | LTDC | L8 only, with ~16 KB of margin in SRAM1. Works; no room to grow, no double buffering, locked to 256 colours. |
| **STM32H7A3ZI** | M7 280 MHz | 1.4 MB | LTDC | **Chosen.** RGB565 fits internally with ~700 KB spare. No external memory, no cleverness. |
| ESP32-S3 (N8R8/N16R8) | LX7 240 MHz | 512 KB + 8 MB octal PSRAM | LCD_CAM | Capable. Loses on temperature grade and GPIO budget — see below. |
| RP2350 | M33 150 MHz | 520 KB | PIO | 520 KB is less than one framebuffer, and external PSRAM bandwidth is worse than the S3's. |

#### Why not the ESP32-S3

**Bandwidth is not the reason.** That objection does not survive arithmetic:
octal PSRAM measures ~80 MB/s in practice, so 43 MB/s at 60 Hz is roughly 53%
duty with half the bandwidth left for drawing. The empirical case is stronger
still — the ESP32-8048S043-class boards (Sunton, Guition, Waveshare) run
800x480 RGB565 under LVGL in volume, and that is a **768 KB** framebuffer,
larger than this one. Anyone re-deriving "the S3 can't do it" should stop here.

The two objections that do hold:

- **Temperature.** WROOM-1 modules carrying PSRAM are typically rated
  -40 to +85 C. A closed 962 cockpit in summer, inside a sealed printed
  enclosure beside an LED strip, does not leave comfortable margin. The H7 has
  industrial and automotive grades, and PSRAM is usually the first thing to get
  flaky.
- **GPIO budget**, which is far tighter than bandwidth ever was. Octal PSRAM
  consumes GPIO 35/36/37 on top of the flash pins. Against roughly 33 usable
  pins: 18-20 for RGB, 3 panel init SPI, 2 RST/BL, 2 CAN, 3 LED driver, 2
  buttons. It fits with no slack, and leaving room for the thing not yet thought
  of is a theme of this design.

If an S3 is used anyway, the part number is the whole decision: **R8 is 8 MB
octal PSRAM and is required. R2 is 2 MB quad PSRAM — half the bus width — and
genuinely cannot do this.**

#### What the H7 buys

- **LTDC** has its own FIFO and DMA path. Deterministic rather than contended.
- **LTDC supports L8** — 8-bit indexed with a hardware CLUT, halving the
  framebuffer and making day/night a palette swap rather than a redraw. The
  S3's RGB peripheral cannot do this; it needs real RGB565 in memory.
- **DMA2D (Chrom-ART)** does fills, blits and format conversion in hardware.
  A dash is entirely rectangles, bars and glyph blits — exactly this workload.
- **FDCAN**, two instances. Second bus for free. (Classic CAN 2.0B is all the
  AiM stream needs, so bxCAN parts are not disqualified by this.)
- Boots in milliseconds. Alive before the engine catches.
- No Wi-Fi/BT radiating in the car for no reason.

#### Part options

Digikey single-unit, Sept 2026. Note the board price, not the chip price, is
the real cost of entry until a carrier PCB exists.

| Part | Price | Flash / RAM | Notes |
|---|---|---|---|
| NUCLEO-H7A3ZI-Q | ~$40 | — | The board to actually buy first. |
| STM32H7A3ZIT6 | ~$15 | 2 MB / 1.4 MB | For the carrier PCB. First choice. |
| STM32H750VBT6 | ~$12 | 128 KB / 1 MB | Value line. Needs external QSPI flash (XIP), which it is designed for. |
| STM32H743VIT6 | ~$12 | 2 MB / 1 MB | Well-trodden, large body of existing LVGL work. |

TODO: confirm the contiguous AXI SRAM block size on the exact H7A3 part before
committing to an internal framebuffer. Fallback is FMC + SDRAM (~200 MB/s),
comfortably above the ~43 MB/s requirement.

TODO: verify in CubeMX that the LTDC alternate-function pins land on Zio
headers actually exposed on the Nucleo-144 before relying on that board for
panel bring-up.

### Shift light

Discrete LEDs above the panel. **Not rendered on the LCD** — panel response plus
a 50 Hz refresh puts you in the 30-40 ms range, and at 8000 rpm that is late.

- **Driver:** constant-current (TLC5947, LP5030, IS31FL3-series) over SPI/I2C,
  not WS2812. Addressables have fussy timing, mediocre temperature behaviour,
  and no precise current control.
- **Update path:** driven directly from the CAN RPM frame handler, not the
  render loop. ~10 ms end to end instead of ~40 ms.
- **Sequence:** progressive fill green -> amber -> red, then full-strip flash at
  and past the shift point. Conventional because it reads preattentively — you
  see a shape, you do not count LEDs.
- **Brightness:** day/night is mandatory, not optional. It must be blinding at
  noon and not blinding at 2am. Source from an ambient sensor, headlight state
  off the PDM, or a manual toggle. Pick one.
- **Per-gear shift points** where gear is available on the bus.

### Electrical

PDM outputs are fused, switched and protected, so the feed is cleaner than a raw
battery tap. Still protect locally: reverse polarity, TVS, and a front end rated
for ISO 7637-2 load-dump transients. Test the master-cutoff case explicitly.

### Wiring through the slip ring

**Four conductors: 12V, GND, CANH, CANL.** The MCU lives in the wheel. This is
why 16-bit RGB never crosses the rotating joint, and it is the single constraint
that shaped the whole architecture.

If you are specifying the slip ring or coiled cable, **buy two more conductors
than you need.** They cost nothing now and are free options later.

---

## 3. Data architecture

### Channels are data, not struct fields

The mistake to avoid is `struct { uint16_t rpm; uint16_t boost; ... }` with
pages referencing `state.rpm`. That makes adding channel #24 a firmware
refactor.

Instead: one table indexed by a stable channel ID, holding value, timestamp,
scaling, units and alarm thresholds. Frames decode into slots. Pages reference
slots by ID. Alarm logic is generic and does not grow per channel.

**Adding a channel = one row in `channels.yaml` + one row in RaceStudio.**
Nothing else changes. That single decision is most of the headroom.

### One source of truth

`config/channels.yaml` describes the contract. `tools/gen_channels.py`
generates the C table, the decoder, and `docs/RACESTUDIO.md` — the sheet you
type into RaceStudio 3.

This exists because otherwise, eighteen months from now, someone nudges a
scaling factor in RaceStudio and oil pressure reads 10x with nobody able to say
why. `--check` is a CI gate for staleness.

The generator also validates: duplicate IDs, signals overrunning byte 8,
overlapping byte claims within a frame, pages referencing unknown channels, and
that the bus stays listen-only.

### CAN

Listen-only, 500 kbit/s, standard 11-bit IDs. Reserved blocks with deliberate
gaps, so growth is "next free ID in the right block" and never "repack
everything":

| Block | Range | Rate | Contents |
|---|---|---|---|
| high | 0x600-0x60F | 50 Hz | RPM, boost, pressures |
| mid | 0x610-0x61F | 20 Hz | EGT, lambda |
| low | 0x620-0x63F | 5 Hz | temps, electrical, PDM currents |

Rate tiering is not premature optimisation, it is matching the physics. A
sheathed thermocouple's time constant is hundreds of milliseconds — EGT at
50 Hz measures noise. Pressures are fast and are what end engines.

Current budget is ~6% of the bus. There is a lot of room.

Keep the wire format dumb: 16-bit signed, byte-aligned, no multiplexing, no
bit-packing across byte boundaries. The decoder should be a load and a shift.

### Decode/render separation

The CAN handler writes the table and a timestamp. The render task samples at
50 Hz. **Never draw from the CAN handler.** Take a coherent snapshot per frame
so you cannot render RPM from t and gear from t+1.

---

## 4. Pages

A page is a table entry: layout template + channel ID list + title.

| # | Page | Content |
|---|---|---|
| 1 | RACE | RPM bar, gear, big centre number, temp/pressure strip |
| 2 | ENGINE | 6 EGT bars by deviation, boost, hottest-cylinder delta |
| 3 | FLUIDS | Oil P/T, fuel P, water, with volatile stint min/max |
| 4 | ELEC | Battery V, PDM per-output currents |
| 5 | DIAG | Frame counters, bus errors, per-channel age, firmware version |

Page 5 is not optional. You will be in a paddock at 2am wondering why one value
is frozen, and "channel 14 last seen 47 seconds ago" answers it in two seconds.

Boot always lands on page 1. No "remember last page" — predictable beats clever
when you are strapped in.

### EGT rendering

Absolute EGT is nearly useless at a glance. What matters on a twin-turbo
flat-six is **spread**: one cylinder 80 C above its neighbours is a lean
cylinder heading for detonation, and that is the failure that ends the engine.

So render the six bars coloured by **deviation from the bank mean**, not by
absolute value, with a numeric callout for the hottest cylinder and its delta.
All six climbing together is you leaning on it. One climbing alone is an
emergency. From three feet away, in peripheral vision, at speed, those two
states must not look similar.

---

## 5. Alarms

The alarm layer is owned by the app, above the page system, because **a fault on
page 3 must be visible from page 1.**

| Level | Behaviour |
|---|---|
| Warning | Persistent banner in a reserved strip. Page stays usable. |
| Critical | Full-screen takeover. Overrides page selection entirely. |

Critical alarms **latch**. They do not clear themselves when the condition
passes — a pressure dip that self-heals in 200 ms still happened. Clearing is
deliberate: a long press, or a key cycle. A single press must not be able to
hide a real fault, because a steering wheel on a race car is a vibration test
fixture with a driver attached.

### Staleness

Every channel carries `timeout_ms`. Past it the channel is STALE and renders as
dashes, greyed. **A confident, frozen, wrong oil pressure is strictly worse than
no reading.** This is the rule people skip and it is the one that matters.

Distinguish three states, visually: never-seen, live, and stale-after-live. They
mean different things — the third one means something just broke.

---

## 6. Buttons

Two. One works but you will hate it.

| Input | Action |
|---|---|
| Left / Right, short | Previous / next page, wrapping both ways |
| Either, long (~1s) | Acknowledge latched alarm |
| Both, long (~2s) | Brightness / config |

Debounce hard: RC on the input plus a software stable-for-25 ms requirement.

If wheel buttons already land on PDM32 digital inputs, page navigation can
arrive **over CAN** — no extra conductors through the slip ring, no local
button harness. Worth checking before routing anything.

---

## 7. Bring-up plan

### Hardware on hand

Surveyed Sept 2026. Determines what can start today versus what needs ordering.

| Item | What it is | Role |
|---|---|---|
| Raspberry Pi Zero 2 W | RP3A0-AU, quad A53, 512 MB | **Panel bring-up rig** |
| RP2040 board (16 MB flash) | 264 KB SRAM, USB-C | **Bench PDM simulator** |
| LilyGO TTGO T-Display | ESP32 (original), 1.14in ST7789 135x240 SPI | LVGL familiarisation only |
| ideaspark ESP-WROOM-32 | ESP32 (original) | Not applicable |

**There is no ESP32-S3 here.** Both ESP32 boards are the original 2016 part, and
**LCD_CAM is S3-only** — the original ESP32 has no parallel RGB output at all.
Its I2S parallel mode drives 8-bit MCU-interface panels, not 16-bit RGB at
26 MHz, and WROOM-32 has 520 KB SRAM with no PSRAM, so a 705 KB framebuffer has
nowhere to live. Neither board can drive this panel. The S3 analysis in section
2 stands as reasoning about the part; it does not describe anything on the
bench.

### Three tracks, deliberately independent

The panel, the data and the UI de-risk separately. None blocks the others.

| Track | Platform | Proves |
|---|---|---|
| **Panel** | Pi Zero 2 W + DPI | ST7701S init sequence, real timings, whether 376x960 is true |
| **CAN** | RP2040 + can2040 + transceiver | The generated decoder against real frames at real rates |
| **UI** | SDL simulator on desktop | Layout, pages, alarm behaviour, EGT deviation rendering |
| *(later)* Target | NUCLEO-H7A3ZI-Q | LTDC config, FDCAN against the live PDM stream |

#### Panel track — Pi Zero 2 W

Pi DPI in RGB565 mode is exactly this panel's interface: 16-bit parallel RGB
plus PCLK/DE/HS/VS. Mainline carries `panel-sitronix-st7701`, so this is driver
configuration rather than bringup from scratch, and 512 MB of RAM makes the
framebuffer arithmetic of section 2 irrelevant for this purpose.

From Bookworm onward it is one overlay line plus the panel's timings:

```
dtoverlay=vc4-kms-dpi-generic,rgb565          # GPIO 0-19
dtoverlay=vc4-kms-dpi-generic,rgb565-padhi    # GPIO 0-8, 12-17, 20-24
```

The point is to answer what no datasheet will: do the vendor timings work, does
the init sequence work, and is the panel really 376x960 given that exceeds the
ST7701S's documented 480x864.

Known friction: the 40-pin header ships unpopulated and needs soldering; DPI
consumes ~20 GPIOs either way; and the generic DPI overlay has a documented
history of working on CM4 while failing on Zero 2 W
(<https://forums.raspberrypi.com/viewtopic.php?t=325759>, resolved) — read that
before starting.

This is a bring-up rig, not a candidate target. Boot time and SD corruption on
power-cut rule a Pi out of the car, as section 2 records.

#### CAN track — RP2040 as a bench PDM

`can2040` implements CAN 2.0B entirely in RP2040 PIO, so the board plus a
transceiver is a complete CAN node with no MCP2515.

That makes it a bench stand-in for the PDM32: replay the frame set from
`channels.yaml` at the configured rates, sweep RPM through the shift points,
stop transmitting one channel to exercise the staleness path, drop oil pressure
to exercise the latching alarm. It exercises the generated decoder rather than a
stub, so the thing under test is the thing that ships.

### To order

- CAN transceivers x2 (SN65HVD230 / TJA1050 class) — one for the simulator, one
  for the dash
- 2x20 0.1in header for the Zero 2 W
- NUCLEO-H7A3ZI-Q

No level shifting needed anywhere: the panel, the Pi and the H7 are all 3.3V
logic.

### Firmware update path — decide now

How do you reflash a board bolted inside a steering wheel? An accessible USB-C
stub, or a CAN bootloader. Retrofitting either after the wheel is assembled and
the carbon is finished is genuinely painful. This is the cheapest decision to
make today and one of the most expensive to defer.

---

## 8. Open questions

- [ ] Confirm every alarm threshold in `channels.yaml` against the engine
      builder's numbers. All current values are placeholders.
- [ ] Confirm redline and per-gear shift points.
- [ ] How many custom CAN output streams does RaceStudio 3 expose on the PDM32,
      and at what maximum rate?
- [ ] Is there an AiM dash or logger already on that bus? Changes nothing
      architecturally but affects ID allocation.
- [ ] Wheel buttons: PDM digital inputs, or free for local GPIO?
- [ ] Panel brightness spec — measured, not claimed.
- [ ] Contiguous AXI SRAM on the chosen H7 part; internal framebuffer or SDRAM.
- [ ] Slip ring conductor count and current rating.
- [ ] Which PDM outputs are worth surfacing as current channels.
- [ ] Source the vendor ST7701S init sequence and timings. Nothing else on the
      panel track can start without it.
- [ ] Confirm the Zero 2 W DPI overlay drives this panel at all, before trusting
      it as the bring-up rig.
