---
name: cooper-system
description: >-
  Gathers Cooper device hardware and system specifications as JSON, and resolves
  AMBARELLA_ARCH / chipset / BSP via cooper_env.sh. Other Cooper skills use
  cooper_env.sh for on-device detection and arch lookup. Use whenever the user
  asks about Cooper system info, device specs, CPU / DRAM / disk / network info on this device, VP / cavalry
  status on this device, SDK version or OS version on device, Ambarella hardware, dram / memory capacity,
  how much dram, AMBARELLA_ARCH, or chipset—including "this device", "the device", "the board", "this board" or
  "device" without naming Cooper or cooper-system even
  if they do not say "cooper-system" explicitly.
compatibility: >-
  Run on a Cooper/Ambarella device. Requires bash, /proc/ambarella, and
  commands such as cavalry_top, ip, mount, and df. Optional jq improves
  per-field extraction.
license: Apache-2.0
---

# Cooper System - Hardware / System Information

Collect hardware and system information from a Cooper device and print **standard JSON**.

## Prerequisites

- Execute on a **Cooper device** (Ambarella BSP with `/proc/ambarella/`).
- Requires **bash** (`run.sh` uses `set -euo pipefail`).
- Typical dependencies: `lscpu` or `/proc/cpuinfo`, `nproc`, `cavalry_top`, `ip`, `mount`, `df`, `/proc/ambarella/clock`, `/proc/ambarella/cavalry_cma`.
- Arch/BSP resolution: `scripts/cooper_env.sh` (env → `/etc/profile.d/ambarella_cooper_env.sh` → `/etc/ambarella.conf`).
- Optional: `jq` (cleaner `--get` for nested objects).
- Run system reports from `cooper-system/scripts/`.

## Agent workflow

1. **Resolve Cooper env** — use `scripts/cooper_env.sh` (see below); other Cooper skills must use this script, not inline `grep`.
2. `cd` to `cooper-system/scripts/` for system reports.
3. Run `./run.sh` for the full system report, or `./run.sh --get <field>` for one top-level section.
4. Use `-o <file>` to also save a copy; output is still printed to stdout.

## `cooper_env.sh` — arch, chipset, device detection

**Canonical entry** for `AMBARELLA_ARCH`, chipset, BSP, and on-device checks. Sibling skills call this script instead of duplicating config parsing.

Resolution order: shell env → `/etc/profile.d/ambarella_cooper_env.sh` → `/etc/ambarella.conf`.

```bash
COOPER_SYSTEM="${SKILLS_ROOT}/cooper-system"   # cooper_skills repo root + /cooper-system

# Export before cmake / pkg-config (cooper-pipeline)
eval "$("${COOPER_SYSTEM}/scripts/cooper_env.sh" --export)"

# Chipset for model garden (--chipset uses hyphens)
"${COOPER_SYSTEM}/scripts/cooper_env.sh" --chipset

# Local vs SSH routing (cooper-remote-device)
"${COOPER_SYSTEM}/scripts/cooper_env.sh" --is-device

# Raw arch / BSP / JSON
"${COOPER_SYSTEM}/scripts/cooper_env.sh" --arch
"${COOPER_SYSTEM}/scripts/cooper_env.sh" --bsp
"${COOPER_SYSTEM}/scripts/cooper_env.sh" --hardware-json
```

| Option | Output | Exit |
|--------|--------|------|
| `--arch` | `AMBARELLA_ARCH` (e.g. `n1_655`) | 0 if found, 1 if not |
| `--chipset` | Normalized chipset (`_` → `-`, e.g. `n1-655`) | 0 if found, 1 if not |
| `--bsp` | `SYS_BOARD_BSP` | 0 if found, 1 if not |
| `--is-device` | `on-device` or `not-on-device` | 0 when arch resolvable |
| `--export` | `export AMBARELLA_ARCH=…` (+ BSP when available) | 0 if arch found |
| `--hardware-json` | JSON: `arch`, `chip`, `chipset`, `evk`, `on_device` | 0 when arch resolvable |

Do **not** guess chipset when all sources are empty.

**Consumers:** on-device skills (e.g. `cooper-peripheral`, `cooper-video-stream`,
`cooper-pipeline`) link here for env/arch resolution; SDK binary checks use
**cooper-sdk** instead.

## CLI reference (`run.sh`)

| Option | Description |
|--------|-------------|
| *(none)* | Full system JSON (same as `--get system`) |
| `--get system` | Full system JSON |
| `--get <field>` | Single top-level section, wrapped as `{"<field>": ...}` |
| `-o`, `--output <file>` | Also write the same output to `<file>` (stdout unchanged) |
| `-h`, `--help` | Show help |

```bash
./run.sh [--get <field>] [-o <file>]
```

### Supported `--get` fields

| Field | Content |
|-------|---------|
| `cpu` | Model, core count, Cortex clock from `/proc/ambarella/clock` |
| `dram` | Size, bandwidth hint, DDR frequency (`fequency` key in JSON—historical typo) |
| `disk` | Array of mounted ext/btrfs filesystems with size/usage |
| `vp` | Cavalry memory summary plus per-device rows from `cavalry_top` |
| `network` | Up interfaces (non-lo): type, speed, IP |
| `os` | OS version, kernel version and build date from `/proc/version` |
| `sdk_version` | SDK version from `test_encode` |
| `username` | Current user |
| `hardware` | `AMBARELLA_ARCH` (`chip`) and `SYS_BOARD_BSP` (`evk`) via `cooper_env.sh` |
| `system` | Alias for full report (not wrapped in an extra key) |

Only **top-level** keys are supported; nested paths like `cpu.cores` are rejected.

## Examples

Full system report:

```bash
./run.sh
```

Same as explicit `system`:

```bash
./run.sh --get system
```

CPU only (wrapped):

```bash
./run.sh --get cpu
```

DRAM and save to file (stdout + file):

```bash
./run.sh --get dram -o dram.json
```

Network interfaces:

```bash
./run.sh --get network
```

Hardware chip and EVK board:

```bash
./run.sh --get hardware -o hardware.json
```

## Output format

- Valid JSON with quoted keys.
- Full report includes top-level objects: `cpu`, `dram`, `disk`, `vp`, `network`, `os`, `sdk_version`, `username`, `hardware`.
- `--get <field>` (except `system`) wraps the section: `{"cpu": { ... }}`, `{"network": [ ... ]}`, etc.
- Known quirk: DRAM frequency is emitted under `"fequency"` (typo preserved for compatibility).

Example fragment:

```json
{
  "cpu": {
    "arch": "Cortex-A53",
    "cores": 4,
    "frequency": "1200.0MHz"
  },
  "hardware": {
    "chip": "cv75",
    "evk": "cv75_ptpd"
  }
}
```

When summarizing for the user, prefer tables or bullet lists parsed from this JSON.
