---
name: cooper-sdk
description: >-
  Cooper SDK overview, Cooper SDK inventory and binary availability on device. Use when the user asks
  whether a Cooper SDK binary or demo exists on device — e.g. Is test_encode
  available on Cooper device?; What demos are in the Cooper SDK?;
  List Cooper SDK tools on this board; check SDK binary availability; Cooper SDK features or
  Cooper SDK overview — even if they do not say "cooper-sdk".
compatibility: Ambarella/Cooper device
license: Apache-2.0
---

# Cooper SDK — Features / Inventory / Unit tests

Guides Cooper SDK capabilities, on-device inventory of libraries/tools/demos, and unit-test apps for CVflow inference, image capture, Cavalry memory, and NVP monitoring.

## Prerequisites

- **Cooper context:** per **cooper-system** conventions — do not duplicate env, arch, or on-device detection ([cooper-system](../cooper-system/SKILL.md)).
- **This skill:** run collectors from the `cooper-sdk/` directory when a task reference requires them.
- **On-device layout:** readable `/usr/lib64` (shared libraries) and `/usr/bin` (tools and `test_*` demos) when live inventory is expected.

## Agent workflow

Hub navigation — follow this loop when the skill is triggered or a task completes. Routing tables and example prompts live only in [references/overview.md](references/overview.md#available-demos--choose-your-next-step).

### 1. Entry (first visit)

Read [references/overview.md](references/overview.md) in full.

**First visit** — skill just triggered; no cooper-sdk task completed yet in this session:

1. Summarize overview sections (Hardware, System, CVFlow, Image).
2. **Must** output every **More details** doc reference from that file.
3. Present numbered next-step prompts from [Available demos](references/overview.md#available-demos--choose-your-next-step); ask what to do next.

Wait for the user's choice unless they already named one task clearly in the same message.

### 2. Route (one reference)

Use the user's choice and the **Agent routing** table in [Available demos](references/overview.md#available-demos--choose-your-next-step) to read **exactly one** task reference (not `overview.md`).

Follow only that reference until the current operation is complete. Do not load multiple task references at once.

### 3. Return (subsequent visit)

When the task is done (command run, question answered, or user confirms success), read [overview.md](references/overview.md) again.

**Subsequent visit** — returning after a task:

- Present **only** numbered next-step prompts from [Available demos](references/overview.md#available-demos--choose-your-next-step).
- Do **not** repeat the overview summary or **More details** doc references.

Repeat steps 2–3 until the user stops or switches to another Cooper skill.

## Rules

- Do **not** grep `/usr/bin` or parse `.so` filenames manually for inventory — use the SDK inventory CLI per [run.sh.md](references/run.sh.md).

## References

| Role | File |
|------|------|
| Hub — overview, first/subsequent visit rules, routing | [references/overview.md](references/overview.md) |
| Task docs | Routed from overview **Agent routing** table only |
