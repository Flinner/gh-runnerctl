# gh-runnerctl

A sudoless, config-driven pool manager for self-hosted GitHub Actions runners.
One Python file, standard library only, docker-style CLI.

```console
$ runnerctl status

pool default (https://github.com/my-org)   base=/home/me/actions-runner   config: count=4
supervisor running (pid 71324, systemd)

  IDX  NAME       RUNNER    GITHUB   JOB
  1    node1-1    running   online   idle
  2    node1-2    running   online   busy
  3    node1-3    running   online   idle
  4    node1-4    running   online   idle
```

Declare the pool once in `runners.toml`; `runnerctl up` converges the host to it —
downloads the runner release, registers missing runners, starts stopped services,
and prunes surplus ones. Scale with one command:

```console
$ runnerctl scale 8      # grow
$ runnerctl scale 2      # shrink (busy runners are skipped unless --force)
```

## Why another one?

Every existing multi-runner tool assumes root, Docker, or Kubernetes.
`runnerctl` targets the host where you have none of those — an HPC login node, a
shared lab workstation, a locked-down VM:

- **No sudo, ever.** `runnerctl` itself is the one service: a single
  *user-level* systemd unit (with automatic session linger) runs the
  `runnerctl daemon` supervisor, which owns every runner process as a child
  and restarts each independently with backoff. On hosts where the systemd
  user bus is disabled the same supervisor runs as a detached process —
  identical behavior, including per-runner crash restarts.
- **No dependencies.** One file, Python 3.6+ stdlib. `curl`-and-run install.
- **No PAT required.** Works best with a token (`GITHUB_TOKEN` or a logged-in
  `gh` CLI), but degrades to an interactive flow that prompts you to paste
  registration tokens from the GitHub settings page.

## Install

```bash
curl -fLo ~/.local/bin/runnerctl https://raw.githubusercontent.com/flinner/gh-runnerctl/main/runnerctl
chmod +x ~/.local/bin/runnerctl
```

## Quick start

```bash
runnerctl init                 # writes runners.toml
$EDITOR runners.toml           # set url (org or repo) and count
runnerctl up                   # converge: download, register, start
runnerctl status               # see the pool
```

Registering at the **org level** (`url = "https://github.com/my-org"`) is
recommended: one pool serves every repo in the org and GitHub load-balances
queued jobs across the runners automatically.

## Configuration

`runnerctl` looks for `runners.toml` in the current directory, then
`~/.config/runnerctl/runners.toml` (override with `-c FILE` or
`$RUNNERCTL_CONFIG`).

| Key | Default | Meaning |
|---|---|---|
| `name` | `default` | Pool identity; names the supervisor service and its state |
| `url` | — | Org URL (`https://github.com/my-org`) or repo URL (`.../owner/repo`) |
| `count` | `2` | Desired number of runners; `up` converges both directions |
| `labels` | `["self-hosted"]` | Runner labels |
| `base_dir` | `~/actions-runner` | Each runner lives in `<base_dir>/<dir_prefix><N>` |
| `version` | `latest` | Runner release to install, e.g. `"2.335.1"` |
| `name_prefix` | hostname | GitHub-side runner names `<prefix>-<N>` |
| `manager` | `auto` | How the supervisor is hosted: `auto` \| `systemd` \| `nohup` (both sudoless) |
| `group` | — | Runner group (org-level registration only) |
| `dir_prefix` | `runner` | Directory naming; see *Adopting an existing setup* |

### Multiple pools

One config file = one pool = one supervisor. For several pools on a host
(different labels, orgs, or repo scopes), write one file per pool with a
distinct `name` and address each with `-c`:

```console
$ runnerctl up                      # pool "default" from ./runners.toml
$ runnerctl -c gpu.toml up          # pool "gpu" -> runnerctl-gpu.service
$ runnerctl -c gpu.toml status
```

Pools may share a `base_dir` (give each a distinct `dir_prefix`) or use
separate ones; supervisors, state, and GitHub-side runner names are
namespaced per pool either way. See `runners.toml.example`.

## Commands

| Command | Does |
|---|---|
| `init` | Write a starter `runners.toml` |
| `up [--count N] [--force]` | Converge the pool to the config (provision, register, supervise, prune) |
| `scale N` | Set `count = N` **in the config file** and converge |
| `status` / `ps` | Pool table: supervisor + runner processes + GitHub online/busy |
| `down [--force]` | Stop the supervisor and all runners, keep registrations |
| `rm <idx...> \| --all [--force]` | Deregister and delete runners |
| `start` / `stop` / `restart [idx...]` | Per-runner control (stopped runners stay registered, not respawned) |
| `logs [idx] [-f]`, `logs --daemon` | Runner logs / supervisor log |
| `config` | Print the resolved configuration |
| `daemon` | Run the supervisor in the foreground (started for you by `up`) |

Anything that would kill a running job (`stop`, `rm`, pruning in `up`) checks
the runner's busy state on GitHub first and skips busy runners unless `--force`.

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

## Adopting an existing setup

Already have hand-configured runners in `~/actions-runner/actions1..4`? Set:

```toml
dir_prefix = "actions"
count = 4
```

`runnerctl up` recognizes registered runners by their `.runner` file, keeps
their existing GitHub-side names, and simply takes over service management.
Stop any tmux/nohup copies first — two listeners on one registration conflict.

## How supervision works

`runnerctl` is the service — runners never touch systemd themselves.

**The supervisor** (`runnerctl daemon`) owns every runner's `run.sh` as a
direct child: it respawns a crashed runner with exponential backoff (without
disturbing its siblings), re-reads desired state on `SIGHUP` (sent by
`up`/`scale`/`start`/`stop`), and shuts every runner down cleanly on
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

Per-runner logs always land in `<runner dir>/runner.log`; the supervisor's
own log is in the journal (systemd) or `.runnerctl/<name>/daemon.log`.

## Comparison

| | root/sudo | deps | declarative `scale` | GitHub-side status | no-token fallback |
|---|---|---|---|---|---|
| **gh-runnerctl** | no | none | yes | yes | yes |
| vbem/multi-runners | required (sudoers, user-per-runner) | bash+jq | no (imperative add/del) | no (`status` unimplemented) | no |
| gershnik/multi-gh-action-runner | sudo to install daemon | Python+PyGithub | config edit + restart; one crash stops the whole pool | no | no |
| ARC / GARM / docker images | k8s / daemon / docker | heavy | yes (autoscale) | yes | no |

If you have Kubernetes or Docker and want autoscaling, use
[actions-runner-controller](https://github.com/actions/actions-runner-controller)
or [GARM](https://github.com/cloudbase/garm) — this tool is deliberately the
small option for a fixed host.

## Limitations

- Linux only (macOS could work with the nohup manager; untested).
- Fixed-size pool — no webhook-driven autoscaling by design.
- No ephemeral (`--ephemeral`) mode yet.
- github.com only (no GHES) for now.

## License

MIT
