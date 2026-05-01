# copilot-bwrap

`copilot-bwrap` runs GitHub Copilot CLI inside `bubblewrap` with network access intact while hiding broad host trees by default on Tails.

## What it does

- keeps the Tails proxy preload (`LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libproxychains.so.4`)
- forces `HISTFILE=/dev/null` inside the sandbox
- hides `$HOME`, `/live`, `/run/user/<uid>`, `/mnt`, `/media`, `/run/nosymfollow`, and Tails persistence-backed mountpoints
- auto-binds the current working directory only when it is not under a hidden path
- binds the host `~/.copilot` by default unless you pass `--no-host-copilot`

## Usage

```bash
./copilot-bwrap
./copilot-bwrap --allow-all
./copilot-bwrap --rw-bind "$HOME/Persistent/src/foo" --allow-all
./copilot-bwrap --no-host-copilot login
./copilot-bwrap --state-dir "$HOME/.copilot-bwrap/private" --no-host-copilot --allow-all
```

Wrapper options must come before `--`. Everything after `--` is passed to `copilot` unchanged.

## Wrapper options

| Option | Meaning |
| --- | --- |
| `--rw-bind PATH` | Re-expose a host path read-write at the same absolute path. |
| `--ro-bind PATH` | Re-expose a host path read-only at the same absolute path. |
| `--hide PATH` | Hide an additional host path behind an empty tmpfs. |
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

## Adding more tools later

Most system-installed tools do not need special handling because `/` is available read-only and standard system binaries remain visible.

Special-casing is needed when a tool depends on hidden per-user or persistence-backed paths, for example:

- executables in `~/bin`, `~/.local/bin`, or another hidden home directory
- per-user config or cache under `~/.config`, `~/.cache`, `~/.local/share`, or `~/.local/state`
- user-managed wrappers that expect extra environment or config files such as `~/.torsocks.conf`
- writable persistent state outside the current project tree

`rebar3` is handled explicitly because this Tails setup uses wrappers in `~/bin` plus persistent config and cache under `~/Persistent/rebar3/`.

If you later install tools such as `cargo`, `uv`, `pipx`, `npm`, `pnpm`, or other home-managed toolchains, the rule is the same: bind only the exact executable and exact config/cache/state paths that the tool needs. Avoid broad binds of `~/.local`, `~/.cargo`, `~/.config`, or `/run/user/<uid>` unless you intentionally want to weaken the sandbox boundary.

## Notes

- This wrapper does **not** add `--allow-all`; pass that explicitly when you want it.
- Mouse behavior is left entirely to normal Copilot arguments or your own shell wrapper.
