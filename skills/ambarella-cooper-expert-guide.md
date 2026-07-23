---
name: cooper-expert-guide
description: >-
  Living knowledge base of Cooper / Ambarella skill pitfalls and verified fixes.
  Use before any multi-skill or on-device Cooper workflow, when debugging
  IAV/canvas errors, model sync or pipeline failures, OSD issues, or user
  corrections — and after a verified fix should be recorded. Read even when
  another Cooper skill is primary; do not skip because the user did not say
  "expert guide".
compatibility: Reference companion for Ambarella/Cooper on-device skills.
license: Apache-2.0
---

# Cooper Expert Guide — Pitfalls & Verified Fixes

A **living document** of verified pitfalls, fixes, and prevention notes for
skills that run on Ambarella/Cooper devices. Domain workflows stay in each
skill; this guide only captures **non-obvious errors and lessons learned**.

**Scope:** Applies to **Cooper skills** — any skill whose `compatibility` targets
device, embedded, or Cooper execution (identified by that field, not install path
or name prefix). **Read** [references/index.md](references/index.md) before multi-step
or debugging work on those skills. **After** a verified fix or user correction,
add or merge an entry so future sessions do not repeat the mistake (see
[When to write](#when-to-write)); do not block the user task when there is nothing
new to record.

## When to read

- Before starting a **multi-skill** workflow (model sync → pipeline → live run).
- When you see a **known error symptom** (path not found, IAV/canvas failure,
  benchmark missing fields, OSD misalignment).
- When the user **corrects** your approach — check whether an entry already
  exists in [references/index.md](references/index.md).

## When to write

Update an entry when **all** of the following are true:

1. The problem is **verified solved** (not a guess).
2. The lesson is **not already** documented in the relevant on-device skill
   (`SKILL.md` or its `references/`) — check those first; do not duplicate
   skill content here.
3. The lesson is **not already** fully documented in an existing expert-guide entry.
4. A future agent could **repeat the mistake** without this note.

Also write when the user explicitly asks to record a fix, or when you discover
a gap between skill docs and actual device behavior.

## How to read

1. Open [references/index.md](references/index.md).
2. Filter by **skill** or **symptom** keyword.
3. Read the linked file under `references/entries/`.
4. Follow the entry's **prevention** steps before retrying.

## How to write

1. Copy [references/entry-template.md](references/entry-template.md).
2. Create `references/entries/<topic>.md` with a short, semantic filename
   (e.g. `model-garden-vp-bw.md`).
3. Fill every field; keep **fix** steps copy-pasteable where possible.
4. Add or update a row in [references/index.md](references/index.md).
5. If an entry already exists, **merge** into it — do not duplicate topics.

## Do not

- Duplicate content that already lives in an on-device skill — fix or extend the
  skill doc instead, unless the user only wants a short cross-skill pointer.
- Duplicate full skill workflows here (link to the skill instead).
- Record unverified hypotheses or one-off environment quirks without confirmation.
- Edit other skills' main docs unless the user asks — expert-guide is for gaps
  between skills, not a second copy of skill documentation.

## Entry index

All entries: [references/index.md](references/index.md)
