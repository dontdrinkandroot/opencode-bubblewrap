# opencode-bwrap

Run [opencode](https://opencode.ai) inside a [bubblewrap](https://github.com/containers/bubblewrap) sandbox, for use with JetBrains IDEs via their [Agent Client Protocol (ACP)](https://www.jetbrains.com/help/ai-assistant/acp.html) integration.

The sandbox isolates opencode so that, apart from its own config/data/cache/state directories, it can only read and write the project directory it was launched in. The rest of `$HOME` is invisible.

## Prerequisites

- Linux
- [`bwrap`](https://github.com/containers/bubblewrap) installed and setuid / user-namespaces enabled:
  - Debian/Ubuntu: `sudo apt install bubblewrap`
  - Fedora: `sudo dnf install bubblewrap`
  - Arch: `sudo pacman -S bubblewrap`
- [opencode](https://opencode.ai) installed and on `$PATH` (or set `OPENCODE_BIN`)

> On Ubuntu 24.04, unprivileged user namespaces are restricted by AppArmor by default and `bwrap` fails with `bwrap: setting up uid map: Permission denied`. See [Ubuntu 24.04: AppArmor](#ubuntu-2404-apparmor) below.

## Install

```sh
cp opencode-bwrap ~/.local/bin/opencode-bwrap
chmod +x ~/.local/bin/opencode-bwrap
```

Test it:

```sh
cd /path/to/your/project
opencode-bwrap --version
```

To run the full verification suite (syntax, DNS, `/etc` whitelist, env isolation, `/dev/shm`, real `--version`) from the repo checkout:

```sh
./test-opencode-bwrap
```

All flags are passed through unchanged, so `opencode-bwrap acp`, `--model ...`, etc. work as expected.

## Wrapper configuration

The sandbox only exposes `/usr` (plus symlinked `/bin`, `/sbin`, `/lib*`), the project directory, and opencode's own directories. Toolchains installed elsewhere — an SDKMAN JDK, a custom JDK under `/opt`, Gradle/Kotlin, ... — are neither mounted nor on `PATH`, and `--clearenv` drops their environment variables. Extend the sandbox per user with `~/.config/opencode/opencode-bwrap.conf` (line-based; `#` comments and blank lines are ignored; directives may repeat):

| Directive | Meaning |
|---|---|
| `bind-ro SRC [DEST]` | Mount `SRC` read-only into the sandbox at `DEST` (defaults to `SRC`). **Fails hard** if `SRC` is not a directory. |
| `bind-rw SRC [DEST]` | Mount `SRC` read-write into the sandbox at `DEST` (defaults to `SRC`); writes persist on the host. **Fails hard** if `SRC` is not a directory. |
| `path ENTRY` | Append `ENTRY` to the sandboxed `PATH` (after `/usr/bin:/bin:<opencode dir>`). |
| `env NAME=VALUE` | Set environment variable `NAME` to `VALUE` inside the sandbox. |

Example (SDKMAN JDK 21 + Gradle, all under `/opt/sdkman`):

```ini
# ~/.config/opencode/opencode-bwrap.conf
bind-ro /opt/sdkman
path /opt/sdkman/candidates/java/current/bin
path /opt/sdkman/candidates/gradle/current/bin
env JAVA_HOME=/opt/sdkman/candidates/java/current
```

Because `bind-ro` binds the whole `candidates` tree, the `current` symlinks resolve inside the sandbox and both the `java`/`javac` and `gradle` binaries are reachable with `JAVA_HOME` set.

To persist the Gradle dependency cache at its default location, add `bind-rw ~/.gradle` (Gradle stores its dependency cache, wrapper distributions and daemon under `$HOME/.gradle`, the `GRADLE_USER_HOME` default — no `env` needed):

```ini
# ~/.config/opencode/opencode-bwrap.conf
bind-ro /opt/sdkman
bind-rw /home/you/.gradle
path /opt/sdkman/candidates/java/current/bin
path /opt/sdkman/candidates/gradle/current/bin
env JAVA_HOME=/opt/sdkman/candidates/java/current
```

Without `bind-rw`, `$HOME/.gradle` resolves to the sandbox's ephemeral tmpfs and every build re-downloads everything.

Any path listed in `bind-ro`/`bind-rw` becomes reachable by the sandbox (read-only or read-write respectively), and `env` values are visible to all processes inside it; everything not explicitly mounted stays invisible.

## Ubuntu 24.04: AppArmor

Ubuntu 24.04 enables AppArmor-based restriction of unprivileged user namespaces by default (`kernel.apparmor_restrict_unprivileged_userns=1`), but the `bubblewrap` package ships no AppArmor profile. Running the script as an unprivileged user then fails with:

```
bwrap: setting up uid map: Permission denied
```

Check whether you are affected:

```sh
sysctl kernel.apparmor_restrict_unprivileged_userns   # 1 = restricted
```

### Fix: allow `bwrap` to create user namespaces

Create `/etc/apparmor.d/bwrap` (needs `sudo`):

```text
abi <abi/4.0>,
include <tunables/global>

profile bwrap /usr/bin/bwrap flags=(unconfined) {
  userns,

  # Site-specific additions and overrides. See local/README for details.
  include if exists <local/bwrap>
}
```

Load it (no reboot needed):

```sh
sudo apparmor_parser -r /etc/apparmor.d/bwrap
```

This grants user-namespace creation only to the `bwrap` binary itself and leaves the system-wide restriction intact.

### Stricter alternative: `bwrap-userns-restrict`

Ubuntu ships a stricter profile (`bwrap-userns-restrict`) that also blocks programs *inside* the sandbox from creating further namespaces. If it is available on your system (it ships in `apparmor-profiles` on newer releases, e.g. Ubuntu 25.04), install and load it instead of the profile above:

```sh
sudo apt install apparmor-profiles apparmor-utils
sudo install -m 0644 \
  /usr/share/apparmor/extra-profiles/bwrap-userns-restrict \
  /etc/apparmor.d/bwrap-userns-restrict
sudo apparmor_parser -r /etc/apparmor.d/bwrap-userns-restrict
```

### Fallback: disable the restriction system-wide

Only if the profiles above do not help — this disables a security feature for the whole system:

```sh
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
echo 'kernel.apparmor_restrict_unprivileged_userns=0' | sudo tee /etc/sysctl.d/90-unpriv-userns.conf
```

Then retry:

```sh
opencode-bwrap --version
```

## Use with JetBrains ACP

1. Open the AI Chat tool window, click the button in the upper-right and select **Add Custom Agent**. This creates `~/.jetbrains/acp.json`.
2. Point the `command` at the absolute path of this wrapper script (ACP requires an absolute path), with `args: ["acp"]`:

```json
{
  "agent_servers": {
    "OpenCode (sandboxed)": {
      "command": "/home/you/.local/bin/opencode-bwrap",
      "args": ["acp"]
    }
  }
}
```

3. Select **OpenCode (sandboxed)** in the AI Chat agent selector and start chatting.

JetBrains can launch ACP subprocesses with a trimmed environment. If the script reports that `opencode` is not found, pin the binary in the entry's `env`:

```json
{
  "agent_servers": {
    "OpenCode (sandboxed)": {
      "command": "/home/you/.local/bin/opencode-bwrap",
      "args": ["acp"],
      "env": { "OPENCODE_BIN": "/home/you/.opencode/bin/opencode" }
    }
  }
}
```

## What the sandbox exposes

| Path | Access |
|---|---|
| `$PWD` (the project opened in the IDE / current dir) | read-write |
| `~/.config/opencode`, `~/.local/share/opencode`, `~/.cache/opencode`, `~/.local/state/opencode` | read-write |
| `/usr` (+ symlinked `/bin`, `/sbin`, `/lib`, `/lib64`) | read-only |
| `/etc` — whitelist only: `hosts`, `resolv.conf`, `nsswitch.conf`, `passwd`, `group`, `ssl`, `localtime`, `timezone`, `ld.so.cache` | read-only |
| directory containing the `opencode` binary | read-only |
| `/proc`, `/dev`, `/dev/shm`, `/tmp`, `/run` | sandbox-internal (proc/devfs, writable tmpfs) |
| Everything else in `$HOME` (`~/.ssh`, `~/.gitconfig`, `~/.config/gh`, dotfiles, ...) | **not mounted / invisible** |

Notes:

- To give the sandboxed opencode its own config, create `~/.config/opencode/opencode.bwrap.json`. When present, it is mounted read-only as `opencode.json` inside the sandbox, replacing the default global config (`~/.config/opencode/opencode.json` stays untouched on the host).
- Network access is kept open (`--share-net`) as it is required for LLM API calls.
- The environment is **not** passed through wholesale. The sandbox clears it (`--clearenv`) and only forwards an allowlist: `HOME`, `PATH` (minimal, includes the `opencode` binary dir), `TMPDIR`, `XDG_CONFIG_HOME`/`XDG_DATA_HOME`/`XDG_CACHE_HOME`/`XDG_STATE_HOME` (aligned with the bound directories), `TERM`, `LANG`, `NO_COLOR`, and the proxy variables (`HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY` plus lowercase variants, only when set). API keys and any other secrets in the parent environment are **not** forwarded.
  - Credentials come from `opencode auth login`, which stores them in `~/.local/share/opencode/auth.json` (mounted read-write inside the sandbox) — no environment variables needed.
  - To forward extra variables (e.g. a provider API key exported in the shell), set `OPENCODE_BWRAP_PASSTHROUGH` to a space-separated list of names:
    ```sh
    OPENCODE_BWRAP_PASSTHROUGH="OPENAI_API_KEY OPENROUTER_API_KEY" opencode-bwrap --model ...
    ```
    For ACP, set it in the entry's `env`:
    ```json
    {
      "agent_servers": {
        "OpenCode (sandboxed)": {
          "command": "/home/you/.local/bin/opencode-bwrap",
          "args": ["acp"],
          "env": { "OPENCODE_BWRAP_PASSTHROUGH": "OPENAI_API_KEY" }
        }
      }
    }
    ```
  - To mount additional read-only (`bind-ro`) or read-write (`bind-rw`) paths, add `PATH` entries, or inject static env vars inside the sandbox (e.g. a Java/Kotlin/Gradle toolchain that lives outside `/usr`, or a persistent `~/.gradle`), use `~/.config/opencode/opencode-bwrap.conf` — see [Wrapper configuration](#wrapper-configuration).
- The project directory is passed through read-write. Anything explicitly mounted is reachable by the sandboxed process — that includes the opencode config/state dirs, which hold credentials; this is inherent to letting opencode work.
- The sandbox uses a PID/IPC/UTS/user namespace, `--die-with-parent`, and `--new-session` (safe here because ACP mode is headless, and it hardens against TIOCSTI escapes). The host kernel is still shared, and a kernel-level exploit could escape.
- `--tmpfs /tmp` gives opencode and the tools it shells out to a writable, ephemeral, private `/tmp`.
- If you launch the sandbox *inside* `$HOME` itself (i.e. the project directory *is* your home dir), the whole home directory is mounted read-write and nothing is isolated — launch from a real project directory instead.
