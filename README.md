# cmdr-pico-breadboard-kit

The [GeeekPi Pico Breadboard Kit](https://wiki.52pi.com/index.php?title=EP-0172) —
3.5" touch TFT, joystick, buttons, LEDs, RGB LED and a buzzer — running as a
[commander](https://github.com/gbryant/commander) project on a **Pico 2 W (RP2350)**.

Every peripheral on the board is a commander module, so the whole device is
reachable from a shell over USB serial or telnet:

```
> lcd test                  colour bars, border and text on the panel
> touch watch               stream touch coordinates
> joy                       stick position and direction
> btn                       button states and press counts
> led 0 blink 250
> buzz play c4:200,e4:200,g4:400
> wled 0 40 0               the RGB LED
> kit selftest              exercise everything in turn
```

`main.cpp` contains no driver code — the drivers are framework modules. What's
here is the *composition*: which screen is showing, what a touch means right now,
what the stick and buttons do.

> **Status: not yet hardware-tested.** Everything builds for RP2350 and the
> drivers are covered by host tests against a recording HAL (every byte they'd
> put on SPI/I2C is asserted), but no part of this has met the actual board yet.
> [`docs/hardware-test.md`](docs/hardware-test.md) is the bring-up checklist,
> in the order things should be switched on.

## Screens

| Screen | What it shows |
|--------|---------------|
| **Status** | hostname, IP, uptime, and the last input event — the default screen |
| **Piano** | eight touch keys across the panel; the buzzer plays, the RGB LED follows the key |
| **Splash** | the boot card |

Controls, on any screen:

| Input | Action |
|-------|--------|
| touch | Piano: play a key. Status: report the coordinate |
| stick left / right | previous / next screen |
| stick up / down | backlight brighter / dimmer (Piano: transpose an octave) |
| button 0 / 1 | toggle LED 0 / 1, with a click |
| `kit …` | the same things from the shell |

## Hardware

Pin assignments are the kit's, recorded in `cmdr.toml`. Change them there and run
`cmdr regen` — nothing is hardcoded in the app.

| Peripheral | Wiring | Module |
|------------|--------|--------|
| 3.5" TFT, ST7796S, 320×480 | SPI0 — SCK GP2, MOSI GP3, CS GP5, DC GP6, RST GP7 | `st7796` |
| Capacitive touch, GT911 | I2C0 — SDA GP8, SCL GP9, addr 0x5D | `gt911` |
| Joystick | ADC0 GP26 (X), ADC1 GP27 (Y) | `joystick` |
| Buttons BTN1 / BTN2 | GP15, GP14 (active low) | `buttons` |
| LEDs D1 / D2 | GP16, GP17 | `leds` |
| Buzzer | GP13 (PWM) | `buzzer` |
| RGB LED (WS2812) | GP12 (PIO) | `ws2812` |

The panel's backlight is hard-wired on for this kit, so `lcd bl 0` falls back to
the panel's own display-off rather than dimming. If you wire the backlight to a
spare GPIO, set `bl` in `cmdr.toml` and it becomes a real PWM dimmer.

## Build and run

Needs `PICO_SDK_PATH` and `FREERTOS_KERNEL_PATH` set — see commander's
[getting-started](https://github.com/gbryant/commander/blob/main/docs/getting-started.md).

```bash
./build        # cmake build → build-pico2/
./bum          # build + upload (BOOTSEL) + monitor
./monitor      # console only
```

WiFi credentials live in `secrets.h` (gitignored; `secrets.h.example` is the
template). Telnet comes up on port 23 once WiFi connects, and the status screen
shows the IP.

### Debugging over SWD

With a CMSIS-DAP probe (the RP2350 debug probe, or a spare Pico running
`debugprobe`) wired SWCLK/SWDIO/GND to the kit:

```bash
./swd-flash    # flash over SWD — no BOOTSEL, board stays wired
./swd-debug    # openocd + arm-none-eabi-gdb attached
./swd-reset    # reset the board when the console is wedged
```

`openocd.cfg` is committed (unlike the generated `build`/`bum` scripts) because
it's a wiring description, not a build artifact.

## Framework dependency

This project uses framework modules that are **not in a released commander tag
yet** — they live on the `feat/pico-breadboard-kit` branch, together with the
HAL's new SPI/ADC/PWM entry points that the display, joystick and buzzer need.

Until that branch is merged and tagged, build against a local checkout:

```bash
cmdr link ~/github/commander      # with the branch checked out
```

`cmdr.toml` and `commander_modules.h` are otherwise ordinary: once the branch
ships in a release, `cmdr unlink && cmdr pin <tag> && cmdr pull` moves this
project onto it.

## What came from where

This is a rewrite of an earlier non-commander project
(`~/pico2/pico_breadboard_kit`), which drove the same board with LVGL and a pile
of vendor demo code. What carried over is the hardware knowledge — the ST7796
init path, the GT911 register map, the WS2812 PIO program, the kit's pinout — and
what changed is where it lives: reusable drivers went into the framework as
modules, so the next ST7796 or GT911 project doesn't start from a vendor demo.

Two deliberate departures from the original:

- **Buttons are polled and debounced by time, not by edge interrupts.** The
  original used edge IRQs with a 2 ms guard and printed from inside the ISR,
  which is a reliable way to collect phantom presses.
- **Nothing blocks.** The original bit-banged the buzzer in a busy loop with the
  scheduler running. Tones and melodies here are advanced from `tick()`, so the
  shell stays live while a melody plays.
