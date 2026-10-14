# Design

Durable design rationale and architecture decisions for `agentcubicle`,
the "why" behind non-obvious choices, meant to outlive any single
session. This is git-tracked and permanent, unlike
`.agentcubicle/MEMORY.local.md` (private, ephemeral work memory), and
distinct from the other git-tracked docs: `README.md` (user-facing usage)
and `CONTRIBUTING.md` (contributor-facing policy). See
`.agentcubicle/AGENTS.local.md`'s "Memory" section for how the durable
docs and the ephemeral memory are meant to relate.

Deliberately avoids citing specific commit hashes or session dates,
because those rot (this repo's entire history was rewritten once already,
invalidating every hash cited anywhere at the time) or become
irrelevant. `git log`/`git blame` are the authoritative source for
"when" and "which commit"; this file is for "why," which doesn't change
just because the code around it does.

## Naming

Named `agentcubicle`, renamed from an earlier `opencode-docker`, chosen
after checking for GitHub name collisions, over roughly twenty other
rejected candidates that were already taken.

## Container `$HOME` vs. the project directory

The container's `$HOME` (`/home/user`) and the bind-mounted project
directory (`/home/user/project`) are deliberately separate paths, not
overlaid (an earlier design conflated them as `/work`). `$HOME` is
throwaway scratch space discarded when the container exits; the project
directory is the user's real, persisted work. Keeping them as distinct,
clearly-named paths means it's never ambiguous which files are
ephemeral container state and which are the user's actual files, which
matters both for correctness (nothing important should ever be
written under `$HOME`) and for the mount-narrowing decisions below
(scoping recursive operations like `chown -R` to skip the project mount
relies on it being a clearly separate subtree).

A side effect of the fixed mount path is that every project looks like a
directory named `project` from inside the container, and a project that is
not a git repo has no repo name either. Only the host knows the real name,
so the script passes `basename` of the host working directory as
`AGENTCUBICLE_PROJECT_NAME` (for both tools, through the same `EXTRA_ENVS`
list as everything else, so it reaches both engine paths without touching
the single-quoted entrypoint). The main consumer is a propagated
`statusLine` command. An explicit `--env AGENTCUBICLE_PROJECT_NAME=...`
wins.

## Claude Code project state persistence

Claude Code normally keeps per-project memory, transcripts, and
worktree state under `~/.claude/projects/<encoded-path>/`, but every
container gets a fresh throwaway `$HOME`, so that would be wiped every
run. `CLAUDE_CONFIG_DIR` is instead pointed at
`/home/user/project/.agentcubicle/claude-home`, inside the read-write
project mount, so it persists across container recreations without
ever giving the container host access outside the project directory,
a hard constraint from the project's inception, not something added
later for convenience.

The host `settings.json` is copied into `claude-home` **only on the
first seed**: the `cp` is guarded by
`[ ! -f "$CLAUDE_STATE_DIR/settings.json" ]`, so once the persisted
copy exists it is authoritative and the (still read-only-mounted) host
file is ignored for seeding on every subsequent run. Two consequences
follow, and they are the point: settings Claude Code writes
*in-container* (a `/model` choice, say) **persist for that container**
across recreations; and editing the host `settings.json` afterward does
**not** propagate back into an already-seeded container, so each
container captures the host state as of its own first run and then
diverges (two containers can legitimately end up on different models).
This is deliberately the opposite of `opencode`, whose config is
re-copied from the host on every run (see "opencode provider and model
configuration") and therefore always reflects the current host while
persisting nothing written in-container. Divergence is the price of
persistence, and persistence is exactly what `claude` needs, since it
accumulates real per-project state, whereas `opencode` does not.

This directory is the one exception to the "recursive `chown -R` skips
the project mount" rule above. The entrypoint creates and seeds
`claude-home` during its root phase (the `mkdir` and the
`settings.json` / `.claude.json` seeds happen before the `su` that
drops privileges), so unlike the user's own files (which the host
writes and which therefore already carry the host UID/GID), it is born
`root:root`. Because it lives *inside* the skipped project mount, the
blanket fixup never touches it, and the unprivileged user we drop to
would hit EACCES writing its own state. So `claude-home` gets its own
explicit `chown -R $HOST_UID:$HOST_GID` right after it's seeded. The
chown is unconditional, which also self-heals a directory left
`root:root` by an older, buggy build on the next run.

The whole root-phase-and-chown scheme above is the **Docker** model. Under
rootless **Podman** none of it applies (see "Container engine" below): the
process already runs as the mapped user, so `claude-home` is born owned by
that user and needs no chown.

### Keys that always follow the host

Seed-once is right for settings Claude Code writes about itself in a given
container, and wrong for settings the user thinks of as theirs rather than
that container's. A status line is the second kind: editing it per
container, or deleting `claude-home/settings.json` to force a re-seed (and
losing everything else in it) is not a reasonable way to change how a
prompt looks.

So there is a second channel next to the seed: `CLAUDE_PROPAGATE_KEYS`, an
allowlist of top-level keys read from the host `settings.json` on every run
and merged **last** into the persisted copy, so they win over both the seed
and anything written in-container. The merge is a shallow `+` in the same
`jq` invocation that sets `skipDangerousModePermissionPrompt`, so a
propagated key replaces its previous value wholesale rather than being deep
merged. Keys the host does not define are not in the merge object at all,
which is deliberately not the same as mirroring the host: removing a key on
the host leaves the container value alone rather than deleting it. Adding
the next key is one array entry, plus a `case` arm if it needs validating.

Extraction and validation happen on the **host** side, not in the
entrypoint, for two reasons: the entrypoints live inside a single-quoted
`sh -c '...'` string where a `jq` program of any complexity is painful to
quote, and a warning about the host's own config belongs where the user can
act on it. The accepted object is passed in base64 (`CLAUDE_PROPAGATE_B64`),
the same trick the redacted opencode config uses, so its JSON never has to
survive that quoting region.

`statusLine` is validated because only an inline command can run in the
container: a `command` whose first word is a path names a host script that
is not in the container's filesystem, and the resulting status line would
just fail silently every render. Such a value is skipped with a warning, and
the same key is also stripped from the *seed* copy
(`CLAUDE_SEED_STRIP_JQ`), since otherwise a fresh container would receive
through the seed exactly the value we just said we would not propagate. The
first-word test is a heuristic and knows nothing about what an inline
command calls: `python3 -c ...` passes even if the script it runs needs a
host-only file.

## User-configured packages

Users can add Alpine packages through a per-user file,
`~/.config/agentcubicle/packages`, and other tools through
`~/.config/agentcubicle/install.sh`.

- **Per user, outside the project.** Both files live in
  `~/.config/agentcubicle/`, not in `.agentcubicle/`. That directory is
  bind-mounted into the container (the agent's rules, memory and Claude
  state live there), so anything in it is editable by the agent. These files
  decide what is installed into the image as root, and `install.sh` is
  arbitrary code: a project-local copy would let the agent write a script
  that runs as root on the host's next `setup`, outside the sandbox. The
  host config directory is not mounted into the container. It also matches
  the image, which is a single shared `agentcubicle` for all
  projects: there is one list, so there is no "last project wins" ambiguity.
  The path is fixed at `$HOME/.config/agentcubicle`, like the existing
  `$HOME/.config/opencode` lookup, and `XDG_CONFIG_HOME` is not consulted.
  Per-project lists would need a per-project image, which is not worth it
  unless image size becomes a problem.
- **Plain text, not JSON.** One entry per line, in the same `pkg` or
  `pkg:binary` form as the built-in list. It is easy to write by hand and
  needs no `jq`, which the image-building path does not otherwise depend on.
  Entries are validated against a strict name pattern because they end up
  in a Dockerfile `RUN` line.
- **A hash label, not a rebuild.** `setup` used to exit as soon as the image
  existed, so an edited list would never have taken effect without
  `setup --clear` and a full rebuild. Instead, the image carries an
  `agentcubicle.user-packages` label holding a hash of the sorted package
  names. When it differs from the current file, `setup` builds a thin layer
  on top of the existing image, as `--claude` does, rather than starting
  over. An absent label equals an empty list, so existing images are not
  rebuilt by the upgrade.
- **Removal is not incremental.** A package deleted from the file stays in
  the image, because `apk del` in a new layer would not reclaim the space and
  could break things that came to depend on it. `setup --clear` is the way
  to drop one.
- **`install.sh` for non-apk tools, untracked.** A shell script at
  `~/.config/agentcubicle/install.sh` is passed to the build as a base64 build arg
  (the build reads its Dockerfile from stdin, so there is no context for
  `COPY`, and base64 avoids quoting problems) and run as root with
  `sh -eux` as the last build step. It follows `gh`: root, installing into
  `/usr/local/bin`. It gets no hash label, unlike `packages`: an arbitrary
  script cannot be diffed or applied incrementally, and re-running it on top
  of a previous layer would have to be idempotent. So it only runs on a
  fresh build, and the user applies changes with `setup --clear`. `setup`
  prints a reminder when the file exists and the image does not need
  building. `check` cannot verify what it installed. The cost of the build
  arg is that its value is recorded in the image history, so the script is
  not a place for secrets; the README says so.

## Container engine (Docker and Podman)

The engine is chosen once at startup: `AGENTCUBICLE_ENGINE` if set, otherwise
auto-detected preferring `podman`, falling back to `docker`, erroring if
neither is present. Every container call goes through a single `ENGINE`
variable. Two engine differences drive the rest:

- **UID mapping.** Docker (rootful) starts the container as root, remaps the
  baked `user` to your host UID/GID, `chown`s, then `su`s down. Rootless
  Podman cannot do that: its container "root" maps to a subordinate UID, not
  your host user, so a post-remap `user` would write files owned by an
  unusable subuid. Instead Podman runs with `--userns=keep-id:uid=1000,gid=1000`,
  which maps your host user straight onto the image's `user` (UID 1000), and
  the process runs as that user directly, with no root phase, `usermod`, or
  `chown`. Bind-mounted writes then land owned by you on the host in both
  models, by opposite mechanisms. The `su` call ends its own options with
  `--` before the user name: the image's `su` is BusyBox on musl, whose
  option parser keeps reading options after operands, so without it any
  tool argument shaped like a `su` option was taken by `su` (`--help`
  failed with a `su` error, and Claude's `-c` would have replaced the
  command `su` runs). The Podman path starts the tool with `sh -c`, so it
  never had this problem.
- **Config staging.** The Docker path reads host config from under `/root`
  (it is root at that point). The unprivileged Podman process cannot read
  `/root`, so config is staged read-only under `/home/user/.config-host/...`
  and copied into place by the entrypoint. The opencode config mount is
  guarded on the source directory existing, because Podman errors on a
  missing bind-mount source where Docker silently creates it (root-owned).

Supporting details: the locally built image is referenced as
`localhost/agentcubicle` under Podman so it is not mistaken for a registry
short name; `--security-opt label=disable` avoids relabeling the project on
SELinux hosts; and the Wayland socket is staged in a container-local runtime
dir rather than reusing the host `/run/user/<uid>` path. Both engine paths
(file ownership round-trip and host config forwarding) are exercised by the
`engines` CI workflow; the clipboard path is not (runners are headless).

## Running as root is not supported

agentcubicle exits with an error if `id -u` is 0, before engine detection, for
every subcommand. The trigger was a user whose account was not in the `docker`
group running `sudo ./agentcubicle opencode`, which got as far as starting a
container and then failed with three errors that all pointed away from the cause:
`groupmod: GID '0' already exists`, `could not lock config file
/home/user/.gitconfig`, and `EACCES: permission denied, mkdir '/home/user/.local'`.

Everything here is keyed to the identity of the invoking user, and under `sudo`
that identity is root, which breaks the run in three places:

1. `$HOME` is `/root`, so the config-home lookup reads root's config rather than
   the user's. In the reported case that produced a "no providers configured in
   `/root/.config/opencode/`" warning: the user's own credentials and settings
   never reached the container.
2. `HOST_UID`/`HOST_GID` are both 0, so the root-phase `groupmod -g 0 user` and
   `usermod -u 0 user` collide with the container root account and fail. `user`
   keeps UID 1000.
3. The `chown`s run anyway with `0:0`, making `/home/user` and its contents
   root-owned, and then the `su` drops to UID 1000 into a home it cannot write.
   That is the gitconfig lock failure and the `EACCES`.

Supporting `sudo` was rejected rather than fixed. It would mean reconstructing
the invoking user from `SUDO_UID`/`SUDO_GID`/`SUDO_USER` and their home, then
doing the same for everything else keyed to the login session: `XAUTHORITY`,
`DISPLAY` and `XDG_RUNTIME_DIR` for the X11/Wayland/clipboard forwarding, plus
git identity forwarding. That is a second identity-resolution path to maintain
for a case that has two existing answers: join the `docker` group, or use
rootless Podman (already preferred when present). `sudo` also gains nothing over
`docker` group membership, since that membership is root-equivalent anyway.

The check is on UID 0, not on `SUDO_USER`, because being logged in as root fails
at step 2 in exactly the same way. There is deliberately no override environment
variable, since there is no working mode behind it to unlock.

Refusing every subcommand rather than only the run path keeps the rule one line
and one message, at the cost of one sharp edge: containers created by an earlier
root run cannot be cleared with `sudo agentcubicle cleanup` either. The error
message therefore points at removing them through the engine directly.

## Git identity forwarding

An explicit **allowlist** (not a blocklist) of the host's
`git config --global` settings is forwarded into the container, so
commits made by an agent carry correct authorship: `user.name`,
`user.email`, `init.defaultBranch`, `pull.rebase`, `push.default`,
`push.autoSetupRemote`, `core.editor`, `color.ui`, `rerere.enabled`,
`merge.conflictstyle`, and all `alias.*` entries. `credential.helper`
and any GPG/commit-signing settings are never forwarded, not because
they're blocked by a rule that could miss something, but because an
allowlist means there's nothing to accidentally leak in the first
place. Path-valued settings (`core.excludesfile`, `core.hooksPath`)
are skipped too, since honoring them would require mounting a host file
outside the project directory.

The forwarded values are decoded straight to a file and pointed at via
`git config --global include.path`, rather than reconstructed as
individual `git config` calls built from arbitrary text. Reconstructing
config from arbitrary values means re-parsing that text as shell
through the container entrypoint's `su -c` invocation, exactly the bug
class that has bitten this script for real (a corrupted JSON blob from
a double shell-reparse, and a stray apostrophe in a single-quoted
region prematurely terminating a string). Decoding a fixed blob
straight to a file sidesteps that class of bug entirely rather than
requiring careful escaping to avoid it.

## Local agent files: `AGENTS.local.md`, `CLAUDE.local.md`, `MEMORY.local.md`

These live under `.agentcubicle/` in the project, are seeded once and
never overwritten, and are **additive**: they apply even when the
project already has its own tracked `AGENTS.md`/`CLAUDE.md`, which
continue to load normally through each tool's own discovery.

The `.local.md` suffix is deliberate. The bare basenames
(`AGENTS.md`/`CLAUDE.md`/`MEMORY.md`) are exactly the ones a project may
track for its own purposes, so a reader (human or agent) couldn't tell
at a glance which files are agentcubicle-managed (local, never committed)
versus project-owned. `.local.md` marks the managed set unambiguously and
makes it uniform with the `CLAUDE.local.md` root symlink, which already
carried that convention. This is purely a naming distinction; the files
still live under `.agentcubicle/`, so the collision-avoidance reasoning
below is unchanged by it.

Because the suffix arrived after projects had already been seeded under
the bare names, the change is backward-compatible: on the next run,
agentcubicle auto-renames any legacy `.agentcubicle/*.md` files to
`*.local.md`, rewrites the `@`-import directives inside the migrated
`CLAUDE` file (they resolve relative to `.agentcubicle/`, so bare-name
imports would otherwise dangle and silently drop the imported content),
repoints the root symlink from the old target, and prints a one-line
notice. It's lossless, idempotent, and (matching how the files are
already seeded, symlinked, and excluded) done without prompting. Only
the load-bearing import directives are rewritten; stale self-referential
*prose* inside a file a user has already customized is left alone.

A naming distinction only helps if the agent reading the file honors it,
and agents drop the `.local` part. Two reasons, and neither needs a
legacy-seeded project: a model summarizing which instruction files it
loaded treats `AGENTS.local.md` and `AGENTS.md` as the same name worth
mentioning, and opencode's own system prompt primes the bare name hard
(it describes the `AGENTS.md` convention at length and tells the agent it
MUST keep `AGENTS.md` files updated, while its native discovery walks up
the tree for `AGENTS.md` specifically; the local files reach it through
the separate `instructions` config array). Left alone, that ends with an
agent writing rules meant for a local, never-committed file into a
tracked `AGENTS.md` at the project root, shared with everyone who clones,
which is the outcome the naming exists to prevent.

So each seeded template states its own identity in its header: this file
is `AGENTS.local.md`, a bare `AGENTS.md` is a different, tracked file,
refer to this one by its full name. The two destination rules that
legitimately mention the bare names say inline that they mean the
project's own shared file. This is a nudge like any other model
instruction, not a guarantee, but it puts the disambiguation in the same
context window as the mistake.

They're deliberately never placed at the literal `AGENTS.md`/`CLAUDE.md`
paths, even as a `.git/info/exclude`d symlink: if the upstream remote
ever adds a real tracked file at that exact path, git refuses to
silently overwrite an existing untracked/excluded file during a merge
or pull, so a working symlink today would break the next `git pull`
later. `CLAUDE.local.md` (a real symlink to `.agentcubicle/CLAUDE.local.md`)
is safe by contrast because it's Claude Code's own documented
convention for exactly this purpose, not a path something else might
someday claim.

`CLAUDE.local.md` `@`-imports `AGENTS.local.md` and `MEMORY.local.md` directly, rather
than a prose instruction telling the model to go read them separately.
This closes an architectural asymmetry: opencode's pickup of these
files is deterministic (a config array it always reads), while a prose
instruction could in principle be skipped under context pressure.
`@`-import makes Claude Code's pickup deterministic too.

Every run auto-sends a short kickoff prompt as the tool's first turn
(opencode's `--prompt`, Claude Code's positional `[prompt]` argument),
asking the agent to announce what it read. A passive instruction inside
`AGENTS.local.md`/`CLAUDE.local.md` alone is not sufficient for this: an interactive
TUI never gives the model an unprompted turn to act on it, so without
an explicit first message, the instruction just sits there unused until
the user happens to say something first.

### The three-tier documentation model

`.agentcubicle/MEMORY.local.md` is temporary work memory only (current
focus, recent decisions not yet written down elsewhere, and open
threads), not a place for anything meant to last. `README.md` is
user-facing usage documentation. This file is durable design rationale.
Content is expected to flow from the first into the other two over
time (a decision gets made and noted in MEMORY.local.md, then later promoted
into README.md or here once it's stable, and trimmed out of MEMORY.local.md
accordingly) rather than accumulating indefinitely in the ephemeral
one. See `.agentcubicle/AGENTS.local.md`'s "Memory" section for the actual
rule an agent follows.

`CONTRIBUTING.md` sits outside this model. It's a git-tracked,
contributor-facing policy doc (how to propose changes, the
human-owns-the-change stance on AI, commit conventions), not a tier that
knowledge flows into from MEMORY.local.md.

## Claude commit trailer

`agentcubicle` does not force-suppress Claude Code's
`Co-Authored-By: Claude` commit trailer. Whether to see that trailer in
commit history is a host-level, user-level preference, not something
a container-launching tool should decide on the user's behalf. The
README documents how to disable it on the host if wanted, but the tool
itself stays deliberately silent on the matter ("discreet about git").

This is a separate question from `CONTRIBUTING.md`'s rule that
contributors not attach an AI co-author trailer to commits they author.
That rule is about contribution authorship (a human owns the change, so
the AI is not a co-author); this section is only about what the launcher
tool imposes, which is nothing. The tool stays neutral on the trailer,
while the contribution policy asks the human to own authorship. The two
do not conflict.

## Claude Code first-run authentication

The container never receives the host's live Claude Code credentials:
only `~/.claude/settings.json` is ever mounted (read-only) and copied
out; the raw `.credentials.json` (a single-use, rotating refresh token)
is never mounted or copied at all, since a read-only copy would go
stale the moment it's refreshed anywhere. Instead, a long-lived static
token from `claude setup-token` is forwarded as an environment variable
(`CLAUDE_CODE_OAUTH_TOKEN`), read from `~/.claude/oauth-token` on the
host (or an already-exported variable, or `ANTHROPIC_API_KEY`). A
static token has no rotation problem, which is what makes it reusable
across many container runs.

`agentcubicle claude` detects, host-side, before the container even
starts, whether any of those credential sources will actually resolve.
If none will, it skips the normal session entirely (starting one would
just hit a login wall) and instead runs `claude setup-token` for the
user automatically, explains what's happening immediately before the
authorization URL is printed (not only once, earlier, on the host side,
where an explanation would scroll away from the actual link by the time
docker finishes starting the container), then tells the user exactly
asks them to paste the printed token.

The pasted token reaches the host through a pending file in the project
mount (`.agentcubicle/.oauth-token-pending`, written with umask 077),
which the host side moves to the token file with mode 600 once the
container exits. This keeps `~/.claude` out of the container entirely,
writable or not; the alternative of mounting it read-write would expose
`.credentials.json` again. The move runs from an EXIT trap so it still
happens when the engine exits non-zero, and any pending file is removed
both before the run and after it. If nothing is pasted, the bootstrap
falls back to printing the manual save steps and drops into an
interactive shell to do them, rather than exiting the container out
from under the user.

`--account NAME` swaps the token file for `~/.claude/oauth-token-NAME`,
which is the whole of multi-account support: the forwarded token is the
only thing that ties a container to an account. With `--account`, a
host-exported `CLAUDE_CODE_OAUTH_TOKEN` is ignored, because it most
likely belongs to the default account and would otherwise silently win
over the account the user explicitly named (and keep the bootstrap from
running for a new one). An explicit `--env` still takes precedence.

The config mount backing `settings.json` is scoped to that single file,
not the whole `~/.claude` directory. Mounting the whole directory only
to copy one file out of it meant `oauth-token` (already forwarded
separately) and the raw `.credentials.json` rode along into the
container's mount namespace too, redundant exposure serving no
purpose. The single-file mount is guarded on the file's existence,
since Docker creates an empty directory at the host path if you
bind-mount a single file that doesn't exist yet, which would corrupt a
future `claude setup-token` save into that same location.

## opencode provider and model configuration

opencode's own host config (`~/.config/opencode/opencode.json` or
`.jsonc`) already propagates into the container (mounted read-only and
copied in on every run), so provider/model configuration is edited on
the host exactly like a native install, no agentcubicle-specific step
needed. Without any provider configured, opencode falls back to its own
built-in free models.

Two things are true here that are easy to assume incorrectly:

- **`opencode auth login` does not work with agentcubicle.** It stores
  credentials at `~/.local/share/opencode/auth.json`, a completely
  different directory from `~/.config/opencode`, which agentcubicle
  never mounts or copies into the container. Anything saved there never
  reaches a session; this was checked directly (`opencode debug
  paths`), not assumed.
- **The mechanism that does work** is referencing the API key in the
  config as `{env:YOUR_VAR}` rather than a literal value, then
  forwarding the real value via `agentcubicle`'s existing `--env` flag.
  This was verified empirically (`opencode debug config` against a
  temp `HOME`) rather than inferred from opencode's docs alone, and it
  requires no agentcubicle-specific plumbing: `{env:VAR}` resolution is
  opencode's own config feature, and `--env` already existed for other
  purposes.

`agentcubicle` checks the host config for a non-empty `provider` entry
before starting an opencode container (tolerating `.jsonc` files with
comments, which `jq` cannot parse, via a text-based fallback check) and
prints a one-line reminder if none is found. This is purely
informational and never blocks a session from starting; the free
fallback models are a legitimate thing to want.

README leads with a self-hosted/local-model example (an OpenAI-
compatible gateway like LiteLLM, vLLM, or Ollama) before the
hosted-vendor case, since a hosted vendor's config is comparatively
trivial (opencode ships built-in support for the common ones, so it's
just an API key) while the self-hosted case needs the fuller shape
(`npm`, `name`, a custom `models` catalog) and benefits more from a
worked example.

**Literal API keys are auto-redacted.** If the host opencode config sets
a provider's `options.apiKey` to a literal value (not a `{env:...}` or
`{file:...}` reference), agentcubicle forwards the real value into the
container as a generated env var (`AC_OPENCODE_<PROVIDER>_APIKEY`) and
feeds the container a redacted copy of the config that references
`{env:<that var>}` instead. The literal therefore never lands in the
container's writable config copy (so it can't be captured by a
`docker commit`), and a user who keeps a literal key in their own config
doesn't have to hand-convert it to the `{env:VAR}` shape, nor type it
into a `--env` flag, where it would land in shell history. When
redaction applies, the raw config file is not mounted into the container
at all: the redacted copy is injected via a base64 env var and written
by the entrypoint, so even the container's read-only mount namespace
never sees the literal.

This is only attempted when `jq` can parse the config. A `.jsonc` file
with comments (which `jq` cannot parse) is deliberately left untouched
and falls back to the plain mount-and-copy path: attempting a
regex-based rewrite risks silently corrupting the config or failing to
redact, which would be worse than not redacting at all. Detection is
per-provider and keys off literal-vs-reference, so configs already using
`{env:...}`/`{file:...}` are unaffected.

## Scope: opencode session state is not persisted

Claude Code's per-project state is persisted via `CLAUDE_CONFIG_DIR`
(see "Claude Code project state persistence" above), but there is
deliberately no equivalent for opencode. opencode keeps its data and
state, such as sessions/history, provider credentials (`auth.json`), and
other state, under `~/.local/share/opencode` and `~/.local/state/opencode`,
both of which live under the container's ephemeral `/home/user` home and
are therefore discarded when the container exits.

This is a conscious scope decision, not an oversight: persistence was
built for claude because that is what was needed, and it is unclear
opencode has per-project state worth the same treatment. If it turns out
to matter, the same pattern applies: point opencode's data/state dirs
(via `XDG_DATA_HOME` / `XDG_STATE_HOME`, or bind mounts) at a location
inside the project mount, never at host paths outside it.

## Container lifetime: removed on exit

Containers are started with `--rm`, so a run cleans up after itself.
The alternative, which is what the script did originally, is to leave
every exited container in place and rely on the user remembering to run
`agentcubicle cleanup`; in practice they accumulate and take disk space.

Removal is the default because the container holds nothing that is not
already somewhere more durable. Files the user cares about are in the
bind-mounted project directory, which is untouched by removing the
container. Claude Code's per-project state lives in that same mount
(see "Claude Code project state persistence" above) precisely so that it
survives container recreation, which already happened on every run,
since each run creates a new container rather than restarting the last
one. What is left is the container's ephemeral `/home/user` (throwaway
by design, see "Container `$HOME` vs. the project directory") and any packages
installed during the session, which `commit` captures into the image.

`--keep-container` opts out. Two things shaped how the opt-out works:

- It is an explicit flag rather than something inferred from `--name`.
  Naming a container could plausibly be read as intent to reuse it, but
  `--name` has an unrelated everyday use: giving the container a fixed
  name so `shell` and `commit` can address it during the session. Tying
  lifetime to it would mean two rules to remember instead of one, and
  would make the common case (a stable name for one session) silently
  keep containers around again.
- No exit-time warning is printed when a container that diverged from
  its image is about to be removed. Detecting divergence cheaply and
  accurately is not straightforward, and a warning on every run that
  installed a package would be noise for the common case where the user
  did not want to keep it anyway.

The cost is that committing a container *after* it exits requires having
decided to keep it before starting. Committing a still-running container
from a second terminal, the interactive `commit` path, is unaffected,
and that is the usual way it is done.

## Tool auto-updates are disabled in the container

Both tools update themselves in the background. In a container that is
removed on exit (see "Container lifetime: removed on exit" above), that
update has nowhere to land: Claude Code installs the new build into
`~/.local/share/claude/versions/` and re-points `~/.local/bin/claude` at
it, both inside the container's throwaway `/home/user`, and both gone the
moment the tool exits. The restart it then asks for cannot help either,
since the tool runs via `exec` as a direct child of the entrypoint shell,
so exiting it exits the container. Measured in a real session: an image
built with `2.1.250` had fetched `2.1.272` (221MB) within minutes, and
would have fetched it again on the next run, and the one after that.

So the run path sets `DISABLE_AUTOUPDATER=1` for Claude Code, and the
generated opencode config sets `autoupdate: false`. opencode would have
failed at it anyway: its binary is root-owned in `/usr/local/bin` and the
tool runs as the unprivileged user.

Three details of how, rather than whether:

- It is an environment variable on the run path, not a key written into
  the seeded `claude-home/settings.json`. That file is persistent state
  the user (and the agent) may edit, seeded once and never rewritten, so
  writing into it would both mutate something owned by the user and miss
  every project seeded before the change. The variable applies to every
  container without touching any file.
- The value is `${DISABLE_AUTOUPDATER:-1}`, expanded by the inner shell
  after the `su` on the Docker path, so `--env DISABLE_AUTOUPDATER=0`
  still wins and the escape hatch needs no extra code.
- `DISABLE_AUTOUPDATER` rather than `DISABLE_UPDATES`: it stops the
  background check while leaving a deliberate `claude update` inside a
  container working. That update is still discarded on exit, but it is
  the user asking for it, which is a different thing from the tool doing
  it unbidden.

The consequence is that the tool version is pinned by the image, and the
image goes stale silently. That is why `check` reports versions for
`claude`, `opencode` and `gh` rather than a bare tick: the refresh path is
a host-side rebuild (`agentcubicle setup --clear --claude`), so the check
has to be what makes staleness visible.

Two alternatives were considered and rejected:

- Persist `~/.local/share/claude/versions` on the host and install a
  custom launcher at `~/.local/bin/claude` picking the newest version
  found there (Claude Code leaves a launcher it did not create alone,
  and keeps installing into `versions/`). Updates would then survive and
  apply on the next run. It costs a new host-side state directory outside
  the project, roughly 220MB per version kept with no pruning story, and
  another uid-ownership question under rootless Podman.
- Install Claude Code from Anthropic's Alpine `apk` repository, which
  does not auto-update at all, instead of `install.sh`. That is a signing
  key plus a repository line plus a different install path in the build,
  for the same end state the environment variable already reaches.

## Named shares

Some workflows need two containers (usually two projects) to exchange
files under the user's control. `--mount` could already do it, but with
two traps: a target under `/home/user` is caught by the Docker root
phase's recursive `chown` of the home (which also errors on a read-only
mount), and a named engine volume starts root-owned, so the unprivileged
user cannot write to it.

A share is a plain host directory under one fixed root,
`${XDG_DATA_HOME:-$HOME/.local/share}/agentcubicle/shares/NAME`, managed
with `agentcubicle share create|list|rm`, and enabled per container with
`--share NAME[:ro|:rw]`, which mounts it at `/shares/NAME`.

- **Fixed location, no registry.** Defining a share is creating a
  directory; the name alone locates it. A registry file mapping names to
  arbitrary paths would add the script's first config file for little
  gain, since `--mount` already covers arbitrary paths.
- **Host directory, not an engine volume.** The user can inspect it from
  the host, it survives image and engine resets, and ownership comes out
  right on both engines with no extra step: Docker remaps the container
  user to the host UID, Podman maps it with `keep-id`.
- **Mounted at `/shares`, not inside the project or the home.** A target
  inside the project bind mount makes the engine create an empty
  mountpoint directory in the host project tree (root-owned under
  rootful Docker), and a target under `/home/user` gets the recursive
  `chown`. `/shares` avoids both.
- **Access control is the mount mode only.** Both containers run as the
  same host user, so ownership cannot separate them; `:ro` per container
  is what decides who writes.
- **`--share` never creates a share, and `share rm` never deletes
  contents.** A typo in `--share` fails instead of silently mounting a
  fresh empty directory, and removing data stays a deliberate step the
  user takes on the host.
- **`--share-dir` for one-off access.** Sharing part of another agent's
  workspace for a session does not need a named share: `--share-dir
  PATH[:ro|:rw]` mounts any host directory at `/shares/<basename>`, with
  the same `--add-dir` handling. It is read-only by default, unlike
  `--share`, because the usual source is another agent's live working
  tree, where an unintended write lands in someone else's work. Any host
  directory is accepted (restricting it to project directories was
  considered and dropped as not worth the friction); only `/` is refused,
  having no basename, and paths with `:` or `,`, which `-v` cannot
  express. Two shares mapping to the same container path are refused
  rather than silently shadowing each other. A live mount cannot be added
  to a running container, so the flag only applies at start.
- **Access outside the working directory.** Mounting outside the
  project means the tools see shares as external paths. Claude Code
  asks before touching files outside its working directories, so each
  enabled share is passed as `--add-dir /shares/NAME`. `--add-dir` takes
  a variable number of values and would swallow the kickoff prompt if it
  came right before it (checked against the real binary: `claude
  --add-dir /tmp hello -p` reads `hello` as a directory), so it is placed
  first, ended by `--permission-mode`. Each path is single-quoted for
  the inner shell, since `--share-dir` paths can contain spaces or
  quotes. Claude Code does not load
  `CLAUDE.md` from added directories by default, so a share written by
  another container does not become instructions. opencode needs
  nothing: its built-in `external_directory: ask` rule is followed by
  the blanket `"*": "allow"` from `OPENCODE_CONFIG_CONTENT`, and its
  permissions are last-match-wins (checked with `opencode agent list`
  for the `plan` and `build` agents).
- **The agent is not told what shares are for.** No env var or seeded
  instruction mentions them (Claude Code only sees them listed as
  additional directories); the user points the agent at `/shares/NAME`
  when it matters. Keeps the seeded templates unchanged.

## Commits, not pushes

The workflow this tool is designed for is commit-only inside the
container: an agent can commit freely, but pushing (and anything else
that assumes push access, like opening a PR) is meant to happen
outside the container, by the user, after reviewing what was committed.
The point is that the user stays the one responsible for what leaves
their machine.

This is a workflow expectation, not a technical guarantee, and it's
worded that way deliberately: the `credential.helper` exclusion (see
"Git identity forwarding" above) means push over HTTPS will not
authenticate out of the box, but a user can still mount host SSH keys
themselves (`--mount /etc/ssh:/etc/ssh:ro`) and push over SSH; nothing
makes that technically impossible, it's just not what the tool is set
up to encourage.

Since the tool itself has no awareness of this workflow by default,
`.agentcubicle/AGENTS.local.md` carries an explicit instruction telling the
agent never to push or suggest pushing (e.g. Claude Code's own "push to
origin" follow-up suggestion after a commit). This reaches both tools,
not just Claude Code, since it lives in the shared `AGENTS.local.md` rather
than the Claude-specific file. Like any model instruction, this is a
strong nudge, not a hard guarantee.
