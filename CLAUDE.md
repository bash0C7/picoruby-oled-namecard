# Development Guide for oled

Practical development conventions for this PicoRuby + picotorokko project. Read this before touching the build pipeline; there are several non-obvious gotchas.

---

## Project-Specific Gotchas (read first)

### 1. Environment config file

The real `.picoruby-env.yml` that ptrk reads lives under `.ptrk_env/.picoruby-env.yml`, **not** the project root. Any empty `.picoruby-env.yml` at the repo root is vestigial and can be removed.

### 2. `ptrk device build` does not re-sync `storage/home/`

`bundle exec ptrk device build` only copies `storage/home/` into the build tree on the **first** build (when `.ptrk_build/<env>/R2P2-ESP32/` does not yet exist). Subsequent builds reuse the copied tree, so file additions, renames, or edits under `storage/home/` or `mrbgems/applib/` **will not** be picked up.

When you modify application sources, always run:

```bash
bundle exec ptrk device prepare   # rm_rf build_path then re-copy
bundle exec ptrk device build
bundle exec ptrk device flash
```

Implementation reference: `vendor/bundle/ruby/3.4.0/bundler/gems/picotorokko-*/lib/picotorokko/commands/device.rb`, method `prepare_build_environment`, guarded by `unless Dir.exist?(build_path)` — `copy_storage_home` only runs on first build. `prepare` bypasses the guard by wiping the directory first.

### 3. Flash baudrate is hard-coded

The R2P2-ESP32 `Rakefile` contains `export ESPBAUD=150000`. The shell-provided `ESP_BAUD` / `ESPBAUD` is overridden. Changing baudrate requires editing the Rakefile and persisting via `bundle exec ptrk patch export`. In practice 150000 is fine.

### 4. Entry point is `app.rb`, not `main.rb`

R2P2-ESP32 auto-runs `storage/home/app.rb` (or a pre-compiled `app.mrb`) at boot. Historical ptrk template snippets that reference `main.rb` are inaccurate for this project.

### 5. picoruby-ws2812 (ksbmyk fork) v2 API

```ruby
# removed (raises "uninitialized constant RMTDriver" at runtime)
led = WS2812.new(RMTDriver.new(27))
led.show_hex(*colors)

# current
led = WS2812.new(pin: 27, num: 25)
colors.each_with_index { |hex, i| led.set_hex(i, hex) }
led.show
```

Per-LED setters: `set_rgb(i, r, g, b)`, `set_hex(i, hex)`, `set_hsb(i, h, s, b)`. Utilities: `fill(r, g, b)`, `clear`, `brightness=` (0-100, default 5), `close`. Commit buffer with `show`. `RMTDriver` is no longer exposed from the gem; `RMT` is instantiated internally.

### 6. picoruby-sh1107 scope is minimal

`SH1107.new(i2c, addr=0x3C)` + `draw_text(page, col, text)` (5×8 ASCII only). OLED init and arbitrary bitmap transfer (e.g. QR) are done with raw `i2c.write` inside `Applib.qr`. Don't expect the gem to handle initialization or bitmap drawing.

### 7. Ruby native extension mismatch

If you have both rbenv (`~/.rbenv/versions/3.4.1`) and mise (`~/.local/share/mise/installs/ruby/3.4.1`), `bundle install` compiles native extensions against whichever `libruby.dylib` is active. Switching active Ruby later causes `bundle exec ptrk` to fail with `LoadError: linked to incompatible libruby`. Fix: `bundle pristine` under the target Ruby.

This project targets **rbenv 3.4.1**, pinned via `/Users/bash/src/.ruby-version`.

### 8. `mrbgems/applib` vs `Mrbgemfile` name

`Mrbgemfile` declares `conf.gem "mrbgems/app"` but the directory on disk is `mrbgems/applib`. This works in the current R2P2-ESP32 build because of how mrbgem paths resolve, but the mismatch is fragile. See `TODO.md`.

---

## Verified Workflow

```bash
# One-time setup
bundle config set --local path vendor/bundle
bundle install
bundle pristine   # only if libruby mismatch

# One-time env creation (or when you want a fresh snapshot)
bundle exec ptrk env set main --latest
bundle exec ptrk env show

# Per-change cycle
bundle exec ptrk device prepare   # always run after editing storage/home or mrbgems
bundle exec ptrk device build
bundle exec ptrk device flash
bundle exec ptrk device monitor   # Ctrl+] to exit

# In the R2P2 REPL, verify the device flash actually picked up your changes
>>> ls
```

---

## Hardware Map

| Component | Pin | Notes |
|---|---|---|
| I2C SDA (OLED) | 26 | M5 OLED Unit |
| I2C SCL (OLED) | 32 | M5 OLED Unit |
| I2C address | 0x3C | SH1107, 128×128 |
| WS2812 data | 27 | ATOM Matrix built-in 5×5 = 25 LEDs |
| Button | 39 | ATOM Matrix center, GPIO::IN, LOW when pressed |

`storage/home/a.rb` uses SDA=25 / SCL=21 — a different pinout for a separate wiring experiment. Don't assume pins are global; each top-level Ruby file may bring its own.

---

## Application Architecture

```
storage/home/app.rb            require 'applib'; Applib.qr; Applib.icon
  └─ mrbgems/applib/mrblib/applib.rb
       ├─ Button class (debounced edge detection on GPIO::IN)
       ├─ Applib.qr    # I2C OLED init + QR bitmap transfer (62×62 embedded hex array)
       └─ Applib.icon  # WS2812 pattern animation + button-press dimming
```

`Applib.icon` runs an infinite `loop` with pattern cycling and button polling. To exit cleanly you power-cycle the device.

---

## Build Artifacts

- `.ptrk_build/<env>/R2P2-ESP32/build/R2P2-ESP32.bin` — firmware (~1.4 MB; partition is 2 MB, so ~30% free)
- `.ptrk_build/<env>/R2P2-ESP32/build/storage.bin` — littlefs image from `storage/home/`, flashed at offset `0x210000`

Device used in verification: `/dev/cu.usbserial-815A096BF0`, ESP32-PICO-D4 rev 1.1.

---

## PicoRuby Compatibility (from global rules)

PicoRuby is not a full Ruby. Avoid these when writing device-side code under `mrbgems/applib/` or `storage/home/`:

- `defined?`
- `Hash#fetch`
- `String#reverse`, `String#rjust`
- Inline `rescue`
- `proc`, `lambda` (in some PicoRuby builds — verify with a smoke test on device if unsure)

Use explicit `nil` checks and basic `while` / `each` constructs. Double-quoted strings are the convention. Japanese comments are OK for device-specific notes.

---

## Memory Optimization (PicoRuby on ESP32)

Target has limited RAM. Keep:

- Avoid building large intermediate arrays; iterate incrementally
- Reuse string literals and buffer objects across loop iterations
- Avoid allocating in hot loops (e.g. reuse a `buffer = Array.new(n)` outside `loop do`)

The current `Applib.icon` follows these — note the reuse of `buffer` and `bash_pattern` across the animation loop.

---

## Testing

`test/` contains Picotest-style specs, runnable via the R2P2 test harness. This project hasn't exercised the test flow recently; treat with caution.

---

## RuboCop

```bash
bundle exec rubocop        # check
bundle exec rubocop -A     # auto-fix
```

Config tweaked for PicoRuby: stricter method length, smaller class size, double-quoted strings.

---

## Operational Conventions

- Commits: conventional commits, English only. Always include `.claude/`, `mrbgems/`, and `patch/` in diffs if modified.
- Don't delete unknown directories — ptrk creates `.ptrk_env/<timestamp>/` and `.ptrk_build/<timestamp>/` on demand; investigate before rm.
- `bundle exec ptrk *` is the only supported entry; don't bypass to `cd .ptrk_build/.../R2P2-ESP32 && rake *` unless you know you need storage-only flash or Rakefile debugging.
- Before claiming "it works", verify on device via monitor + `ls` in the R2P2 REPL.

---

## Related Source Code

- picotorokko (this project's build tool): `vendor/bundle/ruby/3.4.0/bundler/gems/picotorokko-*/`
  - CLI: `lib/picotorokko/cli.rb`
  - Device commands: `lib/picotorokko/commands/device.rb`
  - Env commands: `lib/picotorokko/commands/env.rb`
- R2P2-ESP32 (firmware): `.ptrk_build/<env>/R2P2-ESP32/` (checked out during env setup)
  - `Rakefile` — ESP-IDF + build + flash + flash_storage orchestration
  - `main/` — C entry point
  - `components/picoruby-esp32/` — mrbgems glue
