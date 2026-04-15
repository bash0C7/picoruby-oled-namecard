# oled

A PicoRuby application for ESP32 development using the `picotorokko` (ptrk) build system.

**Created**: 2025-12-06 21:14:58
**Author**: bash0C7

## Quick Start

### 1. Setup Environment

First, fetch the latest repository versions automatically:

```bash
ptrk env set --latest
```

Or, create an environment with specific repository commits:

```bash
ptrk env set main --commit <R2P2-ESP32-hash>
```

Optionally, specify different commits for picoruby-esp32 and picoruby:

```bash
ptrk env set main \
  --commit <R2P2-hash> \
  --esp32-commit <picoruby-esp32-hash> \
  --picoruby-commit <picoruby-hash>
```

### 2. Build Application

```bash
bundle exec ptrk device build
```

This clones repositories, applies patches, and builds firmware for your application.

> **Important — `storage/home/` sync gotcha**
> `ptrk device build` does **not** re-sync `storage/home/` into the build tree when the build directory already exists. If you add/rename/modify files under `storage/home/` or `mrbgems/applib/`, run `ptrk device prepare` first so the build tree is rebuilt from scratch:
>
> ```bash
> bundle exec ptrk device prepare
> bundle exec ptrk device build
> ```

### 3. Flash to Device

```bash
bundle exec ptrk device flash
```

The flash baudrate is hard-coded to `150000` inside the R2P2-ESP32 Rakefile (`export ESPBAUD=150000`). Setting `ESP_BAUD` / `ESPBAUD` from the shell has no effect unless the Rakefile is patched.

### 4. Monitor Serial Output

```bash
bundle exec ptrk device monitor
```

## Application Entry Point

R2P2-ESP32 auto-runs `storage/home/app.rb` (or a compiled `app.mrb`) at boot. The current entry point is:

```ruby
# storage/home/app.rb
require 'applib'
Applib.qr
Applib.icon
```

Application logic lives in `mrbgems/applib/mrblib/applib.rb`, declared in `Mrbgemfile`.

## Project Structure

- **`storage/home/`** — Your PicoRuby application code (git-managed)
- **`patch/`** — Customizations to R2P2-ESP32 and dependencies (git-managed)
- **`.cache/`** — Immutable repository snapshots (git-ignored)
- **`build/`** — Active build working directory (git-ignored)
- **`.ptrk_env/`** — Environment metadata (git-ignored)

## Documentation

- **`SPEC.md`** — Complete specification of ptrk commands (in picotorokko gem)
- **`CLAUDE.md`** — Development guidelines and conventions
- **[picotorokko README](https://github.com/picoruby/picotorokko)** — Gem documentation and examples

## Common Tasks

### List Defined Environments

```bash
ptrk env list
```

### Show Current Environment Details

```bash
ptrk env show main
```

### Export Changes as Patches

After editing files in `build/current/`, export changes:

```bash
ptrk env patch_export main
```

Then commit:

```bash
git add patch/ storage/home/
git commit -m "Update patches and application code"
```

### Switch Between Environments

First, create the new environment:

```bash
ptrk env set development --commit <hash>
```

Then, rebuild with the new environment:

```bash
ptrk device build
```

## Troubleshooting

For detailed troubleshooting and advanced usage, see the picotorokko gem documentation.

### Environment Not Found

Check available environments:

```bash
ptrk env list
```

Create a new one:

```bash
ptrk env set myenv --commit <hash>
```

### Build Fails

Try rebuilding from scratch:

```bash
ptrk device build
```

If the issue persists, verify the environment is correctly set:

```bash
ptrk env show main
```

## Support

For issues with the picotorokko gem, see:
- GitHub: https://github.com/picoruby/picotorokko/issues
- Documentation: https://github.com/picoruby/picotorokko#readme

For PicoRuby and R2P2-ESP32 issues, see:
- PicoRuby: https://github.com/picoruby/picoruby
- R2P2-ESP32: https://github.com/picoruby/R2P2-ESP32
