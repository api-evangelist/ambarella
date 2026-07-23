---
name: cooper-remote-device
description: >-
  SSH bridge for device skills: rsync skill to a target board, exec, ssh cat results,
  pull skill-local config back. Use when the user targets another board (IP,
  hostname, SSH, remote), or when the local machine is not on-device and device
  work is needed — even if they do not say "cooper-remote-device". If already
  on-device with no remote request, run the device skill locally instead.
compatibility: Machine with openssh-client, rsync; sshpass for one-time ssh-copy-id only
license: Apache-2.0
---

# Cooper Remote Device

Run a **device skill** on a **target device** over SSH. The **local machine** is wherever the agent runs commands first — it may be a PC or already a Cooper board.

Device skills keep their own commands and paths; this skill only adds rsync and SSH. No bundled script.

## When to use

**On-device check** (Ambarella / Cooper) — use **[cooper-system](../cooper-system/SKILL.md)** to
determine whether the local machine is a Cooper device (arch resolvable).

- **On-device** → run the device skill locally when the user did not request another board.
- **Not on-device** → use this SSH bridge.

Do not duplicate env parsing or call cooper-system scripts from this skill's docs — follow cooper-system only.

**Remote to another device** — user clearly asks to use a **different** board than the local machine, for example:

- Names an IP, hostname, or `label` from `.device_ssh` that is not “this board”
- Says remote, SSH, another board, or run on a named device

**Decision:**

| Local machine | User intent | Action |
|---------------|-------------|--------|
| On-device (arch resolvable) | No request to use another device | Run the device skill **locally** — do not use this skill |
| On-device | Explicitly operate on **another** device | Use this bridge |
| Not on-device | Device work is needed | Use this bridge (pick target from `.device_ssh` or user IP/label) |
| Not on-device | Explicit remote target | Use this bridge |

When bridging **from** one device **to** another, `HOST_SKILL_DIR` is still the skill folder on the **local** machine; rsync and pull-back use that path on the machine where the agent runs.

The device skill's `compatibility` should mention device, embedded, or Cooper.

## Rules

1. **Public key only** — `rsync`, `ssh`, and `ssh cat` use `ssh -o BatchMode=yes`. Run a quick probe before each step.
2. **`sshpass` only for setup** — one-time `ssh-copy-id`, never for remote tasks.
3. **Do not change device argv** — copy the device skill command as documented; adjust quoting or wrap in `bash -lc` only if device env vars are missing over SSH.
4. **Follow the device skill for paths** — on the **target** device, use its SKILL.md paths, not paths from the local machine.
5. **Pull small persisted files back** — after each bridge, sync skill-local dotfiles from target → local skill folder (not large model or output trees).

## Paths

| Variable | Meaning |
|----------|---------|
| `HOST_SKILL_DIR` | Active device skill folder on the **local** machine (contains `SKILL.md`) |
| `SSH_TARGET` | `ssh_user@ssh_host` for the **target** device (from `.device_ssh`) |
| `REMOTE_STAGING_ROOT` | Staging dir on the target (default `/tmp/skill-staging`) |
| `REMOTE_SKILL` | Staged skill on the target; usually `REMOTE_STAGING_ROOT`, or `REMOTE_STAGING_ROOT/<skill-name>` if shared |

**What gets copied**

- **Local → target:** only `HOST_SKILL_DIR`. Not `cooper-remote-device` or other skills.
- **Staging:** skill code on the target. Large artifacts use paths from the device skill, often outside staging.
- **Target → local:** small persisted files under the skill directory (step 5).

**SSH options (all remote commands):**

`-o BatchMode=yes -o ConnectTimeout=5 -o ServerAliveInterval=30`

`ConnectTimeout` limits **connecting** only, not how long a remote command runs. `ServerAliveInterval` helps long rsync/exec sessions.

**Device environment variables:** on the target, escape as `\$VAR` from the local shell, or use `bash -lc '...'`.

## Device config (`.device_ssh`)

File: `cooper-remote-device/.device_ssh` (gitignored). Create or update after setup. Schema: [`.device_ssh.example`](.device_ssh.example) — do not ask the user to copy it.

**Choose target profile** (first match wins):

1. User names an IP or `label` for the **remote** board
2. `default_device` in `.device_ssh`
3. Only one entry in `devices`
4. Otherwise ask which board is the target

### First-time setup

1. Ask for the target IP (and staging path if not `/tmp/skill-staging`).
2. Test key login: `ssh -o BatchMode=yes root@IP` then `lychee@IP`.
3. If both fail, install the local machine's SSH key:

```bash
test -f ~/.ssh/id_ed25519.pub || ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
sshpass -p '' ssh-copy-id -o StrictHostKeyChecking=accept-new "root@${IP}" \
  || sshpass -p 'ambarella' ssh-copy-id -o StrictHostKeyChecking=accept-new "lychee@${IP}"
```

4. Verify with `BatchMode=yes`, then write `.device_ssh`.
5. If key login fails later, stop and repeat setup — do not use a password for rsync or exec.

## Workflow

Start each remote step with:

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 "${SSH_TARGET}" "echo ok"
```

### 1. Sync skill to target

```bash
rsync -az --delete -e "ssh -o BatchMode=yes -o ServerAliveInterval=30" \
  --exclude '.git' --exclude 'references' \
  "${HOST_SKILL_DIR}/" "${SSH_TARGET}:${REMOTE_SKILL}/"
```

Re-run after reboot if staging was under `/tmp`.

### 2. Resolve config on target

Read the device skill's SKILL.md. Read skill-local dotfiles on the **target** if needed:

```bash
ssh -o BatchMode=yes "${SSH_TARGET}" \
  "cat '${REMOTE_SKILL}/<file>' 2>/dev/null || true"
```

### 3. Run the device skill command

Working directory on target: `REMOTE_SKILL`.

```bash
ssh -o BatchMode=yes -o ServerAliveInterval=30 "${SSH_TARGET}" \
  "bash -lc 'cd \"${REMOTE_SKILL}\" && <device skill command>'"
```

### 4. Read results

```bash
ssh -o BatchMode=yes "${SSH_TARGET}" "cat '<path-on-target>'"
```

Separate `ssh cat` from exec if logs would mix with JSON.

### 5. Pull persisted state to local machine

For each skill-local file from the device skill's SKILL.md (e.g. `.output_dir`):

```bash
ssh -o BatchMode=yes "${SSH_TARGET}" \
  "cat '${REMOTE_SKILL}/<skill-local-file>' 2>/dev/null" > "${HOST_SKILL_DIR}/<skill-local-file>"
```

Skip large directories. Step 1 pushes updates to the target on the next bridge.
