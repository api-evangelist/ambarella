---
name: ambarella-model-garden
description: >-
  Query, sync, and benchmark HuggingFace Ambarella Model Garden `.bin` models.
  Use for Ambarella Model Garden, HuggingFace model download/sync, .bin paths,
  inference time, CV memory, DRAM bandwidth, local.json I/O, or model benchmarks
  — even if the user does not say "model garden".
compatibility: >-
  Python 3 + huggingface_hub; Cooper device for --sync benchmarks
  (test_nnctrl, test_cavalry_mem).
license: Apache-2.0
---

# Ambarella Model Garden — Query, Sync & Benchmark

Query, download, and benchmark Model Garden `.bin` files per chipset. Output JSON under the configured output directory for downstream skills.

## Prerequisites

**Always use the bundled script** — do not reimplement download or benchmark logic.
**Any prerequisite failure → stop** — do not skip to Quick start or run with guessed paths.

| Operation | Environment |
|-----------|-------------|
| `--query` | Python 3, `huggingface_hub`, network; `--chipset` from cooper-system |
| `--sync` | Above + Ambarella/Cooper device; `test_nnctrl`, `test_cavalry_mem` in PATH |

```bash
python3 --version
python3 -c "import huggingface_hub"
```

**Output directory persistence** — model storage path is saved in **`.output_dir`** at the
**skill root** (same directory as this `SKILL.md`). Do not look for `.output_dir` under
workspace or cwd. Contract: [references/output-dir-contract.md](references/output-dir-contract.md).

**Cooper env** — via **[cooper-system](../cooper-system/SKILL.md)** (not inline here):

- Export `AMBARELLA_ARCH` (underscore form, e.g. `n1_655` — vp_bw paths, pkg-config).
- Obtain **chipset** for `--chipset` via cooper-system `--chipset` (e.g. `n1-655`).
- If resolution fails, stop — do not guess.

**Naming:**

- **`--chipset`**: pass cooper-system `--chipset` or `$AMBARELLA_ARCH`; `model_garden.py`
  normalizes `_` → `-` in the argument (`n1_655` → `n1-655`) to match HF files like
  `n1-655_*.bin`.
- **`--model_names`**: short names as-is (`RTMPose`); script adds `Ambarella/` prefix only —
  no `_`→`-` conversion.

**`--sync` only** — vp_bw is a **blocking gate** before any download/benchmark:
[references/vp-bw.md](references/vp-bw.md). If `/usr/share/ambarella/vp_bw/memset_split_0.dvi`
is missing and cannot be installed, stop and wait for user confirmation.

```bash
command -v test_nnctrl test_cavalry_mem
```

**Off-device:** `--query` may work with network + chipset; `--sync` download may succeed but
benchmarks need device — warn before running.

Chipset selects model files (e.g. `cv72_*.bin`). Pass `--chipset` on every invocation.

## Quick start

Command examples only — follow **Instructions** and Prerequisites first.

```bash
SKILL_ROOT=/path/to/skills/ambarella-model-garden
OUTPUT_DIR="$(cat "$SKILL_ROOT/.output_dir")"
mkdir -p "$OUTPUT_DIR"

pip install -r "$SKILL_ROOT/requirements.txt"

python "$SKILL_ROOT/scripts/model_garden.py" \
  --chipset <chipset> \
  --model_names RTMPose \
  --query --sync \
  --output_dir "$OUTPUT_DIR"
```

First run: resolve and persist output dir per [workflow.md](references/workflow.md#output-directory).
On success, stdout includes **`Output dir: /absolute/path`**.

| User intent | Flag |
|-------------|------|
| List / check download status | `--query` |
| Download + benchmarks | `--sync` |
| Both | `--query` and `--sync` together |

Required: `--chipset`, `--output_dir`, `--model_names` (short names or `all`), at least one of
`--query` / `--sync`. Names omit `Ambarella/` prefix.

## Instructions

Follow these steps in order. Full detail: **[references/workflow.md](references/workflow.md)**.

1. **Verify environment** — Prerequisites above (Python, cooper-system, `--sync` tools).
   Any failure → **stop**.
2. **vp_bw (`--sync` only)** — [references/vp-bw.md](references/vp-bw.md). Unresolved → **stop**.
3. **Resolve chipset** — cooper-system; pass `--chipset`. Unresolved → **stop**; do not guess.
4. **Resolve output directory** — per [output-dir-contract.md](references/output-dir-contract.md):
   - Read `"$SKILL_ROOT/.output_dir"` if it exists, or ask user (default `$HOME/model_garden_models`).
   - **Persist** absolute path: `echo /abs/path > "$SKILL_ROOT/.output_dir"` **before** running script.
5. **Run script** — `model_garden.py` with `--output_dir "$(cat "$SKILL_ROOT/.output_dir")"`.
6. **Hand off** — use stdout **`Output dir:`** line or `.output_dir` contents for downstream CLI.

**Never use:**

- `/tmp/model_garden_*` or other invented paths unless user specified and persisted to `.output_dir`.
- Workspace subdirectories as model root without user request and persistence.
- `--output_dir` without reading or writing skill-root `.output_dir` first.

**Output contract** — downstream skills read JSON under `--output_dir` directly
([output-schemas.md](references/output-schemas.md)):

| After | File | Key fields |
|-------|------|------------|
| `--query` | `query_result.json` | `files[].path`, `files[].inputs`, `files[].outputs`, `benchmarks[]` |
| `--sync` | `sync_result.json` | same + `added_files`, `removed_files` |
| Either | `local.json` | Registry keyed `Ambarella/ModelName`; I/O per `.bin` |
| Catalog | `remote.json` | Remote file list |

Resolve `files[].path` relative to `--output_dir` to absolute paths for downstream use.
Inference **codegen** consumers use `inputs`/`outputs` lengths and `shape` from catalog
entries; **runtime deploy** consumers use synced `.bin` paths — re-sync if missing.
Metrics live in `benchmarks[]`, not flat model fields.

**Do not:** reimplement HF download or `test_nnctrl` parsing; call `hf_hub_download` unless
diagnosing script failure; assume flat benchmark fields.

## Called by another skill

When a parent skill (e.g. pipeline deploy) needs the model root:

1. Read **`Output dir:`** from this skill's stdout after query/sync, **or**
   `cat "$SKILL_ROOT/.output_dir"`.
2. Pass that absolute path to the consumer's model-garden CLI flag unchanged.
3. Do not pass workspace paths, `/tmp/model_garden_*`, or paths not persisted in `.output_dir`.

Contract: [references/output-dir-contract.md](references/output-dir-contract.md).

## Examples

**Query one model**

```bash
SKILL_ROOT=/path/to/skills/ambarella-model-garden
OUTPUT_DIR="$(cat "$SKILL_ROOT/.output_dir")"
python "$SKILL_ROOT/scripts/model_garden.py" \
  --chipset <chipset> --model_names RTMPose --query \
  --output_dir "$OUTPUT_DIR"
```

**Sync all models for chipset**

```bash
SKILL_ROOT=/path/to/skills/ambarella-model-garden
OUTPUT_DIR="$(cat "$SKILL_ROOT/.output_dir")"
python "$SKILL_ROOT/scripts/model_garden.py" \
  --chipset <chipset> --model_names all --sync \
  --output_dir "$OUTPUT_DIR"
```

**Query then sync multiple**

```bash
SKILL_ROOT=/path/to/skills/ambarella-model-garden
OUTPUT_DIR="$(cat "$SKILL_ROOT/.output_dir")"
python "$SKILL_ROOT/scripts/model_garden.py" \
  --chipset <chipset> --model_names RTMPose LongCLIP --query --sync \
  --output_dir "$OUTPUT_DIR"
```

## Additional resources

| Topic | File |
|-------|------|
| Output dir persistence contract | [references/output-dir-contract.md](references/output-dir-contract.md) |
| Agent workflow, output dir persistence | [references/workflow.md](references/workflow.md) |
| vp_bw blocking gate | [references/vp-bw.md](references/vp-bw.md) |
| JSON field definitions | [references/output-schemas.md](references/output-schemas.md) |
| LLaVA-OneVision, LongCLIP layouts | [references/special-models.md](references/special-models.md) |
