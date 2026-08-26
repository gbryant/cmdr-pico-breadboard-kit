# Hardware bring-up checklist

Nothing in this project has met the real board yet. Everything here builds for
RP2350 and the driver logic is covered by host tests against a recording HAL
(`commander/tests/fakes`), which pins down the *logic* — register sequences,
address-window arithmetic, coordinate mapping, debounce timing, note parsing.
What it cannot tell you is whether a wire is where the datasheet says it is.

So this list is ordered by **what a failure would tell you**: each step assumes
the ones above it worked, and each has a "if it fails" line naming the most
likely cause. Work down it once and the board is characterized.

Record results as you go — tick the boxes, and note anything that needed a change
so the defaults in `cmdr.toml` can be corrected.

---

## 0. Console — before anything else

```bash
./bum          # or ./swd-flash if the probe is wired
```

- [ ] The board enumerates and `./monitor` shows the greeting.
- [ ] `help` lists: `help`, `version`, `lcd`, `touch`, `joy`, `btn`, `led`,
      `buzz`, `wled`, `wifi`, `i2c`, `kit`, `bootsel`.
- [ ] `help` does **not** print a dropped-command warning.
- [ ] The greeting is not preceded by `[warn] too many tickers`.

*If the console is silent:* USB CDC needs a moment after enumeration — the
transport waits 1.5 s before printing. If it stays silent, it's the build/flash
path, not this project.

*If a command is missing:* `MAX_COMMANDS` is too small — raise it in
`CMakeLists.txt`. The registry warns rather than dropping silently, so `help`
will say so.

## 1. The cheap outputs — LEDs and buzzer

These need no bus, so they isolate "is the firmware running at all" from "is a
peripheral mis-wired".

- [ ] `led 0 on` / `led 1 on` light D1 / D2. `led all off` clears them.
- [ ] `led 0 blink 250` blinks without blocking the shell (you can still type).
- [ ] `buzz 1000 200` makes a tone.
- [ ] `buzz play c4:200,e4:200,g4:400` plays three rising notes, and the console
      stays responsive during playback.

*If an LED is inverted* (lit when "off"): set `active_high = "no"` in
`cmdr.toml` under `[module.leds]`, then `cmdr regen`.

*If the buzzer is silent but the pin toggles:* this is a passive piezo driven by
PWM. An active buzzer with its own oscillator wants a steady level instead —
that's a different module, not a tuning change.

## 2. The RGB LED — first PIO check

- [ ] `wled 40 0 0` → red, `wled 0 40 0` → green, `wled 0 0 40` → blue.
- [ ] `wled test` cycles all three.
- [ ] `wled off` clears it.

*If the colours are permuted* (red shows green): the chip's wire order isn't GRB.
Set `order` in `[module.ws2812]` — the module supports all six permutations.

*If nothing lights and `wled` reports "no free PIO state machine":* the IR module
also claims one; nothing else here should.

## 3. The panel — the big one

```
lcd            → panel info: size, rotation, SPI bus/speed, pins
lcd test       → colour bars, a 1px white border, text at three scales
```

- [ ] `lcd test` draws eight colour bars across the top half.
- [ ] The white border reaches all four edges — a missing edge line means the
      address window is off by one, not a wiring fault.
- [ ] Text at scale 1, 2 and 3 is legible and not mirrored.
- [ ] `lcd fill red` / `lcd clear` paint the whole screen.
- [ ] `lcd rotate 1` gives a 480×320 landscape image, and `lcd test` still
      reaches all four edges in that rotation.

*If the screen stays white/black:* check CS/DC/RST before suspecting the init
sequence — the driver sends a datasheet ST7796S bring-up. If the wiring is
confirmed and it still won't come up, rebuild with `-DST7796_LEGACY_INIT`: that
switches to the vendor's original table, which is known to have driven this exact
panel. If the legacy table works and the datasheet one doesn't, say so in the
module — that's a real finding about this panel variant.

*If colours are inverted:* `lcd invert off` (this panel ships needing INVON, so
the default is on). Make it permanent with `invert = false` in `cmdr.toml`.

*If the image is mirrored or upside down:* that's the MADCTL table; the four
rotations are `0x48, 0x28, 0x88, 0xE8`. Note which rotation looks right and set
it as the default.

*If drawing is visibly slow:* the SPI clock is 40 MHz by default. The original
vendor code asked for 62.5 MHz; the panel may take it. `hz` in `cmdr.toml`.

## 4. Touch — the fiddly one

```
i2c scan       → the GT911 should ACK at 0x5D (or 0x14)
touch info     → product id, panel resolution, current mapping
touch          → one reading
touch watch    → live stream; ctrl-c or `touch stop` to end
```

- [ ] `i2c scan` finds the controller.
- [ ] `touch info` reports a product id (usually `911…`) and a sane resolution.
- [ ] `touch watch` reports a coordinate when you press, `release` when you let go.
- [ ] **Corners map correctly**: pressing the top-left of the glass gives a
      coordinate near `0,0`; bottom-right near `319,479` (in rotation 0).

*If `i2c scan` finds nothing:* the GT911's address depends on the INT pin level
at reset — try `addr = 0x14`. If the bus is silent entirely, check GP8/GP9.

*If the corners are wrong,* fix it with `touch flip` while watching the stream,
then record the answer in `cmdr.toml`:

| Symptom | Fix |
|---------|-----|
| left/right mirrored | `touch flip x` |
| top/bottom mirrored | `touch flip y` |
| axes exchanged | `touch flip swap` |
| rotated 90° from the image | `touch rotate <n>` to match the display |

The mapping arithmetic is host-tested at all four rotations, so a mismatch here
is a mounting/orientation fact about the panel, not a bug — record it.

## 5. Joystick

```
joy raw        → raw ADC counts and the measured centre
joy            → normalized position and direction
joy watch      → stream direction changes
```

- [ ] At rest, `joy` reads near `x=0 y=0` and direction `center`.
- [ ] Pushing each way reports `left` / `right` / `up` / `down`, and the
      magnitude reaches roughly ±1000 at full deflection.
- [ ] `joy cal` (stick centred) re-centres it if the rest position has drifted.

*If a direction is inverted:* the axis wiring runs the other way. The module
takes `invertX` / `invertY` constructor flags — add them to the emitter defaults
if this kit needs them.

*If directions swap (up gives left):* X and Y are exchanged — swap the `x` and
`y` pins in `cmdr.toml`.

*If it reads jittery near centre:* raise `deadzone` (percent of travel, 30 by
default), or `joy deadzone 40` to try a value live.

## 6. The app

- [ ] Boot: splash card appears, boot melody plays, RGB LED goes blue.
- [ ] Buttons toggle their LEDs and click.
- [ ] Stick left/right cycles Status → Piano → Splash.
- [ ] On Status: uptime advances once per second and the "last" row names the
      most recent input.
- [ ] `kit piano`, then touching the panel plays the key under your finger, with
      the RGB LED following.
- [ ] Stick up/down on Piano transposes an octave; on Status it changes the
      backlight (or toggles the display, on this kit's hard-wired backlight).

## 7. WiFi and telnet

- [ ] The status screen shows an IP within ~15 s of boot.
- [ ] `wifi status` agrees with it.
- [ ] `telnet <ip>` (or the hostname) gives the same shell, and `touch watch`
      streams over telnet as it does over serial.

## 8. Long-run sanity

- [ ] Leave it running for an hour on the Status screen. Uptime keeps counting,
      the console still responds, and telnet still connects.
- [ ] Hold a touch down for several seconds — the stream should not wedge, and
      releasing should report `release`.

---

## After the pass

Things worth doing once the board is characterized:

- Correct any defaults in `cmdr.toml` and in commander's `MODULE_SPECS` so the
  next scaffold of this kit starts from the values that actually worked.
- Note the outcome in commander's `PLAN.md` (the modules are marked
  "untested on hardware" there).
- If the datasheet init sequence needed the legacy fallback, or the panel wanted
  a different SPI speed, record it in the module header — those are the facts
  that are expensive to rediscover.
