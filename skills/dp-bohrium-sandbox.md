---
name: bohrium-sandbox
description: "Bohrium platform sandbox (python sdbx.py CLI): on-demand cloud VMs for running shell / Python, with GPU options and optional user-storage mounts. Use when: user wants to execute code, debug scripts, do data processing, or run GPU jobs in an isolated environment. NOT for: Bohrium compute jobs (use bohrium-job) or long-lived dev machines (use bohrium-node)."
---

# SKILL: Bohrium Sandbox (`python sdbx.py`)

## Overview

Bohrium Sandbox is an on-demand cloud VM driven by the `python sdbx.py` CLI. It is **not** the E2B SDK, does **not** need `E2B_API_KEY`, and does **not** call `api.e2b.dev` — everything runs on the Bohrium platform with the Bohrium AccessKey.

**Core capabilities:**

- Create sandboxes from templates (CPU / GPU)
- `exec` commands in three modes (foreground / background / PTY)
- `files read/write` to upload/download files or directories
- `terminal` for interactive workloads (REPL / TUI)
- Optional mount of personal disk + share disks
- Billing against personal wallet or project budget

**Comparison with other skills:**

| Scenario | Use |
|----------|-----|
| Quick script validation, debugging, small GPU inference | **This skill (bohrium-sandbox)** |
| Large-scale batch / long jobs | `bohrium-job` |
| Persistent dev machine (SSH / VSCode) | `bohrium-node` |

---

## Installation

The `sdbx` subcommand currently ships only in the **prerelease (beta)** of `lbg` and requires Python >= 3.10. The stable release does not expose `sdbx` and will fail with `invalid choice: 'sdbx'`.

```bash
# Must install the prerelease, otherwise the sdbx subcommand is missing
python3 -m pip install --pre --upgrade lbg

# Verify
python sdbx.py --help    # should list doctor / create / list / exec / files / terminal / ...
```

See version history at <https://pypi.org/project/lbg/#history>. The current beta looks like `4.0.0bNN`.

## Configuration

Requires `BOHR_ACCESS_KEY` (not an E2B key). `sdbx.py` maps it to the `BOHRIUM_ACCESS_KEY` variable expected by the beta `lbg` CLI.

```bash
export BOHR_ACCESS_KEY=<YOUR_BOHR_ACCESS_KEY>
```

Sanity check:

```bash
python sdbx.py doctor --json   # verify auth / SDK / gateway
```

---

## Sandbox lifecycle

### Create

```bash
# Default template sdbxagent (CPU), personal wallet, 12h auto-destroy
python sdbx.py create --json

# Explicit template
python sdbx.py create my-template --json

# Bill against a project budget
python sdbx.py create my-template --project-id <id> --json

# Lifetime override (seconds); 0 = unlimited
python sdbx.py create my-template --timeout 1800 --json
python sdbx.py create my-template --never-timeout --json

# Mount caller's personal disk + share disks
python sdbx.py create my-template --mount-user-storage --json
```

Response fields: `sandboxID` (used by every later command), `templateID`, `state`, `cpuCount`, `memoryMB`, `metadata`.

> Note: `templateID` accepts the template **name**, not a SKU or numeric id. Look it up with `python sdbx.py template ls`.

### List / describe / processes

```bash
python sdbx.py list --json                                   # your sandboxes
python sdbx.py describe <sandbox_id> --with-processes --json # metadata + running processes
python sdbx.py ps <sandbox_id> --json                        # process list
```

`list` adds an `age` column; rows older than 30 minutes are highlighted as a reminder to clean up.

### Kill

```bash
python sdbx.py kill <sandbox_id> --json
```

Safety behaviour:

- No running processes → silent kill
- Running processes + TTY → interactive confirmation
- Running processes + non-TTY (agents, CI, piped shells) → **refused**; pass `--force` to acknowledge and proceed

**After kill, files cannot be read — always pull anything important with `files read` first.**

---

## Templates

```bash
python sdbx.py template ls              # your templates
python sdbx.py template ls --json
python sdbx.py template ls -q           # names only, pipe-friendly

# Create a template (image path + SKU name required)
lbg image ls                      # find an image path
python sdbx.py machine list             # find a SKU
python sdbx.py template create --name <name> --image <image-path> --sku-name <sku>

# Delete (TTY prompts for confirmation; non-TTY must pass --force)
python sdbx.py template rm <name>
python sdbx.py template rm <name> --force --json
```

GPU workflows: pick a GPU template shortcut (see `platform-snapshot.md`), then `exec nvidia-smi` to verify.

---

## Running commands: `exec`

`exec` joins positional args with spaces and sends them to `bash -l -c` inside the sandbox. Shell operators (`&&` / `|` / `>`) work as written.

### Foreground

```bash
python sdbx.py exec <sandbox_id> 'pwd'
python sdbx.py exec <sandbox_id> 'cd /workspace && python train.py'
python sdbx.py exec <sandbox_id> 'cat log.txt | grep ERROR | wc -l'
```

Default `--timeout 60` (seconds). The caller blocks until done. **Use foreground only for things that finish in under a minute.**

### Background

```bash
python sdbx.py exec --background <sandbox_id> 'python train.py > /workspace/out/run.log 2>&1'
# returns {"pid": N, ...}
```

Background semantics:

- `--timeout` defaults to `0` (unlimited). **Do not pass a finite `--timeout`** — it kills the remote command at that boundary (the CLI warns).
- Check status with `python sdbx.py ps <id>` (pid disappears when finished) or by tailing the log via `files read`.

### Retrieve-before-kill SOP for long jobs

```bash
# 1) Output goes to a known path (convention: /workspace/out/)
python sdbx.py exec --background <id> 'mkdir -p /workspace/out && python train.py > /workspace/out/run.log 2>&1'

# 2) Poll until done
python sdbx.py ps <id> --json
python sdbx.py files read <id> /workspace/out/run.log  # tail while running

# 3) Pull everything you need BEFORE kill — kill destroys the disk
python sdbx.py files read <id> /workspace/out/run.log --output ./run.log
python sdbx.py files read <id> /workspace/out/model.bin --format bytes --output ./model.bin

# 4) Verify locally (size / line count / checksum)

# 5) Finally kill
python sdbx.py kill <id>
```

---

## File transfer: `files`

```bash
# Single file
python sdbx.py files write --source ./run.py <sandbox_id> /workspace/run.py --json

# Entire directory (single batch, relative paths preserved)
python sdbx.py files write --source ./project <sandbox_id> /workspace/project --json

# Download to stdout or to a file
python sdbx.py files read <sandbox_id> /workspace/result.csv
python sdbx.py files read <sandbox_id> /workspace/result.csv --output ./result.csv

# Binary (skip utf-8 decode)
python sdbx.py files read <sandbox_id> /workspace/model.bin --format bytes --output ./model.bin
```

> For very large trees, tar locally first: `python sdbx.py files write --source ./big.tar.gz <id> /tmp/big.tar.gz && python sdbx.py exec <id> 'tar -xzf /tmp/big.tar.gz -C /workspace'`

---

## PTY terminal: `terminal`

Only when you genuinely need a TTY: REPLs, TUIs (`htop` / `vim`), sending Ctrl-C to a stuck process. **For "run a command, get its output", use `exec`.**

```bash
python sdbx.py terminal create <sandbox_id> --json              # default timeout=0
python sdbx.py terminal create <sandbox_id> --cwd /workspace --user root --json
python sdbx.py terminal send   <sandbox_id> <pid> 'echo hi\n'   # add \n yourself
python sdbx.py terminal send   <sandbox_id> <pid> $'\x03'       # Ctrl-C
python sdbx.py terminal kill   <sandbox_id> <pid> --json        # kills the pty only, not the sandbox
```

**Important**: `terminal send` returns only `sent_bytes` — the PTY's stdout is **not** echoed back. To capture output, redirect to a file inside the PTY and then `files read`:

```bash
python sdbx.py terminal send <id> <pid> $'cmd > /tmp/out 2>&1\n'
python sdbx.py files read <id> /tmp/out
```

---

## Network (on-demand HTTP proxy)

Sandboxes default to **no outbound proxy**. The image-level `/etc/pip.conf` Aliyun mirror keeps domestic PyPI fast out of the box. Toggle the `ga.dp.tech:8118` proxy on only for overseas reach (pypi.org / GitHub / HuggingFace), and turn it off afterwards.

### Proxy on

```bash
python sdbx.py exec <id> -- bash -c '
mkdir -p ~/.pip && cat > ~/.pip/pip.conf <<EOF
[global]
proxy=http://ga.dp.tech:8118
EOF
cat > ~/.condarc <<EOF
proxy_servers:
  http: http://ga.dp.tech:8118
  https: http://ga.dp.tech:8118
ssl_verify: false
EOF
cat > ~/.curlrc <<EOF
proxy = http://ga.dp.tech:8118
EOF
git config --global http.proxy http://ga.dp.tech:8118
git config --global https.proxy http://ga.dp.tech:8118
'
```

### Proxy off

```bash
python sdbx.py exec <id> -- bash -c '
rm -f ~/.pip/pip.conf ~/.condarc ~/.wgetrc ~/.curlrc
git config --global --unset http.proxy 2>/dev/null || true
git config --global --unset https.proxy 2>/dev/null || true
'
```

### Per-command bypass

When the proxy is on and one specific command needs to skip it:

```bash
wget --no-proxy https://example.com/file
curl --noproxy '*' https://example.com/file
git -c http.proxy= -c https.proxy= clone <url>
pip install --proxy '' <pkg>
HTTP_PROXY= HTTPS_PROXY= http_proxy= https_proxy= <cmd>
```

When the proxy is on, HuggingFace / large `git clone` may hit intermittent 503 / TLS errors — retries usually succeed. **Turn the proxy off when finished**, otherwise domestic access stays slow.

> `apt` is unavailable in the user-mode sandbox (no root). Bake system packages at image build time, not runtime.

---

## End-to-end examples

### A. Quick Python validation

```bash
SID=$(python sdbx.py create --json | jq -r .sandboxID)
python sdbx.py files write --source ./check_torch.py $SID /workspace/check_torch.py
python sdbx.py exec $SID 'cd /workspace && python check_torch.py'
python sdbx.py kill $SID
```

### B. GPU training (background + retrieve before kill)

```bash
SID=$(python sdbx.py create <gpu-template> --timeout 0 --json | jq -r .sandboxID)
python sdbx.py files write --source ./project $SID /workspace/project
python sdbx.py exec --background $SID 'cd /workspace/project && python train.py > /workspace/out/run.log 2>&1'

# After a while
python sdbx.py ps $SID --json
python sdbx.py files read $SID /workspace/out/run.log --output ./run.log
python sdbx.py files read $SID /workspace/out/model.pt --format bytes --output ./model.pt

python sdbx.py kill $SID
```

### C. HuggingFace (overseas reach)

```bash
SID=$(python sdbx.py create --json | jq -r .sandboxID)
# proxy on
python sdbx.py exec $SID -- bash -c '... see "Proxy on" snippet above ...'
python sdbx.py exec $SID 'pip install -i https://pypi.org/simple/ transformers'
python sdbx.py exec $SID 'python -c "from transformers import AutoTokenizer; ..."'
# proxy off
python sdbx.py exec $SID -- bash -c '... see "Proxy off" snippet above ...'
python sdbx.py kill $SID
```

---

## Best practices

- **Reuse, don't recreate**: check `python sdbx.py list` first. Chain short tasks on one sandbox.
- **Kill stale sandboxes**: anything past 30 minutes idle gets flagged; sandboxes keep billing.
- **Always retrieve before kill**: kill destroys the disk. Use `/workspace/out/` as the convention.
- **Long jobs → `--background --timeout 0`**: foreground's 60s default will kill them.
- **GPU work needs a GPU template**: CPU templates fail `nvidia-smi`.
- **`--mount-user-storage` is off by default**: add it when you need the personal/share disks inside.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `lbg: error: invalid choice: 'sdbx'` | Stable lbg installed; no sdbx subcommand | `python3 -m pip install --pre --upgrade lbg` to get the prerelease |
| `access key required` / auth failure | No `lbg login`, `BOHR_ACCESS_KEY` unset | `python sdbx.py doctor --json` or export the env var |
| Foreground command killed by timeout | Default `--timeout 60` too short | Switch to `--background --timeout 0` |
| Background command killed mid-run | Set both `--background` and a finite `--timeout` | Drop the finite timeout (default is 0) |
| Cannot read files after kill | Sandbox destroyed, disk gone | Always `files read` first, kill last |
| Overseas package install / git clone fails | Proxy not enabled | Turn on `ga.dp.tech:8118`; remember to disable after |
| `apt install` fails | User-mode sandbox, no root | Bake system packages at image build time |
| Invalid `templateID` | Passed a SKU / numeric id | Use the template **name** (`python sdbx.py template ls`) |
| `terminal send` returns no output | PTY is streaming; only `sent_bytes` is returned | Redirect to a file in PTY, then `files read` |
| `kill` refused in non-TTY context | Safety guard against killing busy sandboxes | Add `--force` |
| Sandbox auto-destroyed | Default 12h lifetime | `--timeout N` to extend, or `--never-timeout` (kill manually when done) |

---

## Combined workflows

- **sandbox** validates a script → **bohrium-job** scales it out
- **sandbox** preprocesses data → upload to **bohrium-dataset**
- **sandbox** debugs an image → `lbg image ls`, then `python sdbx.py template create` for a reusable template
