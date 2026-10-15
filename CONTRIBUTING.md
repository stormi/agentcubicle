# Contributing to agentcubicle

Thanks for your interest. This is a personal project I share in case it's
useful to others. I'm happy to take contributions, but I maintain it around
my own needs and my own view of what it should be, so it's worth reading
this before you invest much time. Mostly I'd like to save you from sinking
hours (and tokens) into work I end up not merging because it doesn't quite
fit how I see the project.

## Talk before you build

For anything beyond a small, self-evident fix, please open an issue first
so we can agree on the shape of the change *before* you write it.

I would much rather spend ten minutes talking an idea through in an issue
than have you show up with a large PR I have to turn down. If in doubt, ask
first.

Small stuff is fine to send straight as a PR without an issue.

## AI is a tool, but a human owns the change

You're very welcome to use AI assistants; that's rather the point of this
project. But a few things here are firm, not preferences:

- **No agent-only PRs.** A human must own the change: you have read every
  line, you understand why each part is there, and you can explain and
  defend it in review. If you can't explain it without the AI, it isn't
  ready.
- **You are responsible for what you submit**, exactly as if you'd typed
  every character yourself. "The AI wrote it" is not an answer to a review
  question.
- **PR discussion is human to human.** Reply to review comments in your own
  words. Please don't paste agent output back as your side of the
  conversation.
- **No AI co-author trailers** (for example `Co-Authored-By: Claude`). You
  are the author and owner of the commit; the AI is a tool you used, not a
  co-author.

## Commits

Good commit messages explain **why** before **what**. The diff already
shows what changed; what it can't show is the reason, and that's the part
I (and future-you) will actually need later.

- Lead with the motivation: the problem, the need, or the decision behind
  the change. Then describe what you did.
- Short imperative subject line (for example "Skip chown on the project
  mount"), then a blank line, then a body explaining the reasoning.
- **One logical change per commit.** Try not to bundle an unrelated fix, a
  rename, and a feature together. Keeping separate concerns in separate
  commits makes history easier to read, and lets me review (or revert)
  each piece on its own.
- **Docs go with the change they describe.** `README.md` is the reference
  for how the tool behaves, so a change in behavior updates it in the same
  commit. The same goes for the reasoning behind a non-obvious choice,
  which belongs in `docs/DESIGN.md` rather than only in the commit
  message. A separate "update docs" commit is harder to review and to
  revert as one unit.
- It helps to tidy up messy work-in-progress commits before opening the
  PR, so the history is easy to follow.

Optionally, sign off your commits with `git commit -s` (adding a
`Signed-off-by:` line) to state that you wrote the change and have the
right to submit it under this project's license. It's encouraged, not
required.

## Before you commit

CI (`.github/workflows/engines.yml`) builds the image and runs the tool
under both podman and docker. Please check that it will pass before you
open a PR:

- `bash -n agentcubicle` for syntax.
- `shellcheck agentcubicle` should report no finding that `main` doesn't
  already have.
- With docker or podman installed, run what the workflow runs:
  `./agentcubicle setup`, `./agentcubicle check`, and a run like
  `AGENTCUBICLE_TEST_CMD='true' ./agentcubicle opencode`. CI also runs on
  pushes to your fork, which gives you both engines.
- Without an engine, you can still catch unintended changes: put small
  fake `docker`/`podman` scripts that log their arguments first in your
  `PATH`, run the same commands with `main`'s script and with yours, and
  compare the logs. Only what you meant to change should differ.

Two traps specific to this project:

- **Quoting in the container startup script.** The script that runs first
  in the container is passed to `$ENGINE run ... -c '...'` as one
  single-quoted string (near the end of `agentcubicle`, once per engine).
  A `'` anywhere in it, including in a comment ("doesn't"), ends the
  quoting and breaks the command, and `bash -n` won't always see it.
  Check for one after any change there.
- **How Claude Code and opencode behave.** When a change depends on what
  one of the tools does (where it reads its config, how it parses an
  option), check it against the real binary rather than its docs alone.
  Doc reading has been wrong here more than once.

In the PR, say what you could not test, for example "only checked with a
fake engine" or "not tried on Wayland". I'd rather know than assume.

## Changing the seeded templates

On the first run in a project, agentcubicle writes starter rule files into
its `.agentcubicle/` directory, from templates inside the script
(`AGENTS_CONTENT`, `CLAUDE_CONTENT`, `MEMORY_CONTENT`). Those files are
written once and never overwritten, since users edit them. So a template
change only reaches projects seeded after it, and the PR should say
whether existing users need to do anything.

If you run agentcubicle on its own repository, your local
`.agentcubicle/` files are a separate copy: editing them doesn't change
the templates, and editing the templates doesn't change them.

## Reporting bugs and security issues

For a normal bug, open an issue with enough detail to reproduce it: your
OS, Docker version, the exact command you ran, and what happened versus
what you expected.

For anything security-sensitive, please contact me privately rather than
opening a public issue. And to be clear about scope: this tool sandboxes
an agent against obvious mistakes, it is not a security boundary you should
rely on. Container isolation can be escaped, so "the container isn't a
perfect jail" is a known limitation rather than a vulnerability, and isn't
something to report.
