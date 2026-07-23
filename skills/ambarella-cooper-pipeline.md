---
name: cooper-pipeline
description: >-
  Deploy and run distributed Cooper REST pipeline on device — scaffold copy from
  built-in recipes, build nodes, allocate ports 9000-9999, start image publisher /
  inference / display nodes, verify MJPEG and health endpoints. Consumes RTSP and
  model paths supplied by the Agent (from prerequisite on-device skills or explicit
  overrides). Supports a single workspace with multiple parallel pipelines via
  --pipeline-id. Use when the user asks to deploy a pipeline, start
  pipeline nodes, run distributed inference, open MJPEG viewer, list running pipeline
  nodes, list available pipeline recipes or combinations, what pipelines can be
  deployed, stop one pipeline node, or stop a pipeline session — even if they do not
  name this skill. Never use to edit node C++ or Python source.
compatibility: >-
  Ambarella/Cooper device; Python 3 + PyYAML. Agent runs prerequisite on-device
  skills before deploy when inputs are not overridden.
license: Apache-2.0
---

# Cooper Pipeline — Deploy & Run

Self-contained **runtime** skill: copy scaffolds from [`assets/scaffold/`](assets/scaffold/),
build under a user **workspace** (`--workspace-dir`), deploy from built-in **recipes**,
allocate ports, start nodes, query/stop per pipeline or all.

**Scripts do not call sibling skills.** The Agent runs prerequisite skills and passes contract
outputs on the CLI — see [references/deploy-input-contracts.md](references/deploy-input-contracts.md).

**Not this skill:** edit node source, OpenAPI, prerequisite skill CLIs, or **manually start
nodes** — use `deploy_pipeline.sh` only ([Node startup](#node-startup)).

**Terminology:** **Deploy inputs** = RTSP / models from prerequisite skills (this doc).
**Node `upstream`** = inter-node REST wiring in recipes (`upstream.images`, `--upstream-images-url`).
Do not conflate the two.

## Workspace layout

User specifies **`--workspace-dir`** and **`--pipeline-id`** for each pipeline.
The absolute workspace path is persisted in **`.workspace_dir`** at the **skill root**
(same directory as this `SKILL.md`). Scripts also read/write this file when
`--workspace-dir` is omitted or after successful init/deploy.

Contract: [references/workspace-dir-contract.md](references/workspace-dir-contract.md).

**Resolve workspace (Agent — follow in order):**

1. Path given in the current request → use it (overwrites `.workspace_dir` on init/deploy).
2. Read **`cat "$SKILL_ROOT/.workspace_dir"`** if file exists.
3. **First use** — ask; suggest default **`$HOME/cooper_pipelines`**.

On successful init/deploy, stdout includes **`Workspace dir: /absolute/path`**.

**Never use:**

- `/tmp/cooper_pipeline_*` or invented temp workspace unless user specified and used for deploy.
- `{pipeline_id}/` subdirectory as `--workspace-dir` (workspace is the parent root).
- `.workspace_dir` under workspace or cwd — only under **`$SKILL_ROOT`**.

```bash
SKILL_ROOT=/path/to/skills/cooper-pipeline
WS="$(cat "$SKILL_ROOT/.workspace_dir")"   # or user path / Workspace dir: stdout line
```

```text
~/cooper_pipelines/              # --workspace-dir
  pipelines.json                 # registry
  ID/                            # --pipeline-id ID (user-chosen)
    pipeline.state.json
    deploy.env
    .session.json
    logs/
    image_publisher/ …
```

**Naming:** `--workspace-dir` / `workspace_dir` = workspace root; each node has `node_dir` in
`pipeline.state.json`; C++ node build uses `build.sh --node-dir PATH` (node source root).

## Prerequisites (Agent — before deploy)

On a Cooper device. Pipeline scripts **fail fast** if inputs are missing; they do **not**
invoke prerequisite skills automatically.

| Step | Skill | Agent action |\n|------|-------|--------------|\n| 1 | cooper-system | export build environment |\n| 2 | cooper-video-stream | query; start if idle; note **`Wrote …json`** → `--video-stream-json` |\n| 3 | ambarella-model-garden | sync if needed → read **`Output dir:`** or **`cat $MG_SKILL_ROOT/.output_dir`** → `--model-garden-dir` |\n| 4 | this skill | resolve **`Workspace dir:`** or **`cat $SKILL_ROOT/.workspace_dir`** → `deploy_pipeline.sh --workspace-dir WS --video-stream-json PATH --model-garden-dir DIR --pipeline-id ID --recipe RECIPE` |\n\n**Pitfalls:**\n- **Model Discovery:** If `--model-garden-dir` is not provided or the local JSON mapping is missing, the deployment may fail. Use explicit overrides `--rtsp-url` and `--{model_type}-model` (e.g., `--yolox-model`) to bypass garden resolution when model paths are known locally (e.g., in `~/demo_resources/models/` or `~/model_garden_models/`).\n- **Environment Variables:** Ensure `AMBARELLA_ARCH` and `SYS_BOARD_BSP` are exported in the shell before running deployment scripts if they are not already set in the environment.
|------|-------|--------------|
| 1 | cooper-system | export build environment |
| 2 | cooper-video-stream | query; start if idle; note **`Wrote …json`** → `--video-stream-json` |
| 3 | ambarella-model-garden | sync if needed → read **`Output dir:`** or **`cat $MG_SKILL_ROOT/.output_dir`** → `--model-garden-dir` |
| 4 | this skill | resolve **`Workspace dir:`** or **`cat $SKILL_ROOT/.workspace_dir`** → `deploy_pipeline.sh --workspace-dir WS --video-stream-json PATH --model-garden-dir DIR --pipeline-id ID --recipe RECIPE` |

Remote target → cooper-remote-device skill.

## Deploy sequence

1. **Prerequisite steps** — steps 1–3 above ([deploy-input-contracts.md](references/deploy-input-contracts.md)).
2. **`deploy_pipeline.sh --workspace-dir WS --pipeline-id ID --recipe RECIPE --video-stream-json VS --model-garden-dir MG`**
   - Recipe id from [`list_recipes.sh`](#available-recipes-catalog)
   - **Incremental:** scaffold hash sync; skip build when scaffold matches and binary exists
   - **Reuse:** scaffold/RTSP unchanged, all nodes healthy → reuses session
   - `--build` / `--restart` as documented in [overview.md](references/overview.md)
   - init → build → allocate → start (or reuse)
3. **Verify** — `/health`; display `/` MJPEG URLs.
4. **Status / stop** — `status_pipeline.sh`; `stop_pipeline.sh`.

Deploy success prints Viewer URL and notes the pipeline runs in the background (Agent or user
can stop later).

## Node startup

Start/restart **only** via `deploy_pipeline.sh` (or `start_pipeline.sh` after
init/build/allocate). Scripts inject `--listen-port`, upstream URLs, and `--node-id` from
state + `deploy.env`.

**Never:** hand-run node scripts/binaries with `--port` or guessed flags; do not `nohup` nodes
to "fix" deploy.

**On failure:** stderr **`Log:`** + log tail → `stop_pipeline.sh`, redeploy or `--build`.
Permission denied on Python nodes: `chmod +x` workspace copies. Port in use: stop pipeline first.

## Available recipes (catalog)

**When the user asks which pipeline combinations exist** — run `list_recipes.sh` (no workspace
required). YAML under [`assets/recipes/`](assets/recipes/) is the source of truth.

```bash
SKILL=/path/to/cooper-pipeline
"$SKILL/scripts/list_recipes.sh"
"$SKILL/scripts/list_recipes.sh" --json
```

## RTSP and models

Deploy inputs (RTSP URL, model `.bin` paths) — see
[deploy-input-contracts.md](references/deploy-input-contracts.md) and [env.md](references/env.md).
Node-to-node `upstream` wiring is defined in recipes and [overview.md](references/overview.md).

**Recommended deploy flags:**

```bash
SKILL_ROOT=/path/to/skills/cooper-pipeline
deploy_pipeline.sh \
  --workspace-dir "$(cat "$SKILL_ROOT/.workspace_dir")" \
  --pipeline-id ID \
  --recipe RECIPE \
  --video-stream-json /tmp/cooper-video-stream-….json \
  --model-garden-dir /path/to/model_garden_models
```

- **`--rtsp-url`** / `--yolox-model` override JSON contract resolution.
- Pipeline selects **RTSP stream B** (`/stream2`) from video-stream JSON for inference.
- **Stopping a pipeline does not stop the video stream.**

## Query, list, and stop

```bash
SKILL=/path/to/cooper-pipeline
SKILL_ROOT="$SKILL"
WS="$(cat "$SKILL_ROOT/.workspace_dir")"

"$SKILL/scripts/list_recipes.sh"
"$SKILL/scripts/list_pipelines.sh" --workspace-dir "$WS"
"$SKILL/scripts/status_pipeline.sh" --workspace-dir "$WS" --all
"$SKILL/scripts/stop_pipeline.sh" --workspace-dir "$WS" --pipeline-id ID
```

**Orphan nodes** (deploy failed, port conflict, manual starts without session):

- List: `status_pipeline.sh --orphans` (device-wide) or `--orphans --pipeline-id ID` (one pipeline)
- Stop one pipeline: `stop_pipeline.sh --workspace-dir WS --pipeline-id ID --orphans` (session + orphans matching `ID_*`)
- Stop device-wide: `stop_pipeline.sh --orphans` — **only when user wants full cleanup**; with multiple pipelines running, prefer scoped `--pipeline-id ID`

Session: `{workspace_dir}/{pipeline_id}/.session.json`. Logs:
`{workspace_dir}/{pipeline_id}/logs/{node_id}.log`.

## Examples

**Agent full deploy (YOLOX)**

→ cooper-system export → cooper-video-stream (`Wrote` JSON) → model-garden sync (`Output dir:` or `.output_dir`) →
`deploy_pipeline.sh --video-stream-json "$VS_JSON" --model-garden-dir "$MG_DIR" --pipeline-id yolox --recipe yolox`

**Explicit overrides**

→ `deploy_pipeline.sh … --rtsp-url rtsp://127.0.0.1:554/stream2 --yolox-model /path/to.bin`

## Additional resources

| Topic | File |
|-------|------|
| Workspace dir persistence contract | [references/workspace-dir-contract.md](references/workspace-dir-contract.md) |
| Deploy input JSON + CLI | [references/deploy-input-contracts.md](references/deploy-input-contracts.md) |
| Recipes, state, scripts | [references/overview.md](references/overview.md) |
| Recipe catalog | [assets/recipes/](assets/recipes/) + `list_recipes.sh` |
| RTSP / model resolution | [references/env.md](references/env.md) |
