# picoruby-oled-namecard

PicoRuby + ATOM MATRIX + OLED display Namecard

**Created**: 2025-12-06 21:14:58
**Author**: bash0C7

# WIP

## Quick Start

### 1. Setup Environment

First, fetch the latest repository versions automatically:

```bash
ptrk env set --latest
```

### 2. Build Application

```bash
ptrk device prepair
```

```bash
ptrk device build
```

This clones repositories, applies patches, and builds firmware for your application.

### 3. Flash to Device

```bash
ptrk device flash
```

### 4. Monitor Serial Output

```bash
ptrk device monitor
```

## Project Structure

- **`storage/home/`** — Your PicoRuby application code (git-managed)
- **`patch/`** — Customizations to R2P2-ESP32 and dependencies (git-managed)
- **`.cache/`** — Immutable repository snapshots (git-ignored)
- **`build/`** — Active build working directory (git-ignored)
- **`.ptrk_env/`** — Environment metadata (git-ignored)

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
ptrk env patch export
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
