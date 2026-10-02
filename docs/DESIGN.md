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
  models, by opposite mechanisms.
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
what to do with the resulting token and drops them into an interactive
shell to do it, rather than exiting the container out from under them.

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

## rtk command compaction (opencode only)

opencode sessions run with [rtk](https://github.com/rtk-ai/rtk) wired in:
before it executes a bash/shell command, the command is rewritten to its
`rtk` equivalent, so the output rtk produces (already compacted for git,
tests, linters, `ls`, `grep`, and a few dozen other commands) is what
reaches the model rather than the raw output. This cuts the input tokens
spent on command output, which is the one part of the bill agentcubicle
can influence without touching the prompt or the model.

Two deliberate choices are easy to assume incorrectly:

- **The rewrite rules live in the `rtk` binary, not here.** The thing
  agentcubicle installs is rtk's own OpenCode plugin, a thin shim whose
  only job is to call `rtk rewrite` on each command and use the result if
  it changed. The actual per-command rewrite table lives in rtk's Rust
  source and is updated by the rtk project, not by agentcubicle.
  Consequence: agentcubicle's embedded plugin should only ever change if
  rtk's plugin file itself changes, and even then it is a verbatim copy of
  rtk's `hooks/opencode/rtk.ts`, not a reimplementation. If rtk's plugin
  file and the embedded copy ever drift, the embedded copy is the one to
  update to match rtk, never the reverse.
- **The plugin is injected per-run, never written to the host or the
  project.** It is base64-encoded into an env var and written by the
  entrypoint into the container's *writable* opencode config dir
  (`/home/user/.config/opencode/plugins/rtk.ts`), the same pattern as the
  opencode config redaction above. It is not added to the read-only host
  config mount and not placed in the project, so neither the user's real
  `~/.config/opencode` nor the repo is ever modified. Because the container
  `$HOME` is throwaway, this must happen on every run rather than once.

rtk's binary is baked into the image from an official release tarball, the
same way `gh` is (no apk package exists for it), with the target triple
chosen per architecture (`x86_64-unknown-linux-musl` /
`aarch64-unknown-linux-gnu`; the musl build is used because the image is
Alpine/musl). It is therefore present regardless of which tool is run, but
only opencode activates it.

It is **opencode-only on purpose.** rtk supports other agents through their
hook/plugin mechanisms, but wiring one of those in for Claude Code would be
a second integration surface to keep current with no proportional benefit
here; the claude path is left untouched. If that changes, the claude
integration should get its own section rather than being folded into this
one.

Two escape hatches, both deliberate: `--no-rtk` disables the plugin for a
single run (the binary stays in the image; only the plugin is skipped), and
rtk's telemetry is forced off in the container by default
(`RTK_TELEMETRY_DISABLED=1`), overridable to `0` with `--env` for anyone
who wants to opt rtk's anonymous usage metrics back on. Defaulting to
disabled keeps a throwaway container from phoning home without the user
asking for it.

## ponytail coding ruleset (opencode only)

opencode sessions run with [ponytail](https://github.com/DietrichGebert/ponytail)
loaded: a lazy-senior-dev ruleset whose whole thesis is "the best code is the
code you never wrote" (a YAGNI/reuse/stdlib ladder, root-cause bug fixing,
deletion over addition, and `ponytail:` comments marking any deliberate
corner). It is a prompt-level nudge, not a mechanical transformation, so it
affects the shape of the changes an agent produces rather than the bytes of
any command output — the opposite end of the token budget from rtk (which
shrinks what *enters* the model, ponytail shapes what the model *writes*).

The load-bearing fact that makes this integration cheap: **ponytail ships no
binary and no plugin, only instruction text.** rtk's integration has to bake a
binary into the image, base64 a plugin into an env var, and write that plugin
into the container's opencode config dir. ponytail needs none of that: the
only thing agentcubicle does is make sure opencode actually reads it. So the
two rtk-specific mechanics do not apply and there is no image change, no
env var, and no entrypoint change for ponytail.

Two deliberate choices are worth stating, because the obvious alternatives are
easy to assume:

- **The ruleset is written into the project's `.agentcubicle/` dir, not the
  container's opencode config dir.** rtk's plugin lives in the throwaway
  `$HOME` and must be re-written every run (base64 env var → entrypoint →
  file) because that location dies with the container. ponytail's file instead
  sits at `.agentcubicle/ponytail.md`, inside the read-write project mount,
  which is *already* mounted into the container — so it needs no env var, no
  entrypoint write, and no image change of its own; it simply shows up in the
  container view as part of the project. It is git-excluded for free (the
  existing `/.agentcubicle/` exclude entry covers it), so it never leaks to
  anyone who clones. The trade-off is that, because it is seeded once and then
  left alone (see "Local agent files" above), a user who edits
  `.agentcubicle/ponytail.md` in one run keeps that edit on subsequent runs —
  the same "edits persist, never clobbered" contract as `AGENTS.local.md`.
  agentcubicle's embedded copy is the source of truth only for the *first*
  seed; if ponytail's upstream `AGENTS.md` and the embedded copy drift, the
  embedded copy is the one to update to match upstream, never the reverse.
- **It reaches opencode through the existing `instructions` channel, not a
  new mechanism.** opencode already reads `.agentcubicle/AGENTS.local.md` and
  `MEMORY.local.md` via the `instructions` array in the container-only
  `OPENCODE_CONFIG_CONTENT` override (see the "Local agent files" section
  above). ponytail's file is appended to that same list, conditionally, when
  enabled. This is purely additive: opencode merges `instructions` from the
  override with a file-based config's own array, so it composes with a project
  that also has its own `AGENTS.md`.

It is **opencode-only on purpose**, for the same reason as rtk: ponytail is
tool-agnostic instruction text (its own adapters are just "put this in your
rules file"), so there is no *technical* blocker to also loading it for Claude
Code — but the claude rules live in the static, user-owned
`CLAUDE.local.md` `@`-imports (see "Local agent files"), and wiring a per-run
`--no-ponytail` toggle through a file the user owns is a second integration
surface to keep current for no proportional benefit. The claude path is left
untouched; if that ever changes, it gets its own section.

Two escape hatches, both deliberate: `--no-ponytail` skips both the file write
and the `instructions` entry for a single run (nothing is deleted from disk;
a previously seeded file is simply not loaded), and the file being seeded once
rather than per-run means a user can hand-tune ponytail for their project and
not have agentcubicle revert it.

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
