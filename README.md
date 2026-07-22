# gh-runnerctl

A sudoless, config-driven pool manager for self-hosted GitHub Actions runners.
One Python file, standard library only, docker-compose-style CLI.

```console
$ runnerctl status

instance default   config=runners.toml   pools: main=4, gpu=2
supervisor running (pid 71324, systemd)

  POOL  IDX  NAME          RUNNER   GITHUB   JOB
  main  1    node1-main-1  running  online   idle
  main  2    node1-main-2  running  online   busy
  main  3    node1-main-3  running  online   idle
  main  4    node1-main-4  running  online   idle
  gpu   1    node1-gpu-1   running  online   idle
  gpu   2    node1-gpu-2   stopped  online   -
```

**The config file is the interface.** Declare any number of pools in one
`runners.toml`; edit it, then:

- `runnerctl up` — full converge: download the runner release, register
  missing runners, prune surplus ones, start/reload the supervisor.
- `runnerctl reload` — soft apply: the running supervisor re-reads the file
  and starts/stops runner processes accordingly. No GitHub calls.

Every mutating command takes `--dry-run` / `-n` and prints the plan instead:

```console
$ runnerctl up -n
plan for instance default (runners.toml):
  + register runner gpu/2 as node1-gpu-2 (https://github.com/my-org)
  - deregister and delete runner main/5 (node1-main-5, idle)
  * reload supervisor (SIGHUP pid 71324)
:: dry run — nothing changed
```

## Why another one?

Every existing multi-runner tool assumes root, Docker, or Kubernetes.
`runnerctl` targets the host where you have none of those — an HPC login node,
a shared lab workstation, a locked-down VM:

- **No sudo, ever.** `runnerctl` itself is the one service: a single
  *user-level* systemd unit (with automatic session linger) runs the
  `runnerctl daemon` supervisor, which owns every runner process of every
  pool as a child and restarts each independently with backoff. On hosts
  where the systemd user bus is disabled, the same supervisor runs as a
  detached process — identical behavior, including per-runner crash restarts.
- **No dependencies.** One file, Python 3.6+ standard library only.
- **No PAT required.** Works best with a token (`GITHUB_TOKEN` or a logged-in
  `gh` CLI), but degrades to an interactive flow that prompts you to paste
  registration tokens from the GitHub settings page.

## Install

```bash
curl -fLo ~/.local/bin/runnerctl https://raw.githubusercontent.com/flinner/gh-runnerctl/main/runnerctl
chmod +x ~/.local/bin/runnerctl
curl -fLo runners.toml https://raw.githubusercontent.com/flinner/gh-runnerctl/main/runners.toml.example
```

Updating is never automatic: `runnerctl check-update` tells you when a newer
version exists (and prints the command), `runnerctl update` installs it in
place.

## Quick start

```bash
runnerctl init                 # or start from runners.toml.example
$EDITOR runners.toml           # set url (org or repo) and count
runnerctl up                   # converge: download, register, supervise
runnerctl status               # see the pool
```

Registering at the **org level** (`url = "https://github.com/my-org"`) is
recommended: one pool serves every repo in the org and GitHub load-balances
queued jobs across the runners automatically.

## Configuration

`runnerctl` looks for `runners.toml` in the current directory, then
`~/.config/runnerctl/runners.toml` (override with `-c FILE` or
`$RUNNERCTL_CONFIG`).

```toml
name = "default"          # instance name -> runnerctl-<name>.service
manager = "auto"          # supervisor hosting: auto | systemd | nohup

[defaults]                # keys shared by every pool
url = "https://github.com/my-org"
base_dir = "~/actions-runner"

[pool.main]
count = 4
labels = ["self-hosted", "linux", "x64"]

[pool.gpu]
count = 2
labels = ["self-hosted", "linux", "x64", "gpu"]
```

A single anonymous `[pool]` (no named tables) is also valid — that's the pool
`default`, whose runner directories are `runner<N>` and whose GitHub-side
names are plain `<host>-<N>`.

| Key | Default | Meaning |
|---|---|---|
| `name` (top level) | `default` | Instance identity; names the supervisor service and state |
| `manager` (top level) | `auto` | How the supervisor is hosted: `auto` \| `systemd` \| `nohup` |
| `url` | — | Org URL (`https://github.com/my-org`) or repo URL (`.../owner/repo`) |
| `count` | `2` | Desired runners in the pool; `up` converges both directions |
| `labels` | `["self-hosted"]` | Runner labels |
| `base_dir` | `~/actions-runner` | Runners live in `<base_dir>/<dir_prefix><N>` |
| `version` | `latest` | Runner release to install, e.g. `"2.335.1"` |
| `name_prefix` | hostname | GitHub-side names `<prefix>-<pool>-<N>` |
| `group` | — | Runner group (org-level registration only) |
| `dir_prefix` | pool name | Directory naming; see *Adopting an existing setup* |

Pool keys go in `[defaults]` (shared) or per `[pool.<name>]` (override).
Pools may share a `base_dir` (distinct `dir_prefix` enforced) or use separate
ones — even different orgs/repos per pool.

## Commands

Runners are addressed as `<pool>/<idx>` (`gpu/2`), a pool name (`gpu` = all
of it), or a bare index in single-pool files.

| Command | Does |
|---|---|
| `init` | Write a starter `runners.toml` |
| `up [--force] [-n]` | Converge every pool to the config (provision, register, supervise, prune) |
| `reload` | Supervisor re-reads the config and applies it; no GitHub work |
| `status` / `ps` | Pool table: supervisor + runner processes + GitHub online/busy |
| `down [--force] [-n]` | Stop the supervisor and all runners, keep registrations |
| `rm <ref...> \| --all [--force] [-n]` | Deregister and delete runners |
| `start` / `stop` / `restart <ref...>` | Per-runner control (stopped runners stay registered, not respawned) |
| `logs [ref] [-f]`, `logs --daemon` | Runner logs / supervisor log |
| `config` | Print the resolved configuration |
| `daemon` | Run the supervisor in the foreground (started for you by `up`) |
| `check-update` / `update` | Check for / install a newer runnerctl — never automatic |

Anything that would kill a running job (`stop`, `rm`, pruning in `up`,
`down`) checks the runner's busy state on GitHub first and skips busy runners
unless `--force`.

## Authentication

Three rungs, tried in order:

1. **`GITHUB_TOKEN`** / **`GH_TOKEN`** environment variable.
   Fine-grained PAT permissions: repo-level `administration:write`, or org-level
   `organization_self_hosted_runners:write`. Classic PAT: `repo`, or `admin:org`.
2. **`gh` CLI** — if you're logged in (`gh auth login`), its token is used.
3. **Interactive** — no token at all: `runnerctl` prints the exact GitHub
   settings-page URL, you paste the registration/removal token it shows, and the
   tool loops with a fresh prompt if a token is expired or spent. Remote status
   and busy-checks are unavailable in this mode.

Registration tokens minted via the API are cached and reused across a whole
`up`/`rm` pass (they live ~1 h) instead of one API call per runner.

## Adopting an existing setup

Already have hand-configured runners in `~/actions-runner/actions1..4`? Use a
single anonymous `[pool]` with:

```toml
[pool]
url = "https://github.com/my-org"
count = 4
dir_prefix = "actions"
```

`runnerctl up` recognizes registered runners by their `.runner` file, keeps
their existing GitHub-side names, and simply takes over supervision. Stop any
tmux/nohup copies first — two listeners on one registration conflict.

## How supervision works

`runnerctl` is the service — runners never touch systemd themselves.

**The supervisor** (`runnerctl daemon`) owns every runner's `run.sh` as a
direct child: it respawns a crashed runner with exponential backoff (without
disturbing its siblings), re-reads the config file on `SIGHUP` (sent by
`up`/`reload`/`start`/`stop`), and shuts every runner down cleanly on
`SIGTERM`. It holds no GitHub credentials — registration, busy checks, and
the interactive token flow all happen in the CLI commands, so the daemon can
run headless forever.

**Hosting the supervisor.** With a systemd user bus (default), `up` installs
a single user unit `runnerctl-<name>.service` (`Restart=always`,
`ExecReload` = HUP) and enables session linger (`loginctl enable-linger`), so
the pool survives logout and reboot — no root at any step. On hosts where the
user bus is disabled (typical HPC login nodes), the same supervisor runs as a
detached process with a pidfile: per-runner crash restarts still work; only
the supervisor itself isn't resurrected after a reboot (re-run `runnerctl up`).

Per-runner logs land in `<runner dir>/runner.log`; the supervisor's own log
is in the journal (systemd) or `~/.local/state/runnerctl/<name>/daemon.log`.

## Comparison

| | root/sudo | deps | declarative converge | GitHub-side status | no-token fallback |
|---|---|---|---|---|---|
| **gh-runnerctl** | no | none | yes (`up`/`reload` from config) | yes | yes |
| vbem/multi-runners | required (sudoers, user-per-runner) | bash+jq | no (imperative add/del) | no (`status` unimplemented) | no |
| gershnik/multi-gh-action-runner | sudo to install daemon | Python+PyGithub | config edit + restart; one crash stops the whole pool | no | no |
| ARC / GARM / docker images | k8s / daemon / docker | heavy | yes (autoscale) | yes | no |

If you have Kubernetes or Docker and want autoscaling, use
[actions-runner-controller](https://github.com/actions/actions-runner-controller)
or [GARM](https://github.com/cloudbase/garm) — this tool is deliberately the
small option for a fixed host.

## Limitations

- Linux only (macOS could work with `manager = "nohup"`; untested).
- Fixed-size pools — no webhook-driven autoscaling by design.
- No ephemeral (`--ephemeral`) mode yet.
- github.com only (no GHES) for now.

## License

MIT
