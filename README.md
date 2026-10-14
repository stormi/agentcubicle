# agentcubicle

Run opencode or claude code in an isolated container (Docker, or rootless Podman). Project files are mounted read-write; sessions start in a read-only plan mode that you manually escalate out of once you're ready to let it act.

## Why another tool

Well, I didn't know how to choose between all the "isolate your agents" projects, and had a few needs I wanted to be taken care of in priority.

One of my goals was the ability to run an agent from any directory, without extra configuration, and to keep local-to-me context for the next sessions.

I'm sharing the result in case it's useful to anyone, but I don't make any promises about its usefulness. Nor about its security: container sandboxing can be escaped, it's a protection against the most obvious mistakes, but not real security to protect your computer and data.

## Features

`agentcubicle` runs `opencode` or Claude Code in a disposable Docker sandbox that attempts to stay out of your way. It's non-intrusive by design: the agent rules and working context it manages live under `.agentcubicle/` and are kept out of git (via `.git/info/exclude`). So nothing of your tooling-specific rules pollutes the project you're working on, unless you decide otherwise. The first time you start it on a project, it seeds a starter set of base rules you can then edit freely. A shared local memory file allows context to flow both ways between agents. Notes opencode leaves behind are picked up by Claude Code, and vice versa.

- Commit-only workflow: agents can commit inside the container (and will usually ask first when they respect the default rules), but pushing/PRs are left to you on the host. `git` identity forwarding is an allowlist that deliberately skips `credential.helper`.
- Starts every session in a read-only plan mode you manually escalate out of (`opencode --agent plan`; Claude `--permission-mode plan`).
- Containers are removed when the tool exits, so they don't pile up on disk. Pass `--keep-container` to keep one around (to `commit` it afterwards, say).
- Tool auto-updates are disabled inside the container: an update cannot outlive a container that is removed on exit, and the restart it asks for cannot happen. Tool versions come from the image and are refreshed by rebuilding it.
- Minimal mounts: project dir (rw), plus tool config read-only (`~/.config/opencode` for opencode; only `~/.claude/settings.json` for Claude, never the OAuth/credentials files).
- Named shares: host directories you create once (`agentcubicle share create NAME`) and enable per container (`--share NAME` or `--share NAME:ro`), so two containers can exchange files without either seeing the other's project. `--share-dir PATH` mounts any host directory the same way for one session, read-only by default (for example, part of another agent's workspace).
- Files you create come out owned by you on the host: under Docker via a root-phase UID/GID remap, under rootless Podman via `--userns=keep-id`.
- Runs on Docker or rootless Podman. The engine is auto-detected (Podman preferred when both are installed); set `AGENTCUBICLE_ENGINE=docker` or `=podman` to override.
- Auto-detected clipboard/display passthrough (Wayland socket or X11 `/tmp/.X11-unix` + `XAUTHORITY`). Works well for me with opencode's "selecting text copies it" behaviour. The Podman path is newer and less exercised here than the Docker one.

#### Claude-specific

- Persistent per-project state: `CLAUDE_CONFIG_DIR` points inside the project mount (`.agentcubicle/claude-home`) so memory/transcripts survive across runs.

#### Shell tools

- The image bundles ~30 dev packages: compilers, debuggers, CLI tools useful to work with text files and code, network tools, language runtimes...

### How rules and context are layered

agentcubicle loads instructions in layers, from the shared documents your whole team sees down to the private, machine-local notes only you (and your agent) see. Each layer is additive: a lower layer never overwrites or hides the one above it.

**1. Your project's own tracked instructions** *(shared, in git)*
Any `AGENTS.md` or `CLAUDE.md` your repository already tracks keeps loading exactly as it normally would, through each tool's native discovery.

**2. Local base rules** *(private, git-excluded)*
On first run in a project, agentcubicle seeds two rule files under `.agentcubicle/`, then leaves them yours to edit:
- `AGENTS.local.md`: general, tool-agnostic rules read by every agent.
- `CLAUDE.local.md`: Claude-Code-specific rules only; it `@`-imports the other files so Claude gets everything in one load.

**3. Shared local memory** *(private, git-excluded)*
`MEMORY.local.md` is a single running notebook (current focus, recent decisions, open threads) that **both** opencode and Claude Code read and update. It's a way to carry context from one tool to the other.

**4. Claude persistent project state** *(private, git-excluded)*
`.agentcubicle/claude-home/` (Claude's `CLAUDE_CONFIG_DIR`) holds per-project memory, conversation transcripts, and a one-time seeded copy of your settings, so Claude's own state survives across throwaway containers.

**How each tool wires them in**
- **opencode** reads `AGENTS.local.md` + `MEMORY.local.md` through its `instructions` config list (injected via a container-only config override. Your real `opencode.json` is never edited).
- **Claude Code** loads a root-level `CLAUDE.local.md` symlink automatically each session, which pulls in `AGENTS.local.md` and `MEMORY.local.md` via `@`-imports.

Everything in layers 2–4 stays out of git (via `.git/info/exclude`, not `.gitignore`), so none of it leaks to pollute the original repo.

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

`opencode`'s own config already propagates from your host: `~/.config/opencode/opencode.json` (or `.jsonc`) is mounted read-only and copied into the container on every run (see "How it works" below). Nothing extra to set up.

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

The same `{env:VAR}` + `--env` pattern works for a hosted vendor too. opencode ships built-in support for the common ones, so the config is just an API key:

```jsonc
"provider": { "anthropic": { "options": { "apiKey": "{env:ANTHROPIC_API_KEY}" } } }
```

Note that `opencode auth login` does **not** work for this: it stores credentials at `~/.local/share/opencode/auth.json`, which agentcubicle never mounts or copies into the container (only `~/.config/opencode` is), so anything saved there never reaches a session. The config-file-plus-`--env` approach above is the one that actually works, and is verified against a real `opencode debug config` resolution.

**Literal keys are handled automatically too.** If you'd rather just keep a literal key in your `opencode.json` (as many personally-managed configs do), you don't have to convert it to `{env:VAR}` yourself: agentcubicle detects any literal `provider.*.options.apiKey`, forwards the real value into the container as a generated `AC_OPENCODE_<PROVIDER>_APIKEY` env var, and gives the container a redacted copy of the config that references it. So the literal never lands in the container's config copy (nor in your shell history, since it's read from the file rather than typed into `--env`). This applies only when the config is valid JSON `jq` can parse; a `.jsonc` with comments is left as-is, keeping its literal key.

If your config has no `provider` section at all, `agentcubicle opencode` prints a reminder about this every time you start a session.

## First-time Claude Code authentication

The container never receives your host's live Claude Code credentials: only `~/.claude/settings.json` itself is ever mounted read-only and copied out (see "How it works" below). If you already have a static token at `~/.claude/oauth-token` (or exported as `CLAUDE_CODE_OAUTH_TOKEN`, or an `ANTHROPIC_API_KEY` passed with `--env`), there's nothing to do here: `agentcubicle claude` auto-detects and forwards it for you automatically, no bootstrap needed.

If none of those are found, `agentcubicle claude` detects this itself and, instead of starting a normal session that would just hit a login wall, runs a token bootstrap:

1. Build the image if you haven't yet: `agentcubicle setup --claude`
2. Run `agentcubicle claude`. The container runs `claude setup-token`, which prints a URL.
3. Open the URL in a browser, log into your Claude account, and authorize. The browser session decides which account the token belongs to, so use a private window if you are logged into a different account there.
4. `claude setup-token` prints a long-lived token. Paste it at the prompt that follows (input is hidden) and press Enter.
5. The container exits, and agentcubicle saves the token on the host to `~/.claude/oauth-token` (mode 600). The container never has write access to `~/.claude`: the token passes through a temporary file in the project's `.agentcubicle/` directory, which is moved into place and removed once the container exits.
6. Run `agentcubicle claude` again. It starts straight into a session; from now on, plain `agentcubicle claude` picks the token up with no `--env` needed. (You can still pass `--env CLAUDE_CODE_OAUTH_TOKEN=...` explicitly to override it for one run.)

If you press Enter without pasting anything, the bootstrap prints the manual steps instead and leaves you in a shell to do them: write the token into the bind-mounted project directory (`echo "<token>" > /home/user/project/.claude-oauth-token`), exit, then on the host `mkdir -p ~/.claude && mv .claude-oauth-token ~/.claude/oauth-token && chmod 600 ~/.claude/oauth-token`.

To replace a token (expired, revoked, or for a different account), delete the file and run `agentcubicle claude` again.

### Using another Claude account

`--account NAME` selects a different token file, `~/.claude/oauth-token-NAME`, so you can keep several accounts side by side and choose one when the container starts:

```sh
agentcubicle claude --account perso   # first run: token bootstrap, saved to ~/.claude/oauth-token-perso
agentcubicle claude --account perso   # later runs: session with that account
agentcubicle claude                   # unchanged: ~/.claude/oauth-token
```

With `--account`, only that account's file is used: a `CLAUDE_CODE_OAUTH_TOKEN` exported in your shell is ignored, so it can't silently replace the account you named. An explicit `--env CLAUDE_CODE_OAUTH_TOKEN=...` still wins. Names may only contain letters, digits, `-` and `_`. The account only changes which token is forwarded: the project's `.agentcubicle/claude-home/` (settings, memory, transcripts) is the same whichever account you use.

Why a static token instead of just logging in fresh each time: normal OAuth credentials refresh via a single-use, rotating refresh token, and copying or read-only-mounting that file into containers breaks the moment any container (or the host) refreshes it, invalidating the others. `claude setup-token` produces a long-lived (about a year), non-rotating token instead, so the same one can be reused across many container runs. If you'd rather use API-key billing instead of your subscription, use `--env ANTHROPIC_API_KEY="$(cat ~/.claude-key)"` instead (see the `claude code` examples below). No token bootstrap needed for that path.

## Commits, not pushes

The workflow this tool is designed for lets an agent commit inside the container, but pushing is meant to happen outside it, by you, after reviewing what got committed. The same goes for anything that assumes push access, like opening a PR. This is the point of git identity forwarding being an allowlist that skips `credential.helper` (see "How it works" below): push over HTTPS will not authenticate out of the box. Nothing here makes pushing technically impossible; you could still mount host SSH keys yourself (`--mount /etc/ssh:/etc/ssh:ro`, see below) and push over SSH, but that is not what this tool is set up to encourage. The spirit is that you stay the one responsible for what leaves your machine: an agent can commit freely, but review and pushing are yours to do.

You may still occasionally see the tool itself suggest pushing. Claude Code, for instance, can offer "push to origin" as a follow-up after a commit, since by default it has no idea this container is set up around a no-push workflow. `.agentcubicle/AGENTS.local.md` includes an explicit instruction telling the agent never to push or suggest it, which should suppress this in practice, but like any model instruction, that's a strong nudge, not a hard guarantee. Treat any suggestion that slips through as one to decline, not a sign the workflow expects otherwise; it typically will not authenticate anyway, for the `credential.helper` reason above.

## Disabling the Claude commit trailer on your host

`agentcubicle` does **not** suppress Claude Code's `Co-Authored-By: Claude` commit trailer for you; that's a host-level preference, not something this tool should force. If you want it gone for your own (container or non-container) Claude Code usage, set it on your real host's global config:

```sh
jq '. + {attribution: {commit: ""}}' ~/.claude/settings.json > /tmp/claude-settings.tmp \
  && mv /tmp/claude-settings.tmp ~/.claude/settings.json
```

(or create the file fresh with `{"attribution": {"commit": ""}}` if `~/.claude/settings.json` doesn't exist yet). This supersedes the deprecated `includeCoAuthoredBy` boolean. Note this must be done on the real host: editing `settings.json` from *inside* a running container doesn't persist, since the container's `$HOME` is thrown away when it exits.

## Claude Code settings that follow the host

Your host `~/.claude/settings.json` is copied into a project's `claude-home` once, at
first seed, and the persisted copy owns it from then on (see item 8 under "How it works").
A short allowlist of keys is exempt: they are re-read from the host and written into the
persisted copy on **every** `agentcubicle claude` run, overwriting whatever was there,
including a value a previous session set. The list is `CLAUDE_PROPAGATE_KEYS` near the top
of the run path in the script, and currently holds one key:

| Key          | Notes                                                                 |
|--------------|-----------------------------------------------------------------------|
| `statusLine` | The `command` has to be inline; see below                             |

The point is that a setting you think of as belonging to you rather than to a given
container (how the status line looks, say) can be edited in one place on the host and
show up in every container on its next run, without deleting any
`claude-home/settings.json`.

The container is a different filesystem, so only an **inline** command works there:

```json
{
  "statusLine": {
    "type": "command",
    "command": "jq -r '\"[\\(.model.display_name)] \\(.workspace.current_dir)\"'"
  }
}
```

A `command` whose first word is a path (`~/.claude/statusline.sh`,
`/usr/local/bin/statusline`) points at a host script that does not exist in the container,
so `agentcubicle` prints a one-line warning and leaves the container's own value alone.
The check only looks at that first word: an inline command can still call something the
container does not have, and that is not detected. `jq`, `python3`, `awk` and the rest of
the image's tooling are available.

Inside the container the project is always mounted at `/home/user/project`, so
`.workspace.current_dir` reads the same for every project, and a project that is not a git
repo has no `.workspace.repo.name`. `agentcubicle` sets `AGENTCUBICLE_PROJECT_NAME` to the
name of the host directory you ran it from, which a status line can fall back on:

```json
{
  "statusLine": {
    "type": "command",
    "command": "jq -r '(.workspace.repo.name // \"\" | if . == \"\" then (env.AGENTCUBICLE_PROJECT_NAME // \"\") else . end) + \" [\\(.model.display_name)]\"'"
  }
}
```

The variable is set for both tools, and `--env AGENTCUBICLE_PROJECT_NAME=...` overrides it.

### Optional: a fuller example status line

Nothing in `agentcubicle` depends on this. It is one status line you can copy into your
host `~/.claude/settings.json` if you want it. It needs only `jq`, so the same value works
on the host and in the container. It shows the project name (the repo name, falling back
to `AGENTCUBICLE_PROJECT_NAME`), the model, context usage, and the 5-hour and 7-day rate
limit usage:

```text
myproject [Opus 5.5] Context: 23% (234k/1000k) | 5h: 12% | 7d: 41%
```

```json
{
  "statusLine": {
    "type": "command",
    "command": "jq -r 'def k: (./1000) as $v | if $v == ($v|floor) then \"\\($v|floor)k\" else \"\\(($v*10|round)/10)k\" end; def pct($l): if . == null then \"\\($l): n/a\" else \"\\($l): \\(round)%\" end; def seg($f): (. // \"\") | if . == \"\" then \"\" else $f end; ((.workspace.repo.name // \"\" | if . == \"\" then (env.AGENTCUBICLE_PROJECT_NAME // \"\") else . end) | seg(. + \" \")) + ((.model.display_name) | seg(\"[\" + . + \"] \")) + (if .context_window.used_percentage == null then \"Context: n/a\" else \"Context: \\(.context_window.used_percentage|round)%\" + (if .context_window.total_input_tokens != null and .context_window.context_window_size != null then \" (\\(.context_window.total_input_tokens|k)/\\(.context_window.context_window_size|k))\" else \"\" end) end) + \" | \" + (.rate_limits.five_hour.used_percentage | pct(\"5h\")) + \" | \" + (.rate_limits.seven_day.used_percentage | pct(\"7d\"))'"
  }
}
```

A segment whose data Claude Code does not provide reads `n/a`, and the project name is
left out when there is neither a repo name nor the variable (on the host, outside a repo).

## Tool versions in the container

Both tools ship their own background updater, and neither can work here. The container is
removed when the tool exits, so a downloaded update is thrown away with it, and the tool
runs via `exec` as a direct child of the entrypoint shell, so the restart it asks for has
nothing to restart into. Left alone, Claude Code re-downloads the same ~220MB on every
single run and then asks to be restarted.

So `agentcubicle` turns both off inside the container:

- Claude Code: `DISABLE_AUTOUPDATER=1` on the run path. This only stops the background
  check, so `claude update` still works inside a container if you want a one-off (it is
  discarded when the container exits, like any other change outside the project mount).
  To get the old behaviour back for one run, pass `--env DISABLE_AUTOUPDATER=0`.
- opencode: `"autoupdate": false` in the container-only config. Its binary is root-owned
  and the tool runs unprivileged, so a self-update could not have succeeded anyway.

The version you run is therefore the version the image was built with. `agentcubicle check`
prints it for `claude`, `opencode` and `gh`. To refresh them, rebuild on the host:

```sh
agentcubicle setup --clear --claude
```

`--clear` is what makes this a rebuild: a plain `setup` over an existing image reports
"Image is ready" and a plain `setup --claude` reports "Claude Code is already installed",
neither of which re-runs an installer. Pass both flags together: `--clear` on its own
rebuilds the image *without* Claude Code, and you would need a second `setup --claude` to
put it back. `opencode` comes from the base image, so it is refreshed by the same rebuild
pulling a newer base.

## Subcommands

`agentcubicle --help` lists the commands, and every subcommand accepts `-h`/`--help` to print its usage. Help works without a container engine installed.

### `setup`

Builds the `agentcubicle` image on top of `ghcr.io/anomalyco/opencode` with a full set of development tools and bash. Use `--claude` to install Claude Code on top of the existing image (or add it if missing).

| Flag        | Description                                                  |
|-------------|--------------------------------------------------------------|
| `--clear`   | Force rebuild, removing any existing `agentcubicle` image |
| `--claude`  | Ensure Claude Code is installed; if missing, build on top of the current image |

`setup` does not require a tool argument. It always builds the base image with dev tools. `--claude` adds Claude Code as a separate layer on top of whatever image already exists; it does not rebuild the base from scratch.

### `check`

Inspects an image (default: `agentcubicle`) for all required tools plus the claude binary. Prints ✓/✗ for each package. `claude`, `opencode` and `gh` are reported with their version, since auto-updates are off inside the container (see [Tool versions in the container](#tool-versions-in-the-container)) and the image is what pins them. A missing `opencode` is reported but not treated as a failure: it comes from the base image, and `setup` cannot install it. Use `--img IMAGE` to check an arbitrary image.

```sh
agentcubicle check
agentcubicle check --img my-image
```

### `commit`

Commit the current running container as `agentcubicle`, persisting any runtime changes.

| Flag        | Description                                                    |
|-------------|----------------------------------------------------------------|
| `--name N`  | Commit a specific container by name (running or stopped) |

Without `--name`, the script lists running `ac-*` containers interactively and asks you to pick one.

Containers are removed when the tool exits (see [Container lifetime](#container-lifetime)), so committing a *stopped* container only works if it was started with `--keep-container`. Committing a container while it is still running needs nothing special: run `agentcubicle commit` from a second terminal.

### `cleanup`

Remove all exited containers with names prefixed `ac-`. Since containers are removed on exit, this applies to ones started with `--keep-container`, plus any left behind by an older version of the script.

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

### `share`

Manage named shares: plain host directories that you enable for a container with `--share` (see [Sharing files between containers](#sharing-files-between-containers)). Works without a container engine installed.

| Command            | Description                                                 |
|--------------------|-------------------------------------------------------------|
| `share create N`   | Create share `N` (no-op if it exists) and print its host path |
| `share list`       | List shares with their host paths                           |
| `share rm N`       | Remove share `N`. Refuses if it is not empty: delete its contents yourself first |

Share names use letters, digits, `.`, `_` and `-`, starting with a letter or digit.

```sh
agentcubicle share create xchg
agentcubicle share list
agentcubicle share rm xchg
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
| `--share S` | Mount the named share `S` at `/shares/S`, read-write by default; `S:ro` mounts it read-only. The share must exist (`share create`). Repeatable. |
| `--share-dir D` | Mount host directory `D` at `/shares/<basename of D>` for this session, **read-only by default**; `D:rw` mounts it read-write. Repeatable. |
| `--env E`   | Extra env var, e.g. `--env ANTHROPIC_API_KEY="sk-ant-..."`. Repeatable. |
| `--keep-container` | Keep the container after the tool exits. Without it, the container is removed on exit. |
| `--account A` | `claude` only: use the token in `~/.claude/oauth-token-A` instead of `~/.claude/oauth-token` (see [Using another Claude account](#using-another-claude-account)). |
| `-h`, `--help` | Show agentcubicle's options for running the tool, without starting a container. For the tool's own help, use `agentcubicle claude -- --help`. |
| `--`        | Pass remaining arguments directly to the tool                  |

### Sharing files between containers

To let two containers exchange files, create a named share once and enable it in each container that should see it:

```sh
agentcubicle share create xchg

# In project A: may read and write the share
agentcubicle claude --share xchg

# In project B: may only read it
agentcubicle opencode --share xchg:ro
```

Both containers see the share at `/shares/xchg`. On the host it is the directory `${XDG_DATA_HOME:-~/.local/share}/agentcubicle/shares/xchg`, outside either project, so nothing is added to your repositories, and you can inspect or edit what is in it from the host.

To give an agent temporary access to a directory that is not a named share, typically part of another agent's workspace, use `--share-dir` when you start it:

```sh
# Agent B sees project A's docs/ live, read-only, until B exits
agentcubicle claude --share-dir ~/src/projA/docs

# Read-write instead
agentcubicle claude --share-dir ~/src/projA/docs:rw
```

The directory is mounted at `/shares/<basename>` (here `/shares/docs`); two shares that would land on the same path are refused. It is a live view, not a copy: B sees A's changes as they happen, and with `:rw` A sees B's. Nothing is created on the host, and access ends when the container exits. A running container cannot be given a new mount, so B has to be started (or restarted) with the flag. Any directory works, except `/` and paths containing `:` or `,`, which the engine's mount syntax cannot express.

Things to know:

- Shares are mounted only into containers you start with `--share` or `--share-dir`.
- Agents start in `/home/user/project`, and tools can restrict access outside it. For Claude Code, each enabled share is also passed as `--add-dir /shares/NAME`, so it can use the share without a permission prompt for being outside the project (it also sees the share listed among its working directories). opencode's permission config generated by agentcubicle already allows paths outside the project, so it needs nothing extra. Neither tool is told what a share is for: mention `/shares/NAME` in your instructions or prompt if you want it used.
- Both containers run as your host user, so file ownership does not separate them. The only access control is `:ro` versus the default read-write, set per container. Any container you enable a share for can read everything in it.
- There is no locking or notification: if both sides write, they can overwrite each other's files. A simple convention (one writer per file, or one share per direction) avoids that.

### Container lifetime

When the tool exits, the container is removed. Nothing is lost by that: your work
lives in the bind-mounted project directory, as does the persisted Claude Code
state under `.agentcubicle/claude-home` (which already survived container
recreation before, since a new container is created on every run anyway).
Changes you want to keep *inside* the container (a package you installed, say)
are kept by committing them into the image with [`commit`](#commit). Containers
that stay around only take disk space.

`--keep-container` opts out and leaves the container in place after it exits. The
one thing that needs it is committing a container *after* the fact rather than
while it is still running; `agentcubicle cleanup` removes such containers later.
Note that `--name` on its own does not keep the container: it only gives the
container a fixed name so `shell` and `commit` can find it during the session.

### Container naming

If no `--name` is provided, a unique name is auto-generated:

```
ac-<sanitized-project-name>-<random-hex>
```

The project name is derived from the current working directory basename, with non-alphanumeric characters replaced by dashes.

## How it works

1. **Image**: The default image is `agentcubicle`, built from `ghcr.io/anomalyco/opencode` with ~30 dev packages (via Alpine's `apk`) plus bash. GitHub CLI (`gh`) has no apk package, so it is installed instead from an official release tarball. Claude Code is added separately via `setup --claude`. Packages you list in `~/.config/agentcubicle/packages` are installed too (see "Extra packages").
2. **User & home vs. project**: Under Docker the container starts as root to create a `user` account matching your host UID/GID, copies tool config files into place, then drops privileges via `su`. Under rootless Podman there is no root phase: `--userns=keep-id` maps your host user directly onto the container's `user`, so files land owned by you without a remap (and `--security-opt label=disable` keeps SELinux hosts from relabeling your project). Either way, the container's `$HOME` (`/home/user`) is throwaway scratch space that's discarded when the container exits (as is the container itself, unless you pass `--keep-container`); it is *not* the same thing as your project. Your actual project directory is bind-mounted as a clearly separate child path, `/home/user/project`, so it's never ambiguous which files are ephemeral container state and which are your real, persisted work. Files created under `/home/user/project` are owned by you on the host.
3. **Mounts**:
   - The current working directory is mounted read-write at `/home/user/project` (also the container's working directory).
   - For `opencode`: `~/.config/opencode` is mounted read-only as the source for a config copy (at `/root/.config/opencode` under Docker; under Podman, where the unprivileged user cannot read `/root`, it is staged under `/home/user/.config-host/opencode` instead). Skipped when no opencode config exists.
   - For `claude`: only `~/.claude/settings.json` (not the whole directory) is mounted read-only, so the oauth-token file and the raw `.credentials.json` never enter the container's mount namespace at all (skipped entirely if `settings.json` doesn't exist yet).
   - Any extra mounts from `--mount` are passed through.
   - Each `--share NAME` mounts the host directory `${XDG_DATA_HOME:-~/.local/share}/agentcubicle/shares/NAME` at `/shares/NAME`, with the mode you gave (read-write by default). Each `--share-dir D` mounts `D` at `/shares/<basename of D>` (read-only by default).
   - Beyond the project directory itself, the only other things ever mounted are: whatever you explicitly add via `--mount`, `--share` or `--share-dir`, and (read-write, since the underlying protocols require it) the X11/Wayland display socket and `$XAUTHORITY` when clipboard/display auto-detection kicks in (see item 11 below). Nothing else on the host is ever touched.
4. **Permissions**: containers start in a read-only **plan mode** rather than fully bypassing permissions: you review what's proposed, then manually escalate to let it actually act:
   - `opencode`: started with `--agent plan` (opencode's built-in read-only agent; edits are denied except under `.opencode/plans/*.md`), *plus* the blanket `OPENCODE_CONFIG_CONTENT={"permission":{"*":"allow"}}` override from before. That override is appended after the `plan` agent's own edit-deny rule in the resolved permission list, and opencode uses last-match-wins, so it can override the deny. In practice this makes opencode's plan mode **best-effort, not a hard guarantee** (confirmed by inspecting `opencode agent list` output; a real edit attempt wasn't tested live). Accepted trade-off. Press **Tab** in the TUI to switch to the `build` agent, which can edit and run commands.
   - `claude`: started with `--permission-mode plan` (Claude Code's built-in plan mode: read-only, no edits or command execution, with no blanket override layered on top). Press **shift+tab** in the TUI to cycle to a mode that can act, up to full bypass.
   - Your host config files are never modified.
5. **Claude Code auth**: static-token detection/forwarding and the automatic bootstrap-when-missing flow are both covered in "First-time Claude Code authentication" above, including why a static token is used instead of the rotating OAuth credentials file (`.credentials.json`), which is never mounted or copied.
6. **opencode provider detection**: before starting an opencode container, `agentcubicle` checks the host's `opencode.json`/`.jsonc` for a real `provider` entry (tolerating `.jsonc` comments via a fallback heuristic when `jq` cannot parse the file) and prints a one-line reminder if none is found; purely informational, never blocks the session. See "Configuring opencode models and providers" above for the mechanism and how to add one.
7. **First-run prompts are pre-answered (claude only)**: every container is a brand-new `$HOME`, so Claude Code would otherwise show its one-time setup wizard (theme + login method), the per-folder trust dialog, and the "Bypass Permissions mode" confirmation (should you shift+tab all the way to bypass mode) on *every single run*, even with a valid token, since none of those are gated on auth. The entrypoint stubs the minimal state needed to skip all three: `hasCompletedOnboarding` + a pre-trusted `/home/user/project` entry, and `skipDangerousModePermissionPrompt` in the persisted Claude settings (see below). Since `CLAUDE_CONFIG_DIR` is set (see next item), Claude Code resolves this stub's file at `$CLAUDE_CONFIG_DIR/.claude.json` (confirmed by reading the actual resolution logic in the Claude Code CLI itself, since it's not `$HOME/.claude.json` once that env var is set), not the fixed `$HOME`-relative location the name might suggest. This only suppresses first-run ceremony; it doesn't change what you can still do once you escalate out of plan mode.
8. **Persistent Claude Code project state (claude only)**: Claude Code normally keeps per-project memory, conversation transcripts, and worktree state under `~/.claude/projects/<encoded-path>/`, but since every container gets a fresh throwaway `$HOME`, that would otherwise be wiped on every run, even though your project itself persists fine. To fix this without ever giving the container host access outside the project directory, `CLAUDE_CONFIG_DIR` is pointed at `/home/user/project/.agentcubicle/claude-home`, a directory *inside* the already read-write project mount. This is distinct from the project's own committed `.claude/` directory (CLAUDE.md, `.claude/settings.json`, `.claude/agents/`, `.claude/skills/`, etc.): that one is meant to be checked into git and shared with your team, whereas `.agentcubicle/` is per-machine session state that happens to live next to it. It's kept out of git via the repo's local `.git/info/exclude` (not `.gitignore`, since an entry there is tracked and would be visible to anyone who clones the repo; `.git/info/exclude` behaves identically for git's purposes but is never committed or shared). One consequence worth knowing: your host `~/.claude/settings.json` is copied into `claude-home` only *once*, the first time a container is seeded. After that the persisted copy wins, so anything you change *inside* a claude session (a `/model` choice, for example) sticks for that container on every future run, while later edits to your host `settings.json` do **not** propagate back into an already-seeded container (different containers can thus end up on different models). To make a container pick up fresh host settings, delete its `.agentcubicle/claude-home/settings.json` so it re-seeds on the next run. This is unlike opencode, whose config is re-read from the host on every run. The one exception is a short allowlist of keys (currently just `statusLine`) that is re-read from the host and overwritten in the persisted copy on every run; see "Claude Code settings that follow the host" above.
9. **Git identity forwarding**: a limited subset of your host's `git config --global` settings (`user.name`, `user.email`, `init.defaultBranch`, `pull.rebase`, `push.default`, `push.autoSetupRemote`, `core.editor`, `color.ui`, `rerere.enabled`, `merge.conflictstyle`, and all `alias.*` entries) is forwarded into the container so commits made by an agent have correct authorship. This is an allowlist, not a blocklist: `credential.helper` (host keychain/manager helpers won't exist or function in the container) and any GPG/commit-signing settings (`user.signingkey`, `gpg.*`, `commit.gpgsign`, `tag.gpgsign`; no private key material is available in the container) are never forwarded. Path-valued settings like `core.excludesfile`/`core.hooksPath` are also skipped, since honoring them would require mounting a host file outside the project directory.
10. **Local agent rules and memory, read by both tools**: every run, `agentcubicle` maintains three files under `.agentcubicle/` in your project: `AGENTS.local.md` (general, tool-agnostic rules), `MEMORY.local.md` (a running account of current focus, recent decisions, and open threads, referenced from `AGENTS.local.md`), and `CLAUDE.local.md` (Claude-Code-*specific-only* instructions, which imports the other two via `@` includes). The `.local.md` suffix (matching the `CLAUDE.local.md` root symlink below) marks them unambiguously as agentcubicle-managed, so they never collide with a bare `AGENTS.md`/`CLAUDE.md`/`MEMORY.md` your project might track itself. These are seeded with a starter template the first time, then left alone; edits persist and are never overwritten. This is **additive**: it applies even if your project already has its own tracked `AGENTS.md`/`CLAUDE.md`, which continue to load normally through each tool's own discovery. Projects seeded by an earlier version (which used the bare names) are migrated automatically on the next run: the legacy `.agentcubicle/*.md` files are renamed to `*.local.md`, their `@` imports and the root symlink are repointed, and a one-line notice is printed; the move is lossless and one-time.
   - `opencode` reads `.agentcubicle/AGENTS.local.md` and `.agentcubicle/MEMORY.local.md` directly via its `instructions` config list, merged in through the container-only `OPENCODE_CONFIG_CONTENT` override (see above), never by writing to your real `opencode.json`/`.jsonc`.
   - `claude` reads `.agentcubicle/CLAUDE.local.md` via `CLAUDE.local.md` at the project root, a symlink Claude Code loads automatically every session (matching its own documented `CLAUDE.md` symlink convention). `CLAUDE.local.md` is a strong, tooling-enforced convention for personal/never-committed content, so it's not at risk of colliding with a file the repository might track later. That `CLAUDE.local.md` in turn pulls `AGENTS.local.md` and `MEMORY.local.md` in via Claude Code `@` imports, so all three are inlined into the session rather than relying on the model to go read them; the imports resolve correctly even through the `CLAUDE.local.md` symlink.
   - Deliberately, nothing is ever placed at the literal `AGENTS.md`/`CLAUDE.md` paths themselves: even a `.git/info/exclude`d symlink there would risk breaking a future `git pull` if the upstream remote ever adds a real tracked file at that exact path: git refuses to silently overwrite an untracked/excluded file during a merge, exclude or not.
   - `.agentcubicle/` and `CLAUDE.local.md` are both registered in the repo's local `.git/info/exclude` (not `.gitignore`, for the same leak-prevention reason as the persisted Claude state above).
   - The seeded rules point agents at `MEMORY.local.md` for anything that has to survive across sessions, and ask that any *additional* memory file they create be listed there by its repo-relative path, preferably under `.agentcubicle/memories/`. The point is that a file no tool loads automatically is a file the next session will not find. That directory needs no separate git handling: the existing `/.agentcubicle/` exclude entry already covers it. A memory directory a tool manages and loads itself (Claude Code's, under `claude-home/`) is exempt and stays where the tool puts it.
   - Both `AGENTS.local.md` and `CLAUDE.local.md` ask the tool to state, at the start of every session, which local files it read and a one-paragraph summary of what it picked up, but an interactive session never gives the model an unprompted turn to say anything until it receives input. So every run also auto-sends a short kickoff prompt (via opencode's `--prompt` / Claude Code's positional `[prompt]` argument, both of which submit an initial message without leaving interactive mode) asking it to do exactly that as its first reply. Any of your own extra args are still respected; `--prompt`/the positional prompt argument just becomes the tool's first turn.
11. **Clipboard / Display**: Auto-detected based on environment:
    - **Wayland**: If `WAYLAND_DISPLAY` and `XDG_RUNTIME_DIR` are set, the Wayland socket is mounted and made available. Under Podman it is staged in a container-local runtime dir (the host `/run/user/<uid>` path is not reusable when the container UID differs from the host).
    - **X11**: If `DISPLAY` is set, `/tmp/.X11-unix` is mounted and `XAUTHORITY` is forwarded if present.
    - The Podman clipboard path has not been verified on a real desktop yet; report back if it misbehaves.
12. **Git**: `/home/user/project` is registered as a safe directory so git operations work without warnings.

## Installed packages

The following packages are installed in the `agentcubicle` image:

| Category     | Packages                                                                                                                                                       |
|--------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Editors**  | `vim`, `nano`                                                                                                                                                  |
| **Compilers**| `gcc`, `g++`, `build-base` (make, etc.)                                                                                                                        |
| **Debuggers**| `gdb`, `gdb-dashboard`, `cgdb`, `valgrind`, `strace`                                                                                                           |
| **Git**      | `git`, `gh`                                                                                                                                                    |
| **CLI tools**| `bat`, `diffutils`, `file`, `fzf`, `jq`, `lsof`, `patch`, `perl`, `sqlite`, `shellcheck`, `tree`                                                              |
| **Network**  | `curl`, `curlie`, `httpie`, `nmap`, `netcat-openbsd`, `tcpdump`, `wrk`                                                                                         |
| **Languages**| `lua5.4`, `luajit`, `nodejs`, `python3`, `py3-pip`, `uv`                                                                                                       |
| **System**   | `htop`, `tmux`, `bash`                                                                                                                                         |
| **Clipboard**| `wl-clipboard`, `xclip`                                                                                                                                        |
| **AI tools** | `opencode` (from base image), `claude` (optional, installed via `setup --claude`)                                                                              |

Everything above comes from Alpine's `apk`, except `gh` (no apk package exists for it, so it comes from an official release tarball) and `opencode`/`claude` (base image and the Claude installer, respectively).

## Extra packages

To add your own packages to the image, list them in
`~/.config/agentcubicle/packages`, one per line. The file belongs to you, not to
a project: it applies to every project. It is deliberately outside
`.agentcubicle/`, which is mounted into the container and writable by the agent;
`~/.config/agentcubicle/` is not mounted. The file decides what gets installed
into the image as root:

```
# Anything after a # is a comment
figlet
postgresql-client:psql   # "package:binary" lets `check` verify the binary
```

Names are Alpine `apk` packages. An invalid line is skipped with a warning, and
a name `apk` does not know makes `agentcubicle setup` fail and point at this file.

Run `agentcubicle setup` after editing the file. If the image already exists,
only the changed package list is installed on top of it; nothing is rebuilt from
scratch. `agentcubicle check` reports these packages too, marked `(user)`.
Removing a line does not uninstall the package from an existing image: run
`agentcubicle setup --clear` to rebuild without it.

The image and the list are both shared by all your projects.

### Tools that are not Alpine packages

For anything `apk` does not carry (a release tarball, a `curl | sh` installer),
put a shell script in `~/.config/agentcubicle/install.sh`. It runs as root, with
`sh -eux`, at the end of the image build, after the packages and `gh`, so a
failing command fails `setup`. Follow what `gh` does and install into
`/usr/local/bin`, which is on the PATH. For something that belongs in the
user's home, drop privileges as the Claude install does:
`su -s /bin/sh user -c '...'`.

The script's content is passed to the build as a build argument, so it is
recorded in the image history (`docker history`, `podman history`). Do not put
tokens or other secrets in it.

```sh
# ~/.config/agentcubicle/install.sh
curl -fsSL https://example.com/tool.tgz | tar -xz -C /usr/local/bin tool
```

Unlike `packages`, the script is not tracked: it only runs when the image is
built, so after editing it run `agentcubicle setup --clear` (adding `--claude`
if you want Claude Code). A plain `setup` on an existing image leaves it alone
and says so. `check` does not verify what the script installed.

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

# Exchange files with another container through a named share
agentcubicle share create xchg
agentcubicle claude --share xchg

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

# Keep the container after the tool exits, so it can be committed afterwards
agentcubicle claude --keep-container --name ac-myproject-abc123
agentcubicle commit --name ac-myproject-abc123

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

- Docker, or rootless Podman. Auto-detected, Podman preferred when both are
  installed; override with `AGENTCUBICLE_ENGINE=docker` or `=podman`.
- `jq`
- Current directory as the project workspace
- Your normal user account. agentcubicle refuses to start as root, including
  under `sudo`: it maps your host UID/GID onto the container user and reads tool
  config from your `$HOME`, and as root both of those give the wrong answer. If
  your user cannot reach the Docker daemon, add it to the `docker` group
  (`sudo usermod -aG docker <user>`, then start a fresh login) or use rootless
  Podman, which needs neither. See
  [docs/DESIGN.md](docs/DESIGN.md) for the details.

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md)
first, especially before anything larger than a small fix. It covers what
to raise in an issue up front, how AI-assisted changes are expected to work
(a human owns and understands them), and the commit conventions I follow.
