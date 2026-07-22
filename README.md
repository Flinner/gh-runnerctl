# gh-runnerctl

A sudoless, config-driven pool manager for self-hosted GitHub Actions runners.
One Python file, standard library only, docker-style CLI.

```console
$ runnerctl status

pool https://github.com/my-org   manager=systemd   base=/home/me/actions-runner   config: count=4

  IDX  NAME       SERVICE   GITHUB   JOB
  1    node1-1    active    online   idle
  2    node1-2    active    online   busy
  3    node1-3    active    online   idle
  4    node1-4    active    online   idle
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

- **No sudo, ever.** Services are *user-level* systemd units (with automatic
  session linger), and on hosts where the systemd user bus is disabled it falls
  back to a detached-process manager transparently.
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
| `url` | — | Org URL (`https://github.com/my-org`) or repo URL (`.../owner/repo`) |
| `count` | `2` | Desired number of runners; `up` converges both directions |
| `labels` | `["self-hosted"]` | Runner labels |
| `base_dir` | `~/actions-runner` | Each runner lives in `<base_dir>/<dir_prefix><N>` |
| `version` | `latest` | Runner release to install, e.g. `"2.335.1"` |
| `name_prefix` | hostname | GitHub-side runner names `<prefix>-<N>` |
| `manager` | `auto` | `auto` \| `systemd` \| `nohup` (both sudoless) |
| `group` | — | Runner group (org-level registration only) |
| `dir_prefix` | `runner` | Directory naming; see *Adopting an existing setup* |

## Commands

| Command | Does |
|---|---|
| `init` | Write a starter `runners.toml` |
| `up [--count N] [--force]` | Converge the pool to the config (provision, register, start, prune) |
| `scale N` | `up --count N` |
| `status` / `ps` | Pool table: local directories + service state + GitHub online/busy |
| `down [--force]` | Stop all runners, keep registrations |
| `rm <idx...> \| --all [--force]` | Deregister and delete runners |
| `start` / `stop` / `restart [idx...]` | Service control |
| `logs [idx] [-f]` | Runner logs (journalctl or logfile, per manager) |
| `config` | Print the resolved configuration |

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

## Service management details

**systemd (default when available).** One template unit,
`~/.config/systemd/user/gh-runner@.service`, instantiated per runner
(`gh-runner@1`, `gh-runner@2`, ...): `Restart=always`, `KillSignal=SIGINT` so a
runner exits cleanly. Session linger is enabled automatically (`loginctl
enable-linger`) so runners survive logout and reboot — no root at any step.

**nohup fallback.** Some hosts (typically HPC login nodes) disable the systemd
user bus. `manager = "auto"` detects this and falls back to detached process
groups with pidfiles (`.runnerctl.pid`) and per-runner logfiles (`runner.log`).
Same CLI, no restart-on-crash.

## Comparison

| | root/sudo | deps | declarative `scale` | GitHub-side status | no-token fallback |
|---|---|---|---|---|---|
| **gh-runnerctl** | no | none | yes | yes | yes |
| vbem/multi-runners | required | bash+jq | no (imperative add/del) | no | no |
| gershnik/multi-gh-action-runner | no | Python | config edit + restart | no | no |
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
