---
name: cooper-video-stream
description: >-
  Start, stop, or query Ambarella Cooper video push streaming (RTSP preview,
  VCA analysis canvas). Use when the user asks to start or stop push streaming,
  turn on/off the video stream or camera encode, or check whether streaming is
  active on Cooper.
compatibility:
  arch: aarch64
  deps: test_encode, rtsp_server, test_aaa_service
license: Apache-2.0
---

# Cooper Video Stream — Start / Stop / Query

Start, stop, or query Cooper video push streaming. Stream state must come only from the script's **query** action (see Quick start)—not `ps`, open ports, or other OS probes.

**Always use the bundled scripts** — run `scripts/start_video_stream.py` for start, stop, and query. Do not call `start_stream.sh`, `stop_stream.sh`, `test_encode`, or `rtsp_server` directly; do not write new scripts that reimplement start/stop/query or RTSP URL logic. Read results from the JSON file the script writes (`Wrote <path>` on stdout).

## Prerequisites

Run on the target Cooper device (`python3` on PATH).

Before any action, verify via upstream skills (do not duplicate their logic here):

| Check | Skill | Requirement |
|-------|-------|-------------|
| Cooper env | **[cooper-system](../cooper-system/SKILL.md)** | On-device; `AMBARELLA_ARCH` resolvable and exported |
| SDK binaries | **[cooper-sdk](../cooper-sdk/SKILL.md)** | `test_encode`, `rtsp_server`, `test_aaa_service` under `/usr/bin` |
| Runtime | this skill | `python3` available; `scripts/start_stream.sh`, `scripts/stop_stream.sh`, and `scripts/start_stream_A_1080p_stream_B_720p.lua` alongside `start_video_stream.py` |

If any check fails, stop — do not run `start_video_stream.py`.

## Quick start

```bash
python3 scripts/start_video_stream.py                 # start — default sensor
python3 scripts/start_video_stream.py --sensor imx678_mipi   # start — custom sensor
python3 scripts/start_video_stream.py --stop          # stop — no extra flags
python3 scripts/start_video_stream.py --show-video-info   # query (test_encode --show-stream-info)
```

JSON → `/tmp/cooper-video-stream-<timestamp>.json` (UTC+8). Use stdout `Wrote <path>`.

**Start** runs `scripts/start_stream.sh` (fixed preset: stream A 1920×1080, stream B 720×480). **Stop** runs `scripts/stop_stream.sh` with no extra flags.

**Optional sensor on start:** If the user names a sensor in the prompt, pass `--sensor <sensor-name>` (kernel module name only, e.g. `imx678_mipi` — no modprobe parameters). Omit `--sensor` for the script default bring-up (`max96712` + `os08a10_mipi_brg`, with internal modprobe args). If the user names **`os08a10_mipi_brg`**, treat it as the default — same dual-modprobe bring-up; omit `--sensor` or pass `--sensor os08a10_mipi_brg`; behavior is identical.

**After a successful start**, the JSON includes **`rtsp_preview`** with localhost RTSP URLs for each active stream:
- **`urls.localhost`** — always (`rtsp://127.0.0.1:554/stream1` for Display / Stream A, `…/stream2` for Inference / Stream B)

The same URLs are mirrored under `video_info.stream_a.urls` / `stream_b.urls` and `initialization.rtsp_preview`.

## Instructions

1. **Prerequisites** — cooper-system (env) + cooper-sdk (binaries) + `python3`; abort if any fail.
2. **Choose action** — start, stop, or query only. On **start**, if the user specifies a sensor **name** (from cooper-peripheral or their prompt), pass `--sensor <name>`.
3. **Run** `scripts/start_video_stream.py` (start and stop **automatically pre-check** stream state via `test_encode --show-stream-info` before acting):
   - **Start** — bare command, or `--sensor <sensor-name>` when the user named a sensor
   - **Stop** — `--stop` only, no other flags
   - **Query** — `--show-video-info` (status-only; no second pre-check needed)
4. **Read JSON** — validate `action`, then:
   - start / stop / query → `video_info.stream_a` (Display), `video_info.stream_b` (Inference), `initialization.readiness`, `command`
   - start (streaming) → `rtsp_preview` / `initialization.rtsp_preview`: **`urls.localhost`** (loopback) for each streaming encode path
5. **Respond**:
   - **User-facing — start**: If `action` is `start`, `initialization.readiness` is `streaming`, and **`command` is `null`**, the stream was **already active** — cite `initialization.summary` and localhost RTSP URLs; ask before stop-then-restart; do **not** claim start ran again. If `command` is set (start script ran) and `readiness` is `streaming`, report start success with localhost RTSP URLs. Otherwise report failure.
   - **User-facing — RTSP URLs**: Report **only** `urls.localhost` (e.g. `rtsp://127.0.0.1:554/stream1`, `rtsp://127.0.0.1:554/stream2`). Do **not** print interface-specific IPs (e.g. `end0`). Tell the user that for remote preview they should use the IP they actually use to reach the device: `rtsp://<IP>:554/stream1` (Display / Stream A) and `rtsp://<IP>:554/stream2` (Inference / Stream B).
   - **User-facing — stop**: If `action` is `stop`, `initialization.readiness` is `stopped`, and **`command` is `null`**, no stream was running — cite `initialization.summary`; do not claim stop ran. If `command` is set and `readiness` is `stopped`, report stop success. Otherwise report failure. **Stopping always runs when streaming** — pre-check never blocks stop while `stream_active` is true.
   - **User-facing — query**: one or two sentences — Display (Stream A) and Inference (Stream B) streaming yes/no; include resolution only when streaming.
   - **Called by another skill**: return structured fields or JSON path from `Wrote <path>`; parent skill reads the file — no user-facing prose.

Do not use `ps`, ports, or other OS probes for stream state. Do not run a separate `--show-video-info` before start/stop unless the user only asked for status — the script pre-checks automatically.

Query runs `test_encode --show-stream-info` and treats `State: encoding` as active; `State: idle` means stopped. Start/stop run `scripts/start_stream.sh` / `scripts/stop_stream.sh` — see [reference/fields-parameters-reference.md](reference/fields-parameters-reference.md#start_stream-output-convention).

## Examples

**Start stream (default sensor)**

```bash
python3 scripts/start_video_stream.py
```

**Start stream (user-specified sensor)**

```bash
python3 scripts/start_video_stream.py --sensor imx678_mipi
```

→ If `initialization.readiness` is `streaming` and `command` is set: start succeeded — report status and localhost RTSP URLs; hint remote preview as `rtsp://<IP>:554/stream1` / `stream2`. If a custom sensor was used, mention it from `stream.sensor`. If `readiness` is `streaming` and **`command` is `null`**: stream was already active (`initialization.summary` mentions skipped) — tell the user; do not claim start ran again.

**Stop stream**

```bash
python3 scripts/start_video_stream.py --stop
```

→ If `initialization.readiness` is `stopped` and `command` is set: stop succeeded. If `readiness` is `stopped` and **`command` is `null`**: no stream was running — cite `initialization.summary`; do not claim stop ran.

**Query status**

```bash
python3 scripts/start_video_stream.py --show-video-info
```

→ e.g. "Display (Stream A): streaming 1920x1080; Inference (Stream B): streaming 720x480." / "Display and Inference not streaming."

For pre-check flows and edge cases, see [examples/example.md](examples/example.md).

## Additional resources

- JSON schemas, field reference, script flags, pre-check table → [reference/fields-parameters-reference.md](reference/fields-parameters-reference.md)
- Detailed usage examples → [examples/example.md](examples/example.md)
