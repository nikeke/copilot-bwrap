# copilot-bwrap

`copilot-bwrap` runs GitHub Copilot CLI inside `bubblewrap` with network access intact while hiding broad host trees by default on Tails.

## What it does

- keeps the Tails proxy preload (`LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libproxychains.so.4`)
- forces `HISTFILE=/dev/null` inside the sandbox to avoid a Copilot bug that destroys bash history.
- hides `$HOME`, `/live`, `/run/user/<uid>`, `/mnt`, `/media`, `/run/nosymfollow`, and Tails persistence-backed mountpoints
- auto-binds the current working directory only when it is not under a hidden path
- binds the host `~/.copilot` by default unless you pass `--no-host-copilot`
- kills the sandbox when the wrapper exits unless you pass `--allow-detach`

## Usage

```bash
./copilot-bwrap
./copilot-bwrap --allow-all
./copilot-bwrap --allow-detach --allow-all
./copilot-bwrap --rw-bind "$HOME/Persistent/src/foo" --allow-all
./copilot-bwrap --no-host-copilot login
./copilot-bwrap --host-keyring-token --allow-all
./copilot-bwrap --state-dir "$HOME/.copilot-bwrap/private" --no-host-copilot --allow-all
```

Wrapper options must come before `--`. Everything after `--` is passed to `copilot` unchanged.

## Wrapper options

| Option | Meaning |
| --- | --- |
| `--rw-bind PATH` | Re-expose a host path read-write at the same absolute path. |
| `--ro-bind PATH` | Re-expose a host path read-only at the same absolute path. |
| `--hide PATH` | Hide an additional host path behind an empty tmpfs. |
| `--allow-detach` | Omit `bwrap --die-with-parent` so sandboxed background children may survive after the launcher exits. |
| `--host-keyring-token` | Read the Copilot token from the host Secret Service and pass it in as `COPILOT_GITHUB_TOKEN`. |
| `--state-dir PATH` | Use `PATH` as the host-backed sandbox home store. |
| `--no-host-copilot` | Do not bind the host `~/.copilot`; use a sandbox-private `~/.copilot` under `--state-dir` instead. |
| `--no-bind-pwd` | Do not auto-bind the current working directory. |

## What `--state-dir` persists

`--state-dir` is not metadata about bubblewrap mounts. It is the real host directory that backs the sandbox home at `/home/$USER`.

The wrapper creates a small home skeleton there, including:

- `home/.config`
- `home/.cache`
- `home/.local/bin`
- `home/.local/share`
- `home/.local/state`
- `home/bin`

Anything a process writes under the sandbox home will persist there across runs.

If you use the default host `~/.copilot` bind, Copilot state still comes from your real `~/.copilot`, and the `--state-dir` copy is mainly for other sandbox-home state.

If you use `--no-host-copilot` or the host `~/.copilot` does not exist, Copilot will use the private copy under:

```text
<state-dir>/home/.copilot/
```

That private tree can contain normal Copilot files such as logs, settings, permissions config, command history, and session state.

## Overriding the default host `~/.copilot` bind

Use:

```bash
./copilot-bwrap --no-host-copilot ...
```

After that, Copilot uses the sandbox-private `~/.copilot` inside `--state-dir`. If you want some different host path instead, disable the default and bind exactly what you want with `--rw-bind` or `--ro-bind`.

## Using the host keyring token

Use:

```bash
./copilot-bwrap --host-keyring-token ...
```

The wrapper looks up the host Secret Service item with:

- `service=copilot-cli`
- `account=<host>:<login>`

where `<host>` and `<login>` come from `~/.copilot/settings.json`.

When lookup succeeds, the wrapper exports the token as `COPILOT_GITHUB_TOKEN` only for the sandboxed Copilot process and adds `--secret-env-vars=COPILOT_GITHUB_TOKEN` unless you already set that option yourself.

This is useful when Copilot is logged in via the host keyring and the sandbox intentionally hides the session bus and keyring sockets.

## Detached sandbox children

By default the wrapper uses `bwrap --die-with-parent`, so the sandbox dies when the wrapper process dies.

If you need Copilot to launch watchdogs, monitors, or other detached helpers that survive after the main Copilot process exits, use:

```bash
./copilot-bwrap --allow-detach ...
```

This does **not** widen filesystem exposure by itself, but it is still a meaningful security and control tradeoff:

- sandboxed background processes can keep running after you think the session is over
- they keep their existing network access
- they keep access to whatever host paths and credentials you exposed to that sandbox

So `--allow-detach` is intentionally opt-in rather than the default.

## Adding more tools later

Most system-installed tools do not need special handling because `/` is available read-only and standard system binaries remain visible.

Special-casing is needed when a tool depends on hidden per-user or persistence-backed paths, for example:

- executables in `~/bin`, `~/.local/bin`, or another hidden home directory
- per-user config or cache under `~/.config`, `~/.cache`, `~/.local/share`, or `~/.local/state`
- user-managed wrappers that expect extra environment or config files such as `~/.torsocks.conf`
- writable persistent state outside the current project tree

If you later install tools such as `cargo`, `uv`, `pipx`, `npm`, `pnpm`, or other home-managed toolchains, the rule is the same: bind only the exact executable and exact config/cache/state paths that the tool needs. Avoid broad binds of `~/.local`, `~/.cargo`, `~/.config`, or `/run/user/<uid>` unless you intentionally want to weaken the sandbox boundary.
