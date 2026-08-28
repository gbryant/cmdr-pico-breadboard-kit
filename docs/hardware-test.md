# Hardware bring-up checklist

> **First pass: 2026-08-27.** Flashed over SWD through the RP2350-GEEK probe
> (a Waveshare RP2350-GEEK running debugprobe with a
> commander status screen). Every peripheral responded. Touch corner mapping is confirmed
> correct (step 4) and WiFi/telnet work (step 7 — the `[wifi] connect=-7` at boot
> is a retry, not a failure). Still open: the panel's border/rotation checks,
> joystick direction throws, and the soak (step 8). Don't read the ticks as more
> than they are.
>
> A note on testing method, learned the hard way here: **prompts printed by a
> host-side script are invisible until the command finishes**, so a test that
> asks for a press at a particular moment silently collects nothing. Put the
> prompt on the device's own screen, and let the test advance when the input
> arrives rather than on a stopwatch.
>
> Two firmware bugs came out of it, both in commander's Pico HAL and both
> invisible to the host tests, since the fake HAL has its own implementations:
>
> - `hal_pwm_stop()` disabled the PWM slice without settling the pin, freezing it
>   high about half the time — the buzzer stuck in a continuous tone. Presented
>   as an intermittent fault from *both* the touch and button paths, because both
>   stop through the same call.
> - `hal_i2c_write_read()` never sent a STOP when no read followed, leaving the
>   GT911's touch-acknowledge write unterminated.
>
> What made the first one findable was instrumenting the buzzer to report its own
> start/stop counts. `tones: 5, stops: 5, lost stops: 0` proved the module was
> behaving and moved the search a layer down. Worth reaching for earlier next
> time: make the suspect account for itself before reading its code again.

The driver logic is covered by host tests against a recording HAL
(`commander/tests/fakes`), which pins down register sequences, address-window
arithmetic, coordinate mapping, debounce timing and note parsing. What it cannot
tell you is whether a wire is where the datasheet says it is — or what state the
HAL leaves a peripheral in, which is where both of the first pass's bugs lived.

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

- [x] The board enumerates and `./monitor` shows the greeting.
- [x] `help` lists: `help`, `version`, `lcd`, `touch`, `joy`, `btn`, `led`,
      `buzz`, `wled`, `wifi`, `i2c`, `kit`, `bootsel`.
- [x] `help` does **not** print a dropped-command warning.
- [x] No `[warn] too many tickers` (6 tickers against a cap of 8, so a drop
      isn't reachable — but the greeting itself wasn't captured).

*If the console is silent:* USB CDC needs a moment after enumeration — the
transport waits 1.5 s before printing. If it stays silent, it's the build/flash
path, not this project.

*If a command is missing:* `MAX_COMMANDS` is too small — raise it in
`CMakeLists.txt`. The registry warns rather than dropping silently, so `help`
will say so.

## 1. The cheap outputs — LEDs and buzzer

These need no bus, so they isolate "is the firmware running at all" from "is a
peripheral mis-wired".

- [x] `led 0 on` / `led 1 on` light D1 / D2. `led all off` clears them.
- [ ] `led 0 blink 250` blinks without blocking the shell (you can still type).
- [x] `buzz 1000 200` makes a tone.
- [x] `buzz play c4:200,e4:200,g4:400` plays three rising notes, and the console
      stays responsive during playback.

*If an LED is inverted* (lit when "off"): set `active_high = "no"` in
`cmdr.toml` under `[module.leds]`, then `cmdr regen`.

*If the buzzer is silent but the pin toggles:* this is a passive piezo driven by
PWM. An active buzzer with its own oscillator wants a steady level instead —
that's a different module, not a tuning change.

## 2. The RGB LED — first PIO check

- [x] `wled 40 0 0` → red, `wled 0 40 0` → green (blue not tried).
- [ ] `wled test` cycles all three.
- [x] `wled off` clears it.

*If the colours are permuted* (red shows green): the chip's wire order isn't GRB.
Set `order` in `[module.ws2812]` — the module supports all six permutations.

*If nothing lights and `wled` reports "no free PIO state machine":* the IR module
also claims one; nothing else here should.

## 3. The panel — the big one

```
lcd            → panel info: size, rotation, SPI bus/speed, pins
lcd test       → colour bars, a 1px white border, text at three scales
```

- [x] `lcd test` draws eight colour bars across the top half.
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

- [x] `i2c scan` finds the controller — `device at 0x5d`.
- [x] `touch info` reports `product: 911`, panel `320x480`, `state: ok`.
- [x] Presses register (each one triggers the app's touch beep). Coordinate
      streaming itself not read back yet.
- [x] **Corners map correctly** (2026-08-27, rotation 0). Measured by drawing a
      target on screen and touching it, so display and touch are compared
      directly rather than against an assumed orientation:

      | pressed | reported |
      |---------|----------|
      | centre ≈ (160,240) | (140,214) |
      | top-right (285,35) | (292,27)  |
      | bottom-left (35,445) | (51,444) |

      The two diagonal opposites are what make this conclusive: an x-flip would
      put the top-right press near x=35, a y-flip near y=445, an axis swap at
      (35,285), a 180° rotation at (35,445). **No `touch flip` needed.**

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

- [x] At rest, `joy` reads `x=5 y=6`, direction `center` (raw 1964/2285 against
      a measured centre of 1951/2272).
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

- [x] The status screen shows an IP within ~15 s of boot.
- [x] `telnet <ip>` gives the same shell (confirmed 2026-08-27).
- [ ] `wifi status` cross-check.
- Note: `touch watch` and the other `watch` streams go to the **board console**,
  not to the telnet session that started them — see `modules/ConsoleOut.h` for
  why (a Writer is a stack local owned by one dispatch).

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
