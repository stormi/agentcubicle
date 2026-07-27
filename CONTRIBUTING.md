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
- It helps to tidy up messy work-in-progress commits before opening the
  PR, so the history is easy to follow.

Optionally, sign off your commits with `git commit -s` (adding a
`Signed-off-by:` line) to state that you wrote the change and have the
right to submit it under this project's license. It's encouraged, not
required.

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
