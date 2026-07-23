---
name: cooper-peripheral
description: >-
  Enumerate Ambarella Cooper camera peripherals (VIN controllers, VSRC,
  connected sensors); report probe status, sensor name, and VSRC stream
  readiness. Use when the user asks to list connected cameras or sensors, check
  VIN/VSRC topology, probe camera/sensor status, detect whether a sensor is
  active or streaming at the VSRC layer, troubleshoot missing camera detection,
  or assess camera pipeline readiness on Cooper devices.
compatibility:
  arch: aarch64
  deps: test_encode
license: Apache-2.0
---

# Cooper Peripheral

Enumerate video source (VSRC) devices on VIN controllers, persist JSON, and summarize sensor readiness and next steps. This skill probes and reports only—it does not run `modprobe`, `test_aaa_service`, or `test_encode` bring-up.

## Prerequisites

Run on the target Cooper device (`python3` on PATH).

Before any action, verify via upstream skills (do not duplicate their logic here):

| Check | Skill | Requirement |
|-------|-------|-------------|
| Cooper env | **[cooper-system](../cooper-system/SKILL.md)** | On-device; `AMBARELLA_ARCH` resolvable and exported |
| SDK binary | **[cooper-sdk](../cooper-sdk/SKILL.md)** | `test_encode` present under `/usr/bin` |
| Runtime | this skill | `python3` available |

If any check fails, stop — do not run `show_vsrc_info.py`. Do not infer Cooper SDK from `test_encode` on PATH alone without cooper-system confirmation.

## Quick start

```bash
python3 scripts/show_vsrc_info.py
```

JSON → `/tmp/vsrc-info-<timestamp>.json` (UTC+8). Use stdout `Wrote <path>`.

## Instructions

1. **Prerequisites** — cooper-system (env) + cooper-sdk (`test_encode`) + `python3`; abort if any fail.
2. **Run probe** — on the target device, run `scripts/show_vsrc_info.py`.
3. **Read JSON** — load the file from stdout `Wrote <path>` or `-o`. Check **`probe_status` first**:
   - `binary_missing` / `command_failed` → use [reference/reference.md](reference/reference.md#probe-failures); do not interpret VSRC or `initialization.readiness` as camera state
   - `ok` → read `vsrc_information`, `initialization.sensors_detected`, `initialization.readiness`
4. **Respond**:
   - **User-facing**: short summary per VIN/vsrc — `sensor_name`, `status`, `stream_status`, `initialization.readiness` (and `sensors_detected` if the user asked how many cameras are connected). Omit `raw_output`, `attributes`, `recommended_steps`, and full JSON unless asked. If `probe_status` is not `ok`, explain the probe error in plain language.
   - **Called by another skill**: return JSON path from `Wrote <path>` or structured fields; parent skill reads the file — no user-facing prose.
5. **Follow-up** — if `probe_status` is `ok` and `initialization.readiness` is `sensor_active_not_streaming`, tell the user the sensor is detected but not streaming; they can start the pipeline with a follow-up request. Do **not** run `modprobe`, `test_aaa_service`, or `test_encode` bring-up from this skill.

## Examples

**List connected cameras**

```bash
python3 scripts/show_vsrc_info.py
```

→ e.g. "VIN Controller[0] vsrc[0]: os08a10, active, not streaming."

**Probe failed**

→ If `probe_status` is `binary_missing`: tell user to install SDK or pass `--test-encode /path/to/test_encode`. Do not report "no camera detected."

For readiness handling, skill-to-skill calls, and edge cases, see [examples/example.md](examples/example.md).

## Additional resources

- JSON schema, field reference, probe failures, readiness table → [reference/reference.md](reference/reference.md)
- Detailed usage examples → [examples/example.md](examples/example.md)
