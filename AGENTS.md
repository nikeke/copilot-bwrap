# AGENTS.md

## Purpose

This repository contains a security-sensitive `bubblewrap` wrapper for running GitHub Copilot CLI on Tails.

## Key invariants

- Preserve network access.
- Preserve the Tails proxy preload environment that Copilot needs.
- Do not widen host filesystem access by default.
- Treat `$HOME`, `/live`, `/run/user/$UID`, `/run/nosymfollow`, and Tails persistence-backed mountpoints as sensitive.
- Any new host exposure should be narrow and explicit.
- Avoid binding session IPC or desktop integration paths unless the change intentionally weakens the sandbox and the reason is documented.

## Editing guidance

- Keep changes small and easy to audit.
- Prefer exact path binds over broad directory binds.
- Prefer Bash built-ins and straightforward shell over clever tricks.
- Keep the wrapper usable on Tails; if a tool needs special handling, document why in `README.md`.

## Validation

After changing `copilot-bwrap`, run at least:

```bash
./copilot-bwrap --help
./copilot-bwrap -- --version
```

If you change path exposure logic, also verify both:

1. a normal project directory still works
2. a hidden path does not get re-exposed accidentally
