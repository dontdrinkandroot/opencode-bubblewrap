# AGENTS.md

## Project

A single-file bash wrapper (`opencode-bwrap`) that runs [opencode](https://opencode.ai) inside a [bubblewrap](https://github.com/containers/bubblewrap) sandbox, primarily for use as a JetBrains ACP custom agent. The sandbox grants the process read-write access to the project dir and opencode's config/data/cache/state dirs only; the rest of `$HOME` is invisible.

Files:

- `opencode-bwrap` — the wrapper (the only code in the repo)
- `README.md` — user-facing docs, including Ubuntu 24.04 AppArmor setup
- `acp.json.example` — JetBrains ACP agent entry example

## Key constraints

- Target platforms: **Ubuntu 24.04** and **Fedora 44** (both use systemd-resolved: `/etc/resolv.conf -> /run/systemd/resolve/stub-resolv.conf`).
- The script must stay a single self-contained bash script with no dependencies beyond `bwrap`, `bash`, and the `opencode` binary.
- `--tmpfs /run` wipes the resolver file the `/etc/resolv.conf` symlink points to; that is why line 40 (`--ro-bind-try /run/systemd/resolve /run/systemd/resolve`) exists and must not be removed. `--ro-bind-try` (not `--ro-bind`) keeps the script working on systems without `/run/systemd/resolve`.
- Network is shared (`--share-net`) on purpose: LLM API calls must work.
- Per-user, line-based `~/.config/opencode/opencode-bwrap.conf` extends the sandbox: `bind-ro SRC [DEST]` (read-only mount, fails hard if `SRC` is missing), `bind-rw SRC [DEST]` (read-write mount, persists to the host, fails hard if `SRC` is missing), `path ENTRY` (appended to the sandboxed `PATH`), `env NAME=VALUE` (`--setenv`). It is parsed by the wrapper on the host before `bwrap` runs; CLI args still pass straight to opencode.
- `/etc` is not bound wholesale; only a read-only whitelist (`hosts`, `resolv.conf`, `nsswitch.conf`, `passwd`, `group`, `ssl`, `localtime`, `timezone`, `ld.so.cache`). Do not reintroduce `--ro-bind /etc /etc`. `resolv.conf` must be bound explicitly — glibc reads it at `/etc/resolv.conf`, not via `/run`.
- Unprivileged user namespaces are restricted on Ubuntu 24.04 (AppArmor); README documents the workaround. Do not silently rely on setuid bwrap.

## Testing

Run the committed test suite (requires the real `opencode` on `PATH`, plus `getent`, `ls`, `env`, `sh` from the sandboxed `/usr`):

```sh
./test-opencode-bwrap
```

It verifies, in order: `bash -n` syntax, DNS resolution, the minimal `/etc` whitelist, environment isolation (`--clearenv` + allowlist, no pass-through secrets), `OPENCODE_BWRAP_PASSTHROUGH` forwarding, writable `/dev/shm`, and the real binary's `--version`. Expected output ends with `all checks passed`.

For manual debugging, the underlying smoke tests are:

```sh
OPENCODE_BIN=/usr/bin/getent ./opencode-bwrap hosts api.openai.com
# expect: resolved IPs, exit 0

OPENCODE_BIN=/usr/bin/ls ./opencode-bwrap /etc
# expect: only hosts, resolv.conf, nsswitch.conf, passwd, group, ssl, localtime, timezone, ld.so.cache
```

## Style

- `set -euo pipefail`, no comments unless needed, no external deps.
- Keep bwrap flags grouped logically (namespaces, filesystems, binds, env, exec).
- Run `./opencode-bwrap --version` from the repo dir as a final sanity check after changes.
