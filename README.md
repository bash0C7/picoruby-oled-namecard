# oled — PicoRuby OLED + LED Name Card

A PicoRuby application for ESP32 that drives an M5Stack OLED Unit (SH1107, 128×128, I2C) and an M5Stack ATOM Matrix (25 WS2812 LEDs) as a physical name card. Built with the [picotorokko](https://github.com/picoruby/picotorokko) (ptrk) build system.

**Status**: verified working on real hardware (ESP32-PICO-D4 rev 1.1).

---

## Hardware

| Component | Part | Pins |
|---|---|---|
| MCU | ESP32-PICO-D4 (M5Stack ATOM Matrix) | — |
| OLED | M5Stack OLED Unit (SH1107, 0x3C) | SDA=26, SCL=32 (I2C, 100 kHz) |
| RGB matrix | ATOM Matrix built-in 5×5 WS2812 | GPIO 27 (25 LEDs) |
| Button | ATOM Matrix center button | GPIO 39 (GPIO::IN) |

The `a.rb` variant uses SDA=25 / SCL=21 — those are the pins for a different wiring and are not used by the current default `app.rb` flow.

---

## Quick Start (verified workflow)

All commands assume the project root and use `bundle exec`. Native extensions must be compiled against the Ruby install you actually use — see "Ruby environment notes" below if `bundle exec ptrk` fails with a `LoadError` about incompatible `libruby.dylib`.

### 1. Bundle install

```bash
bundle config set --local path vendor/bundle
bundle install
# If you switched Ruby installations (rbenv ↔ mise) and hit libruby incompatibility:
bundle pristine
```

### 2. Create environment (first time only)

```bash
bundle exec ptrk env set main --latest
bundle exec ptrk env show   # confirm "current"
```

Environment data is persisted in `.ptrk_env/.picoruby-env.yml`, **not** in a root-level `.picoruby-env.yml`.

### 3. Build + flash + monitor

When `storage/home/` or `mrbgems/applib/` has been modified, always run `prepare` first (see "Gotchas"):

```bash
bundle exec ptrk device prepare   # wipes and re-copies storage/home into build tree
bundle exec ptrk device build     # compiles firmware (R2P2-ESP32.bin + storage.bin)
bundle exec ptrk device flash     # writes both bins to the device
bundle exec ptrk device monitor   # serial terminal (Ctrl+] to exit)
```

For no-source-change reflashes, `prepare` can be skipped.

### 4. Confirm on device

In the R2P2 REPL:

```
>>> ls
a.rb
app.rb
```

Then `app.rb` auto-runs on next boot.

---

## Gotchas

### `ptrk device build` does not re-sync `storage/home/`

The build command only copies `storage/home/` into the build tree on the **first** build (when `.ptrk_build/<env>/R2P2-ESP32/` does not yet exist). Subsequent builds reuse the stale copy, so **file additions, renames, or edits are silently ignored**. Symptom: device's `ls` shows old filenames. Fix: always run `bundle exec ptrk device prepare` before `build` whenever application sources change.

Root cause: `picotorokko/lib/picotorokko/commands/device.rb` guards `prepare_build_environment` behind `unless Dir.exist?(build_path)`. `prepare` does `FileUtils.rm_rf(build_path)` and then the copy, so it always re-syncs.

### Flash baudrate is hard-coded

The R2P2-ESP32 `Rakefile` (inside `.ptrk_build/<env>/R2P2-ESP32/`) contains `export ESPBAUD=150000`. Setting `ESP_BAUD` / `ESPBAUD` from the shell has **no effect** — the Rakefile's own export wins. To change the baudrate permanently, edit that line and run `bundle exec ptrk patch export` to persist the change.

In practice 150000 works fine for this device; we accept it.

### Application entry point is `app.rb` (not `main.rb`)

R2P2-ESP32 auto-runs `storage/home/app.rb` (or a pre-compiled `app.mrb`) at boot. Older docs that mention `main.rb` are inaccurate for R2P2-ESP32. Current entry point:

```ruby
# storage/home/app.rb
require 'applib'
Applib.qr
Applib.icon
```

Application logic lives in `mrbgems/applib/mrblib/applib.rb`, declared in `Mrbgemfile`.

### picoruby-ws2812 API v2 (breaking change)

The [ksbmyk/picoruby-ws2812](https://github.com/ksbmyk/picoruby-ws2812) fork used here migrated to a keyword-argument constructor and a buffered setter API. `RMTDriver` is no longer exposed:

```ruby
# old (removed — triggers "uninitialized constant RMTDriver")
led = WS2812.new(RMTDriver.new(27))
led.show_hex(*colors)

# new
led = WS2812.new(pin: 27, num: 25)
colors.each_with_index { |hex, i| led.set_hex(i, hex) }
led.show
```

Setters: `set_rgb(i, r, g, b)`, `set_hex(i, hex)`, `set_hsb(i, h, s, b)`. Utilities: `fill(r, g, b)`, `clear`, `brightness=` (0-100, default 5), `close`. Commit the buffer with `show`.

### picoruby-sh1107 scope

The [ksbmyk/picoruby-sh1107](https://github.com/ksbmyk/picoruby-sh1107) gem provides only `SH1107.new(i2c, addr=0x3C)` + `draw_text(page, col, text)` (5×8 ASCII). OLED init commands and arbitrary bitmap transfer (e.g. QR) are still done with raw `i2c.write` in `Applib.qr`. Keep both in mind when refactoring.

### Ruby environment notes

- Target Ruby: 3.4.x (pinned via parent `/Users/bash/src/.ruby-version` to 3.4.1).
- If you have both rbenv and mise installed, native extensions (json, prism, rbs, etc.) are compiled against whichever `libruby` was active at `bundle install` time. Switching back causes `LoadError: linked to incompatible libruby`. Fix: `bundle pristine` under the target Ruby.

---

## Project Structure

```
.
├── storage/home/                 # PicoRuby app code copied onto device flash
│   ├── app.rb                    # auto-loaded entry point
│   └── a.rb                      # alternate standalone script (different pinout)
├── mrbgems/applib/               # application logic as an mrbgem
│   ├── mrbgem.rake
│   └── mrblib/applib.rb          # Applib.qr + Applib.icon + Button class
├── patch/picoruby/               # compile-time patches (build_config/xtensa-esp.rb)
├── test/                         # Picotest unit tests (picoruby-side)
├── Mrbgemfile                    # mrbgems declaration for this app
├── Gemfile / Gemfile.lock        # host-side Ruby deps (ptrk etc.)
├── .rubocop.yml                  # style config
├── .ptrk_env/                    # tracked: .gitkeep + .picoruby-env.yml
│   ├── .picoruby-env.yml         # the real env registry (NOT the root stub)
│   └── <timestamp>/              # ignored: checked-out source trees
├── .ptrk_build/                  # ignored: actual build working copy (per env)
├── .cache/                       # ignored: immutable repo snapshots
└── vendor/bundle/                # ignored: gem cache
```

Note: `.gitignore` currently only lists `build/`, but the real working directory is `.ptrk_build/`. Works in practice because `.ptrk_build/` was created after the initial ignore file; consider adding explicitly (see `TODO.md`).

---

## Common Tasks

### Environment inspection

```bash
bundle exec ptrk env list
bundle exec ptrk env show
```

### Switch to a new environment snapshot

```bash
bundle exec ptrk env set main --latest   # create a new timestamp
bundle exec ptrk device prepare          # then wipe build tree
bundle exec ptrk device build
```

### Export manual patches to `patch/`

If you edited files inside `.ptrk_build/<env>/R2P2-ESP32/` directly:

```bash
bundle exec ptrk patch export
git add patch/ storage/home/ mrbgems/
git commit -m "feat: ..."
```

### Storage-only reflash (skip firmware rebuild)

The R2P2-ESP32 `Rakefile` exposes a `:flash_storage` task that writes only `build/storage.bin` (offset `0x210000`). Requires the source-side `storage/home/` to be synced into the build tree first (i.e. run `ptrk device prepare` + `ptrk device build` before this):

```bash
cd .ptrk_build/<env>/R2P2-ESP32
rake flash_storage
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `bundle exec ptrk` → `linked to incompatible libruby.dylib` | Ruby install mismatch | `bundle pristine` |
| Device `ls` shows old filenames | storage/home not re-synced | `ptrk device prepare` before build |
| `uninitialized constant RMTDriver (NameError)` | old ws2812 API | migrate to keyword-arg constructor (see Gotchas) |
| `ptrk env list` empty | `.picoruby-env.yml` not created | `ptrk env set main --latest` |
| Flash stuck / `Connecting…` forever | USB cable or button held | swap cable; ESP32-PICO-D4 requires no button hold |

---

## Related Repositories

- [picoruby/R2P2-ESP32](https://github.com/picoruby/R2P2-ESP32) — firmware base
- [picoruby/picoruby](https://github.com/picoruby/picoruby) — language
- [bash0C7/picotorokko](https://github.com/bash0C7/picotorokko) — the `ptrk` build orchestrator (fork used here)
- [ksbmyk/picoruby-sh1107](https://github.com/ksbmyk/picoruby-sh1107) — OLED driver
- [ksbmyk/picoruby-ws2812](https://github.com/ksbmyk/picoruby-ws2812) — RGB LED driver
