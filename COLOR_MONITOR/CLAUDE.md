# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Does

**COLOR_MONITOR** is a real-time LED wall control system. A webcam captures live video; the Python script processes each frame and streams it to two types of LED displays simultaneously:

1. **ESP LED panels** (6x panels, 15×44 px each = 90×44 total) — streamed over UDP
2. **Arduino LED matrix** (4× 8×32 panels = 32×32 total) — streamed over serial at 500000 baud

The system also does color detection (CIE LAB Delta-E CIE2000) and can export color palettes as JSON/PNG.

## Running the Application

```bash
# Install dependencies
pip install -r requirements.txt   # opencv-python, numpy, pygame, pyserial

# Run
python minimalv2.py
```

**Runtime controls:**
- `F` — toggle fullscreen
- `I` — invert colors (Common Anode mode)
- `Q` / `ESC` — quit

## Configuration (top of `minimalv2.py`)

All hardware configuration is hardcoded near the top of `minimalv2.py`:

- `ESP_IPS` — list of ESP panel IPs (default: 6 panels on 192.168.1.x)
- `ESP_PORT` — UDP port (default: 4210)
- `PANEL_WIDTH` / `PANEL_HEIGHT` — single ESP panel dimensions (15×44)
- `ARDUINO_ENABLED` / `ARDUINO_PORT` / `ARDUINO_BAUD` — serial config (`"auto"` = auto-detect)
- `ARDUINO_PANEL_ORDER` / `ARDUINO_PANEL_START_BOTTOM` / `ARDUINO_SERPENTINE_X` — physical wiring layout
- `ESP_SERPENTINE_HORIZONTAL` / `ESP_START_BOTTOM` — ESP wiring serpentine direction
- `GAMMA` — gamma correction value (default 2.5)
- `COMMON_ANODE` — invert RGB for common-anode strips

## Architecture

### Python (`minimalv2.py`)

**Data flow per frame:**
1. Webcam frame captured via OpenCV
2. Frame horizontally flipped, aspect-ratio preserved, centered on a black `TOTAL_WIDTH × PANEL_HEIGHT` canvas
3. Gamma correction applied via pre-computed lookup table
4. **ESP path**: canvas sliced into `len(ESP_IPS)` vertical strips → serpentine reordering → split into two UDP packets per panel (one for each half, prefixed with `0x00` or `0x01`)
5. **Arduino path**: frame resized to `ARDUINO_COLS × ARDUINO_ROWS` → gamma → `map_frame_to_leds()` remaps to physical serpentine wiring → sent as `b'V' + 3072 raw RGB bytes` over serial

**Key functions:**
- `map_frame_to_leds(frame_rgb)` — remaps a 32×32 image to the physical serpentine order of 4 chained Arduino panels. Panel order and start direction are controlled by `ARDUINO_PANEL_ORDER` and `ARDUINO_PANEL_START_BOTTOM`.
- `send_arduino_frame(ser, frame)` — full Arduino frame pipeline (resize → gamma → remap → serial write)
- `detect_center_color(frame)` / `detect_grid_colors(frame)` — color detection using CLAHE + K-Means + Delta-E CIE2000 matching against `COLOR_DATABASE`
- `find_closest_color(rgb)` — matches any RGB to the nearest named color in the ~200-entry bilingual (EN/IT) `COLOR_DATABASE`

### Arduino (`arduino_palette_sketch/arduino_palette_sketch.ino`)

Runs on FastLED with WS2812B strips. Listens on serial at 500000 baud for two protocols:

- **Video mode**: byte `'V'` followed by `NUM_LEDS * 3` raw RGB bytes → direct `FastLED.show()`
- **Single color**: `"R,G,B\n"` → fill with sweep animation
- **Palette mode**: `"P:N:RRGGBB:RRGGBB:...\n"` → divide LEDs into N color blocks with sweep animation and crossfade

Color calibration factors (`RED_FACTOR`, `GREEN_FACTOR`, `BLUE_FACTOR`) are tuned for the specific LED strip model (LOEFL1RGB/6024).

## Hardware Notes

- Arduino matrix: 4 panels × 8×32 = 1024 LEDs, chained as one `FastLED` strip on pin 6
- Serial baud must match both ends: `ARDUINO_BAUD = 500000` in Python and `Serial.begin(500000)` in the sketch
- Each ESP panel receives two UDP packets per frame (330 LEDs each, 990 bytes + 1 header byte)
- The 3ms sleep between ESP panels (`time.sleep(0.003)`) prevents router congestion
