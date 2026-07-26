# agentcubicle

Run opencode or claude code in an isolated Docker container. Project files are mounted read-write; sessions start in a read-only plan mode that you manually escalate out of once you're ready to let it act.

## Quick Start

```sh
# Build the custom image with opencode and dev tools
agentcubicle setup

# Add Claude Code to the image
agentcubicle setup --claude

# Inspect the image for installed tools
agentcubicle check

# Run opencode in a container
agentcubicle opencode

# Run claude code in a container
agentcubicle claude
```

## Configuring opencode models and providers

`opencode`'s own config already propagates from your host: `~/.config/opencode/opencode.json` (or `.jsonc`) is mounted read-only and copied into the container on every run (see "How it works" below) — nothing extra to set up.

Without any provider configured there, only opencode's built-in free fallback models are available. Point it at a self-hosted/local model behind an OpenAI-compatible gateway (LiteLLM, vLLM, an Ollama OpenAI-compat endpoint, etc.) by adding a `provider` section referencing the API key as `{env:YOUR_VAR}` rather than a literal value, and set `model`/`small_model` to select it by default:

```jsonc
// ~/.config/opencode/opencode.json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "local/qwen3-coder",
  "small_model": "local/qwen3-coder",
  "provider": {
    "local": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local LiteLLM",
      "options": {
        "baseURL": "https://your-litellm-gateway.example",
        "apiKey": "{env:LITELLM_API_KEY}"
      },
      "models": {
        "qwen3-coder": {
          "name": "qwen3-coder",
          "reasoning": false,
          "limit": {
            "context": 131072,
            "output": 60000
          }
        }
      }
    }
  }
}
```

```sh
agentcubicle opencode --env LITELLM_API_KEY="$(cat ~/.litellm-key)"
```

The same `{env:VAR}` + `--env` pattern works for a hosted vendor too — opencode ships built-in support for the common ones, so the config is just an API key:

```jsonc
"provider": { "anthropic": { "options": { "apiKey": "{env:ANTHROPIC_API_KEY}" } } }
```

Note that `opencode auth login` does **not** work for this: it stores credentials at `~/.local/share/opencode/auth.json`, which agentcubicle never mounts or copies into the container (only `~/.config/opencode` is) — anything saved there never reaches a session. The config-file-plus-`--env` approach above is the one that actually works, and is verified against a real `opencode debug config` resolution.

**Literal keys are handled automatically too.** If you'd rather just keep a literal key in your `opencode.json` (as many personally-managed configs do), you don't have to convert it to `{env:VAR}` yourself: agentcubicle detects any literal `provider.*.options.apiKey`, forwards the real value into the container as a generated `AC_OPENCODE_<PROVIDER>_APIKEY` env var, and gives the container a redacted copy of the config that references it. So the literal never lands in the container's config copy (nor in your shell history, since it's read from the file rather than typed into `--env`). This applies only when the config is valid JSON `jq` can parse — a `.jsonc` with comments is left as-is, keeping its literal key.

If your config has no `provider` section at all, `agentcubicle opencode` prints a reminder about this every time you start a session.

## First-time Claude Code authentication

The container never receives your host's live Claude Code credentials — only `~/.claude/settings.json` itself is ever mounted read-only and copied out (see "How it works" below). If you already have a static token at `~/.claude/oauth-token` (or exported as `CLAUDE_CODE_OAUTH_TOKEN`, or an `ANTHROPIC_API_KEY` passed with `--env`), there's nothing to do here — `agentcubicle claude` auto-detects and forwards it for you automatically, no bootstrap needed.

If none of those are found, `agentcubicle claude` detects this itself and runs the bootstrap below for you automatically instead of starting a normal session that would just hit a login wall: it runs `claude setup-token` directly, then prints exactly what to do with the resulting token and drops you into a shell to do it. You do not need to remember these steps — just run `agentcubicle claude` and follow along. They are spelled out here for reference, and for doing it manually if you ever want to:

1. Build the image if you haven't yet: `agentcubicle setup --claude`
2. Start an interactive container: `agentcubicle claude`
3. Inside the container, generate a long-lived token: `claude setup-token`
   - This prints a URL. Open it in a browser, log into your Claude account, and authorize.
   - It then prints a token string. This only exists in the container's throwaway home directory — it will **not** survive the container exiting unless you save it yourself.
4. Save the token into the bind-mounted project directory so it survives the container exiting, e.g.:
   ```sh
   echo "<paste the printed token>" > /home/user/project/.claude-oauth-token
   ```
5. Exit the container (`exit` or Ctrl-D) — the token is now safely saved under the bind-mounted project directory, so this throwaway container has served its purpose and can be discarded.
6. On the host, move the token out of the project directory and into its permanent location:
   ```sh
   mkdir -p ~/.claude
   mv .claude-oauth-token ~/.claude/oauth-token
   chmod 600 ~/.claude/oauth-token
   ```
7. Start a fresh container to confirm it worked: `agentcubicle claude` again. If it starts straight into a session with no login prompt, the token is being picked up correctly. From now on, plain `agentcubicle claude` just works — no `--env` needed. (You can still pass `--env CLAUDE_CODE_OAUTH_TOKEN=...` explicitly to override with a different token for one run.)

Why a static token instead of just logging in fresh each time: normal OAuth credentials refresh via a single-use, rotating refresh token — copying or read-only-mounting that file into containers breaks the moment any container (or the host) refreshes it, invalidating the others. `claude setup-token` produces a long-lived (about a year), non-rotating token instead, so the same one can be reused across many container runs safely. If you'd rather use API-key billing instead of your subscription, use `--env ANTHROPIC_API_KEY="$(cat ~/.claude-key)"` instead (see the `claude code` examples below) — no token bootstrap needed for that path.

## Commits, not pushes

The workflow this tool is designed for lets an agent commit inside the container, but pushing is meant to happen outside it, by you, after reviewing what got committed — same for anything that assumes push access, like opening a PR. This is the point of git identity forwarding being an allowlist that skips `credential.helper` (see "How it works" below): push over HTTPS will not authenticate out of the box. Nothing here makes pushing technically impossible — you could still mount host SSH keys yourself (`--mount /etc/ssh:/etc/ssh:ro`, see below) and push over SSH — but that is not what this tool is set up to encourage. The spirit is that you stay the one responsible for what leaves your machine: an agent can commit freely, but review and pushing are yours to do.

You may still occasionally see the tool itself suggest pushing — Claude Code, for instance, can offer "push to origin" as a follow-up after a commit, since by default it has no idea this container is set up around a no-push workflow. `.agentcubicle/AGENTS.local.md` includes an explicit instruction telling the agent never to push or suggest it, which should suppress this in practice — but like any model instruction, that's a strong nudge, not a hard guarantee. Treat any suggestion that slips through as one to decline, not a sign the workflow expects otherwise; it typically will not authenticate anyway, for the `credential.helper` reason above.

## Disabling the Claude commit trailer on your host

`agentcubicle` does **not** suppress Claude Code's `Co-Authored-By: Claude` commit trailer for you — that's a host-level preference, not something this tool should force. If you want it gone for your own (container or non-container) Claude Code usage, set it on your real host's global config:

```sh
jq '. + {attribution: {commit: ""}}' ~/.claude/settings.json > /tmp/claude-settings.tmp \
  && mv /tmp/claude-settings.tmp ~/.claude/settings.json
```

(or create the file fresh with `{"attribution": {"commit": ""}}` if `~/.claude/settings.json` doesn't exist yet). This supersedes the deprecated `includeCoAuthoredBy` boolean. Note this must be done on the real host — editing `settings.json` from *inside* a running container doesn't persist, since the container's `$HOME` is thrown away when it exits.

## Subcommands

### `setup`

Builds the `agentcubicle` image on top of `ghcr.io/anomalyco/opencode` with a full set of development tools and bash. Use `--claude` to install Claude Code on top of the existing image (or add it if missing).

| Flag        | Description                                                  |
|-------------|--------------------------------------------------------------|
| `--clear`   | Force rebuild, removing any existing `agentcubicle` image |
| `--claude`  | Ensure Claude Code is installed; if missing, build on top of the current image |

`setup` does not require a tool argument. It always builds the base image with dev tools. `--claude` adds Claude Code as a separate layer on top of whatever image already exists — it does not rebuild the base from scratch.

### `check`

Inspects an image (default: `agentcubicle`) for all required tools plus the claude binary. Prints ✓/✗ for each package. Use `--img IMAGE` to check an arbitrary image.

```sh
agentcubicle check
agentcubicle check --img my-image
```

### `commit`

Commit the current running container as `agentcubicle`, persisting any runtime changes.

| Flag        | Description                                                    |
|-------------|----------------------------------------------------------------|
| `--name N`  | Commit a specific container by name (works for both running and stopped containers) |

Without `--name`, the script lists running `ac-*` containers interactively and asks you to pick one.

### `cleanup`

Remove all exited containers with names prefixed `ac-`.

### `list`

List all `ac-*` containers (running and stopped) with status, name, image, and creation time.

| Flag        | Description                                              |
|-------------|----------------------------------------------------------|
| `--name N`  | Filter to a specific container by name                   |

```sh
agentcubicle list
agentcubicle list --name ac-myproject-abc123
```

### `shell`

Start an interactive root shell in a running container.

| Flag        | Description                                              |
|-------------|----------------------------------------------------------|
| `--name N`  | Exec into a specific container by name                   |

Without `--name`, lists running `ac-*` containers interactively and asks you to pick one. Refuses to shell into a stopped container.

```sh
agentcubicle shell
agentcubicle shell --name ac-myproject-abc123
```

## Running a tool (default mode)

```sh
agentcubicle <opencode|claude> [options] [tool args...]
```

If no subcommand is given, a new container is started based on `agentcubicle` with the chosen tool.

### Options

| Flag        | Description                                                    |
|-------------|----------------------------------------------------------------|
| `--name N`  | Explicit container name. If the name is already in use, the existing container is force-removed and reused. |
| `--img I`   | Override the image (overrides default `agentcubicle`)       |
| `--mount P` | Extra mount, e.g. `--mount /etc/ssh:/etc/ssh:ro`. Repeatable.  |
| `--env E`   | Extra env var, e.g. `--env ANTHROPIC_API_KEY="sk-ant-..."`. Repeatable. |
| `--`        | Pass remaining arguments directly to the tool                  |

### Container naming

If no `--name` is provided, a unique name is auto-generated:

```
ac-<sanitized-project-name>-<random-hex>
```

The project name is derived from the current working directory basename, with non-alphanumeric characters replaced by dashes.

## How it works

1. **Image**: The default image is `agentcubicle`, built from `ghcr.io/anomalyco/opencode` with ~30 dev packages (via Alpine's `apk`) plus bash. Claude Code is added separately via `setup --claude`.
2. **User & home vs. project**: The container starts as root to create a `user` account matching your host UID/GID, copies tool config files into place, then drops privileges via `su`. The container's `$HOME` (`/home/user`) is throwaway scratch space that's discarded when the container exits — it is *not* the same thing as your project. Your actual project directory is bind-mounted as a clearly separate child path, `/home/user/project`, so it's never ambiguous which files are ephemeral container state and which are your real, persisted work. Files created under `/home/user/project` are owned by you on the host — no group tricks needed.
3. **Mounts**:
   - The current working directory is mounted read-write at `/home/user/project` (also the container's working directory).
   - For `opencode`: `~/.config/opencode` is mounted read-only at `/root/.config/opencode` (source for config copy).
   - For `claude`: only `~/.claude/settings.json` — not the whole directory — is mounted read-only, at `/root/.claude/settings.json`, so the oauth-token file and the raw `.credentials.json` never enter the container's mount namespace at all (skipped entirely if `settings.json` doesn't exist yet).
   - Any extra mounts from `--mount` are passed through.
   - Beyond the project directory itself, the only other things ever mounted are: whatever you explicitly add via `--mount`, and — read-write, since the underlying protocols require it — the X11/Wayland display socket and `$XAUTHORITY` when clipboard/display auto-detection kicks in (see item 11 below). Nothing else on the host is ever touched.
4. **Permissions**: containers start in a read-only **plan mode** rather than fully bypassing permissions — you review what's proposed, then manually escalate to let it actually act:
   - `opencode`: started with `--agent plan` (opencode's built-in read-only agent — edits are denied except under `.opencode/plans/*.md`), *plus* the blanket `OPENCODE_CONFIG_CONTENT={"permission":{"*":"allow"}}` override from before. That override is appended after the `plan` agent's own edit-deny rule in the resolved permission list, and opencode uses last-match-wins — so it can override the deny. In practice this makes opencode's plan mode **best-effort, not a hard guarantee** (confirmed by inspecting `opencode agent list` output; a real edit attempt wasn't tested live). Accepted trade-off. Press **Tab** in the TUI to switch to the `build` agent, which can edit and run commands.
   - `claude`: started with `--permission-mode plan` (Claude Code's built-in plan mode — read-only, no edits or command execution, with no blanket override layered on top). Press **shift+tab** in the TUI to cycle to a mode that can act, up to full bypass.
   - Your host config files are never modified.
5. **Claude Code auth**: static-token detection/forwarding and the automatic bootstrap-when-missing flow are both covered in "First-time Claude Code authentication" above, including why a static token is used instead of the rotating OAuth credentials file (`.credentials.json`), which is never mounted or copied.
6. **opencode provider detection**: before starting an opencode container, `agentcubicle` checks the host's `opencode.json`/`.jsonc` for a real `provider` entry (tolerating `.jsonc` comments via a fallback heuristic when `jq` cannot parse the file) and prints a one-line reminder if none is found — purely informational, never blocks the session. See "Configuring opencode models and providers" above for the mechanism and how to add one.
7. **First-run prompts are pre-answered (claude only)**: every container is a brand-new `$HOME`, so Claude Code would otherwise show its one-time setup wizard (theme + login method), the per-folder trust dialog, and the "Bypass Permissions mode" confirmation (should you shift+tab all the way to bypass mode) on *every single run* — even with a valid token, since none of those are gated on auth. The entrypoint stubs the minimal state needed to skip all three: `hasCompletedOnboarding` + a pre-trusted `/home/user/project` entry, and `skipDangerousModePermissionPrompt` in the persisted Claude settings (see below). Since `CLAUDE_CONFIG_DIR` is set (see next item), Claude Code resolves this stub's file at `$CLAUDE_CONFIG_DIR/.claude.json` — confirmed by reading the actual resolution logic in the Claude Code CLI itself, since it's not `$HOME/.claude.json` once that env var is set — not the fixed `$HOME`-relative location the name might suggest. This only suppresses first-run ceremony; it doesn't change what you can still do once you escalate out of plan mode.
8. **Persistent Claude Code project state (claude only)**: Claude Code normally keeps per-project memory, conversation transcripts, and worktree state under `~/.claude/projects/<encoded-path>/` — but since every container gets a fresh throwaway `$HOME`, that would otherwise be wiped on every run, even though your project itself persists fine. To fix this without ever giving the container host access outside the project directory, `CLAUDE_CONFIG_DIR` is pointed at `/home/user/project/.agentcubicle/claude-home` — a directory *inside* the already read-write project mount. This is distinct from the project's own committed `.claude/` directory (CLAUDE.md, `.claude/settings.json`, `.claude/agents/`, `.claude/skills/`, etc.) — that one is meant to be checked into git and shared with your team; `.agentcubicle/` is per-machine session state that happens to live next to it. It's kept out of git via the repo's local `.git/info/exclude` (not `.gitignore` — an entry there is tracked and would be visible to anyone who clones the repo; `.git/info/exclude` behaves identically for git's purposes but is never committed or shared).
9. **Git identity forwarding**: a curated, safe subset of your host's `git config --global` settings — `user.name`, `user.email`, `init.defaultBranch`, `pull.rebase`, `push.default`, `push.autoSetupRemote`, `core.editor`, `color.ui`, `rerere.enabled`, `merge.conflictstyle`, and all `alias.*` entries — is forwarded into the container so commits made by an agent have correct authorship. This is an allowlist, not a blocklist: `credential.helper` (host keychain/manager helpers won't exist or function in the container) and any GPG/commit-signing settings (`user.signingkey`, `gpg.*`, `commit.gpgsign`, `tag.gpgsign` — no private key material is available in the container) are never forwarded. Path-valued settings like `core.excludesfile`/`core.hooksPath` are also skipped, since honoring them would require mounting a host file outside the project directory.
10. **Local agent rules and memory, read by both tools**: every run, `agentcubicle` maintains three files under `.agentcubicle/` in your project — `AGENTS.local.md` (general, tool-agnostic rules), `MEMORY.local.md` (a running account of current focus, recent decisions, and open threads, referenced from `AGENTS.local.md`), and `CLAUDE.local.md` (Claude-Code-*specific-only* instructions, which imports the other two via `@` includes). The `.local.md` suffix (matching the `CLAUDE.local.md` root symlink below) marks them unambiguously as agentcubicle-managed, so they never collide with a bare `AGENTS.md`/`CLAUDE.md`/`MEMORY.md` your project might track itself. These are seeded with a starter template the first time, then left alone — edits persist and are never overwritten. This is **additive**: it applies even if your project already has its own tracked `AGENTS.md`/`CLAUDE.md`, which continue to load normally through each tool's own discovery. Projects seeded by an earlier version (which used the bare names) are migrated automatically on the next run — the legacy `.agentcubicle/*.md` files are renamed to `*.local.md`, their `@` imports and the root symlink are repointed, and a one-line notice is printed; the move is lossless and one-time.
   - `opencode` reads `.agentcubicle/AGENTS.local.md` and `.agentcubicle/MEMORY.local.md` directly via its `instructions` config list, merged in through the container-only `OPENCODE_CONFIG_CONTENT` override (see above) — never by writing to your real `opencode.json`/`.jsonc`.
   - `claude` reads `.agentcubicle/CLAUDE.local.md` via `CLAUDE.local.md` at the project root, a symlink Claude Code loads automatically every session (matching its own documented `CLAUDE.md` symlink convention). `CLAUDE.local.md` is a strong, tooling-enforced convention for personal/never-committed content, so it's not at risk of colliding with a file the repository might track later. That `CLAUDE.local.md` in turn pulls `AGENTS.local.md` and `MEMORY.local.md` in via Claude Code `@` imports, so all three are inlined into the session rather than relying on the model to go read them — the imports resolve correctly even through the `CLAUDE.local.md` symlink.
   - Deliberately, nothing is ever placed at the literal `AGENTS.md`/`CLAUDE.md` paths themselves: even a `.git/info/exclude`d symlink there would risk breaking a future `git pull` if the upstream remote ever adds a real tracked file at that exact path — git refuses to silently overwrite an untracked/excluded file during a merge, exclude or not.
   - `.agentcubicle/` and `CLAUDE.local.md` are both registered in the repo's local `.git/info/exclude` (not `.gitignore`, for the same leak-prevention reason as the persisted Claude state above).
   - Both `AGENTS.local.md` and `CLAUDE.local.md` ask the tool to state, at the start of every session, which local files it read and a one-paragraph summary of what it picked up — but an interactive session never gives the model an unprompted turn to say anything until it receives input. So every run also auto-sends a short kickoff prompt (via opencode's `--prompt` / Claude Code's positional `[prompt]` argument, both of which submit an initial message without leaving interactive mode) asking it to do exactly that as its first reply. Any of your own extra args are still respected — `--prompt`/the positional prompt argument just becomes the tool's first turn.
11. **Clipboard / Display**: Auto-detected based on environment:
    - **Wayland**: If `WAYLAND_DISPLAY` and `XDG_RUNTIME_DIR` are set, the Wayland socket is mounted and made available.
    - **X11**: If `DISPLAY` is set, `/tmp/.X11-unix` is mounted and `XAUTHORITY` is forwarded if present.
12. **Git**: `/home/user/project` is registered as a safe directory so git operations work without warnings.

## Installed packages

The following packages are installed in the `agentcubicle` image:

| Category     | Packages                                                                                                                                                       |
|--------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Editors**  | `vim`, `nano`                                                                                                                                                  |
| **Compilers**| `gcc`, `g++`, `build-base` (make, etc.)                                                                                                                        |
| **Debuggers**| `gdb`, `gdb-dashboard`, `cgdb`, `valgrind`, `strace`                                                                                                           |
| **Git**      | `git`                                                                                                                                                          |
| **CLI tools**| `bat`, `diffutils`, `file`, `fzf`, `jq`, `lsof`, `patch`, `perl`, `sqlite`, `shellcheck`, `tree`                                                              |
| **Network**  | `curl`, `curlie`, `httpie`, `nmap`, `netcat-openbsd`, `tcpdump`, `wrk`                                                                                         |
| **Languages**| `lua5.4`, `luajit`, `nodejs`, `python3`, `py3-pip`                                                                                                             |
| **System**   | `htop`, `tmux`, `bash`                                                                                                                                         |
| **Clipboard**| `wl-clipboard`, `xclip`                                                                                                                                        |
| **AI tools** | `opencode` (from base image), `claude` (optional, installed via `setup --claude`)                                                                              |

## Examples

### Setup

```sh
# Base image with dev tools
agentcubicle setup

# Add Claude Code
agentcubicle setup --claude

# Rebuild everything from scratch
agentcubicle setup --clear
```

### opencode

```sh
# Provide a provider API key referenced in your opencode.json as {env:VAR}
# (see "Configuring opencode models and providers" above)
agentcubicle opencode --env ANTHROPIC_API_KEY="$(cat ~/.anthropic-key)"

# Run opencode with extra mounts and a GitHub token
agentcubicle opencode \
  --env GITHUB_TOKEN="$(cat github-PAT)" \
  --mount /etc/ssh:/etc/ssh:ro

# Run a specific named container
agentcubicle opencode --name my-session

# Pass arguments to opencode
agentcubicle opencode -- --model claude-sonnet-4
```

### claude code

```sh
# Just works: auto-detects an existing token, or bootstraps one for you
# on first run (see "First-time Claude Code authentication" above).
agentcubicle claude

# Alternative: API-key billing instead of your subscription
agentcubicle claude --env ANTHROPIC_API_KEY="$(cat ~/.claude-key)"

# Run claude code with extra mounts
agentcubicle claude --mount /etc/ssh:/etc/ssh:ro

# Pass arguments to claude
agentcubicle claude -- --model sonnet
```

### Management

```sh
# Check image for installed tools
agentcubicle check
agentcubicle check --img my-image

# Commit the running container to persist changes
agentcubicle commit --name ac-myproject-abc123

# Interactive container selection for commit
agentcubicle commit

# Clean up stopped containers
agentcubicle cleanup

# List all containers
agentcubicle list

# Interactive shell into a running container
agentcubicle shell

# Shell into a specific container
agentcubicle shell --name ac-myproject-abc123
```

## Requirements

- Docker
- `jq`
- Current directory as the project workspace
