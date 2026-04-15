# TODO — Outstanding Concerns

Open items that didn't fit README.md or CLAUDE.md but should be addressed if this project is revived. Ordered roughly by impact.

## 1. `.gitignore` is missing `.ptrk_build/`

Current ignore file has `build/` but the actual build directory is `.ptrk_build/<env>/`. It happens to not show up in `git status` because the directory was created after the initial gitignore was written, and `build/` pattern doesn't match. If someone runs `ptrk device build` from a clean checkout the tree stays clean for other reasons, but the ignore file should explicitly list:

```
.ptrk_build/
```

Not touched during the working session per the "no preemptive fixes" rule (we had no concrete failure to justify the edit).

## 2. `Mrbgemfile` gem path doesn't match directory name

`Mrbgemfile`:

```ruby
conf.gem "mrbgems/app"
```

Actual directory:

```
mrbgems/applib/
```

Build completes and runtime works because the mrbgem system tolerates the mismatch in this particular configuration. But it's brittle — a future ptrk / picoruby version may start matching on directory basename. Two possible fixes:

- Rename `mrbgems/applib/` → `mrbgems/app/`, or
- Update `Mrbgemfile` to `conf.gem "mrbgems/applib"`

Pick whichever matches the author's mental model of "app" vs "applib".

## 3. `sh1107` gem is under-used

`mrbgems/applib/mrblib/applib.rb` does `require 'sh1107'` and calls `SH1107.new(i2c, 0x3C); display.draw_text(...)` for three text rows, then drops back to raw `i2c.write(OLED_ADDR, 0x00, ...)` for OLED init and the 62×62 QR bitmap. Could be refactored if the upstream gem grows:

- Raw OLED init sequence (lines 86-91 of `applib.rb`) should move into `SH1107.initialize` ideally
- Bitmap transfer (QR rendering loop) could become a `display.draw_bitmap(page, col, data, width)` method in the gem

File a feature request on `ksbmyk/picoruby-sh1107` if you care.

## 4. Flash baudrate override path

Current state: Rakefile hard-codes `export ESPBAUD=150000`, ignoring shell env. If a future user needs different baud (e.g. 115200 for a flaky USB bridge), they need to:

1. Edit `.ptrk_build/<env>/R2P2-ESP32/Rakefile`, change the export line
2. Run `bundle exec ptrk patch export` to persist into `patch/R2P2-ESP32/`
3. Re-run `ptrk device prepare` + `build` on new env setups

Consider adding a wrapper option to `ptrk device flash --baud <n>` upstream in picotorokko instead. Filed-in-spirit; no actual issue link.

## 5. `ptrk device build` storage-sync silent stale bug

The root-cause fix is upstream in picotorokko: `lib/picotorokko/commands/device.rb` should re-run `copy_storage_home` (or a mtime-aware subset) on every `build` invocation, not only when `build_path` doesn't exist. Currently it only runs once and `ptrk device prepare` is the only escape hatch. A PR to `bash0C7/picotorokko` to:

- Either always re-copy `storage/home/` and `mrbgems/` on `build`
- Or add a `ptrk device build --sync` flag

would save future frustration.

## 6. `.picoruby-env.yml` vestigial root file

During the working session a zero-byte `.picoruby-env.yml` existed at the repo root and was proven to have no effect (renamed to `.bak`, then deleted, build still worked). It was removed, but consider sending a PR to picotorokko's `ptrk new` template to stop creating it at all.

## 7. CLAUDE.md still references some ptrk template boilerplate

The auto-generated ptrk template leaks some commands that don't exist in the current picotorokko (`ptrk build setup`, `ptrk cache list`, `ptrk build clean`, `ptrk env patch_export`). The working CLAUDE.md was largely rewritten but be alert for dangling references if the template leaks back in via `ptrk new --regenerate`.

## 8. `storage/home/a.rb` purpose is unclear

`a.rb` is a 4.4 KB standalone QR/OLED script with a different pinout (SDA=25, SCL=21) and no `require 'applib'`. It's not auto-loaded. Possibilities:

- A scratch/experiment file kept around as a reference
- A manually-invoked alt flow (`load "a.rb"` in the REPL)

If it's dead code, delete; if it's canonical for a second hardware config, add a header comment explaining.

## 9. Picotest coverage is zero

`test/app_test.rb` exists but hasn't been exercised during this session. The picoruby-side test harness is finicky; if you revive development, run the tests once and capture any failures here before shipping new features.

## 10. No CI

Consider a GitHub Action that runs at minimum:

- `bundle install`
- `bundle exec ptrk env set main --latest`
- `bundle exec ptrk device build`

to catch breakage from upstream picoruby / picoruby-esp32 / R2P2-ESP32 drift. Flash/monitor can't be automated without dedicated hardware.

---

*Captured at the close of the 2026-04-16 working session when the project was handed off. Hardware: ESP32-PICO-D4 + M5 OLED Unit + M5 ATOM Matrix. All items above are follow-ups, not blockers — the name card was verified working.*
