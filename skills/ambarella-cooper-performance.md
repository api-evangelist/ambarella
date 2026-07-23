---
name: cooper-performance
description: >-
  Use for live CPU usage, memory usage, system load, DRAM used, bandwidth, and runtime
  performance on Cooper/Ambarella devices—including "this device", "the device",
  "the board", or "device" without saying Cooper. Collects standard JSON. Use when
  the user asks how loaded the device is, performance right now, DRAM or VP
  bandwidth usage, cavalry/CV memory, a named process performance - e.g. What is test_nnctrl performance on this Cooper device?; measure loading and bandwidth while running a pipeline, a pipeline
  loading/bandwidth, pipeline performance, or how much X is using.
  Never pipe collector stdout, use shell redirects, tee, or
  hand-write JSON for persistence.
compatibility: Cooper/Ambarella on-device execution with bash.
license: Apache-2.0
---

# Cooper Performance — Live Metrics / Per-Process / Pipeline

Collect **runtime** performance on a Cooper device and return standard JSON for the agent to summarize.

## Prerequisites

- **Cooper context:** per **cooper-system** conventions — do not duplicate env/arch detection ([cooper-system](../cooper-system/SKILL.md)).
- **This skill:** run `./scripts/run.sh`, `./scripts/process.sh`, or `./scripts/merge_pipeline.sh` from the `cooper-performance/` directory.
- **Latency:** CPU ~0.5s sample window; bandwidth ~1s.

## Agent workflow

1. **Verify context** — per **cooper-system** (on-device, arch resolvable).
2. **`cd` to `cooper-performance/`**.
3. **Open the reference for the question type** (below)—run collectors and interpret JSON only from that file.
4. **Persist when asked** — pass `-o <file>` to the collector; never pipe stdout, `>`, `tee`, or hand-author JSON.
5. **Summarize** — tables or bullets from parsed JSON.

## Route to the right reference

| User asks about | Read and follow |
|-----------------|-----------------|
| CPU, load, DRAM, bandwidth, cavalry (CV) memory, device performance | [system-performance.md](references/system-performance.md) |
| A named process (CPU, memory, NVP) | [process-performance.md](references/process-performance.md) |
| YOLOX pipeline loading/bandwidth; test/measure loading and bandwidth while running a pipeline; any pipeline program load and bandwidth | [pipeline-performance.md](references/pipeline-performance.md) |

Do not load all references—pick one row, then read that file only.

## Related skills

- **cooper-system** — static specs; env/arch/on-device conventions ([SKILL.md](../cooper-system/SKILL.md))
- **cooper-pipeline** — generate, build, and run `{name}_pipeline` binaries; session pid/log for readiness ([SKILL.md](../cooper-pipeline/SKILL.md))
- **cooper-sdk** — installed `test_*` demos to correlate with process metrics ([SKILL.md](../cooper-sdk/SKILL.md))
