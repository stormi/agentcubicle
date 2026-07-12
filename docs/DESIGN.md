# Design

Durable design rationale and architecture decisions for `agentcubicle`
— the "why" behind non-obvious choices, meant to outlive any single
session. This is git-tracked and permanent, unlike
`.agentcubicle/MEMORY.md` (private, ephemeral work memory) or
`README.md` (user-facing usage docs). See `.agentcubicle/AGENTS.md`'s
"Memory" section for how these three are meant to relate.

Deliberately avoids citing specific commit hashes or session dates —
those rot (this repo's entire history was rewritten once already,
invalidating every hash cited anywhere at the time) or become
irrelevant. `git log`/`git blame` are the authoritative source for
"when" and "which commit"; this file is for "why," which doesn't change
just because the code around it does.

## Naming

Named `agentcubicle`, renamed from an earlier `opencode-docker` — chosen
after checking for GitHub name collisions, over roughly twenty other
rejected candidates that were already taken.

## Container `$HOME` vs. the project directory

The container's `$HOME` (`/home/user`) and the bind-mounted project
directory (`/home/user/project`) are deliberately separate paths, not
overlaid (an earlier design conflated them as `/work`). `$HOME` is
throwaway scratch space discarded when the container exits; the project
directory is the user's real, persisted work. Keeping them as distinct,
clearly-named paths means it's never ambiguous which files are
ephemeral container state and which are the user's actual files —
important both for correctness (nothing important should ever be
written under `$HOME`) and for the mount-narrowing decisions below
(scoping recursive operations like `chown -R` to skip the project mount
relies on it being a clearly separate subtree).

## Claude Code project state persistence

Claude Code normally keeps per-project memory, transcripts, and
worktree state under `~/.claude/projects/<encoded-path>/` — but every
container gets a fresh throwaway `$HOME`, so that would be wiped every
run. `CLAUDE_CONFIG_DIR` is instead pointed at
`/home/user/project/.agentcubicle/claude-home`, inside the read-write
project mount, so it persists across container recreations without
ever giving the container host access outside the project directory —
a hard constraint from the project's inception, not something added
later for convenience.

## Git identity forwarding

A curated **allowlist** (not a blocklist) of the host's
`git config --global` settings is forwarded into the container, so
commits made by an agent carry correct authorship: `user.name`,
`user.email`, `init.defaultBranch`, `pull.rebase`, `push.default`,
`push.autoSetupRemote`, `core.editor`, `color.ui`, `rerere.enabled`,
`merge.conflictstyle`, and all `alias.*` entries. `credential.helper`
and any GPG/commit-signing settings are never forwarded — not because
they're blocked by a rule that could miss something, but because an
allowlist means there's nothing to accidentally leak in the first
place. Path-valued settings (`core.excludesfile`, `core.hooksPath`)
are skipped too, since honoring them would require mounting a host file
outside the project directory.

The forwarded values are decoded straight to a file and pointed at via
`git config --global include.path`, rather than reconstructed as
individual `git config` calls built from arbitrary text. Reconstructing
config from arbitrary values means re-parsing that text as shell
through the container entrypoint's `su -c` invocation — exactly the bug
class that has bitten this script for real (a corrupted JSON blob from
a double shell-reparse, and a stray apostrophe in a single-quoted
region prematurely terminating a string). Decoding a fixed blob
straight to a file sidesteps that class of bug entirely rather than
requiring careful escaping to avoid it.

## Local agent files: `AGENTS.md`, `CLAUDE.md`, `MEMORY.md`

These live under `.agentcubicle/` in the project, are seeded once and
never overwritten, and are **additive** — they apply even when the
project already has its own tracked `AGENTS.md`/`CLAUDE.md`, which
continue to load normally through each tool's own discovery.

They're deliberately never placed at the literal `AGENTS.md`/`CLAUDE.md`
paths, even as a `.git/info/exclude`d symlink: if the upstream remote
ever adds a real tracked file at that exact path, git refuses to
silently overwrite an existing untracked/excluded file during a merge
or pull — a working symlink today would break the next `git pull`
later. `CLAUDE.local.md` (a real symlink to `.agentcubicle/CLAUDE.md`)
is safe by contrast because it's Claude Code's own documented
convention for exactly this purpose, not a path something else might
someday claim.

`CLAUDE.md` `@`-imports `AGENTS.md` and `MEMORY.md` directly, rather
than a prose instruction telling the model to go read them separately.
This closes an architectural asymmetry: opencode's pickup of these
files is deterministic (a config array it always reads), while a prose
instruction could in principle be skipped under context pressure.
`@`-import makes Claude Code's pickup deterministic too.

Every run auto-sends a short kickoff prompt as the tool's first turn
(opencode's `--prompt`, Claude Code's positional `[prompt]` argument),
asking the agent to announce what it read. A passive instruction inside
`AGENTS.md`/`CLAUDE.md` alone is not sufficient for this: an interactive
TUI never gives the model an unprompted turn to act on it, so without
an explicit first message, the instruction just sits there unused until
the user happens to say something first.

### The three-tier documentation model

`.agentcubicle/MEMORY.md` is temporary work memory only — current
focus, recent decisions not yet written down elsewhere, and open
threads — not a place for anything meant to last. `README.md` is
user-facing usage documentation. This file is durable design rationale.
Content is expected to flow from the first into the other two over
time (a decision gets made and noted in MEMORY.md, then later promoted
into README.md or here once it's stable, and trimmed out of MEMORY.md
accordingly) rather than accumulating indefinitely in the ephemeral
one. See `.agentcubicle/AGENTS.md`'s "Memory" section for the actual
rule an agent follows.

## Claude commit trailer

`agentcubicle` does not force-suppress Claude Code's
`Co-Authored-By: Claude` commit trailer. Whether to see that trailer in
commit history is a host-level, user-level preference — not something
a container-launching tool should decide on the user's behalf. The
README documents how to disable it on the host if wanted, but the tool
itself stays deliberately silent on the matter ("discreet about git").

## Claude Code first-run authentication

The container never receives the host's live Claude Code credentials —
only `~/.claude/settings.json` is ever mounted (read-only) and copied
out; the raw `.credentials.json` (a single-use, rotating refresh token)
is never mounted or copied at all, since a read-only copy would go
stale the moment it's refreshed anywhere. Instead, a long-lived static
token from `claude setup-token` is forwarded as an environment variable
(`CLAUDE_CODE_OAUTH_TOKEN`), read from `~/.claude/oauth-token` on the
host (or an already-exported variable, or `ANTHROPIC_API_KEY`). A
static token has no rotation problem, which is what makes it safe to
reuse across many container runs.

`agentcubicle claude` detects, host-side, before the container even
starts, whether any of those credential sources will actually resolve.
If none will, it skips the normal session entirely — starting one would
just hit a login wall — and instead runs `claude setup-token` for the
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
container's mount namespace too — redundant exposure serving no
purpose. The single-file mount is guarded on the file's existence,
since Docker creates an empty directory at the host path if you
bind-mount a single file that doesn't exist yet, which would corrupt a
future `claude setup-token` save into that same location.

## opencode provider and model configuration

opencode's own host config (`~/.config/opencode/opencode.json` or
`.jsonc`) already propagates into the container — mounted read-only and
copied in on every run — so provider/model configuration is edited on
the host exactly like a native install, no agentcubicle-specific step
needed. Without any provider configured, opencode falls back to its own
built-in free models.

Two things are true here that are easy to assume incorrectly:

- **`opencode auth login` does not work with agentcubicle.** It stores
  credentials at `~/.local/share/opencode/auth.json`, a completely
  different directory from `~/.config/opencode`, which agentcubicle
  never mounts or copies into the container. Anything saved there never
  reaches a session — this was checked directly (`opencode debug
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
informational and never blocks a session from starting — the free
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
doesn't have to hand-convert it to the `{env:VAR}` shape — nor type it
into a `--env` flag, where it would land in shell history. When
redaction applies, the raw config file is not mounted into the container
at all: the redacted copy is injected via a base64 env var and written
by the entrypoint, so even the container's read-only mount namespace
never sees the literal.

This is only attempted when `jq` can parse the config. A `.jsonc` file
with comments (which `jq` cannot parse) is deliberately left untouched
and falls back to the plain mount-and-copy path — attempting a
regex-based rewrite risks silently corrupting the config or failing to
redact, which would be worse than not redacting at all. Detection is
per-provider and keys off literal-vs-reference, so configs already using
`{env:...}`/`{file:...}` are unaffected.

## Scope: opencode session state is not persisted

Claude Code's per-project state is persisted via `CLAUDE_CONFIG_DIR`
(see "Claude Code project state persistence" above), but there is
deliberately no equivalent for opencode. opencode keeps its data and
state — sessions/history, provider credentials (`auth.json`), and other
state — under `~/.local/share/opencode` and `~/.local/state/opencode`,
both of which live under the container's ephemeral `/home/user` home and
are therefore discarded when the container exits.

This is a conscious scope decision, not an oversight: persistence was
built for claude because that is what was needed, and it is unclear
opencode has per-project state worth the same treatment. If it turns out
to matter, the same pattern applies — point opencode's data/state dirs
(via `XDG_DATA_HOME` / `XDG_STATE_HOME`, or bind mounts) at a location
inside the project mount, never at host paths outside it.

## Commits, not pushes

The workflow this tool is designed for is commit-only inside the
container: an agent can commit freely, but pushing — and anything else
that assumes push access, like opening a PR — is meant to happen
outside the container, by the user, after reviewing what was committed.
The point is that the user stays the one responsible for what leaves
their machine.

This is a workflow expectation, not a technical guarantee, and it's
worded that way deliberately: the `credential.helper` exclusion (see
"Git identity forwarding" above) means push over HTTPS will not
authenticate out of the box, but a user can still mount host SSH keys
themselves (`--mount /etc/ssh:/etc/ssh:ro`) and push over SSH — nothing
makes that technically impossible, it's just not what the tool is set
up to encourage.

Since the tool itself has no awareness of this workflow by default,
`.agentcubicle/AGENTS.md` carries an explicit instruction telling the
agent never to push or suggest pushing (e.g. Claude Code's own "push to
origin" follow-up suggestion after a commit). This reaches both tools,
not just Claude Code, since it lives in the shared `AGENTS.md` rather
than the Claude-specific file. Like any model instruction, this is a
strong nudge, not a hard guarantee.
