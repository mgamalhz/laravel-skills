# Working on this repository

Agent skills, one folder per skill under `skills/`, each with a `SKILL.md`
(Agent Skills format). The repo is also a Claude Code marketplace: the plugin
`laravel-engineering` is the repo root and exposes every folder in `skills/`.

## Rules

- English only — skills, docs and commit messages.
- Commits are a single line, `type(module): message`, with no body or footer.
- A skill states when to use it in its `description`, gives rules the agent can
  act on (exact APIs, not slogans), and ends with how to verify the work.
- Every rule must be correct for the framework version the skill targets.
  When unsure, check the docs before writing it.
- Keep `SKILL.md` focused; move long reference material into `references/`.
- Don't copy third-party skills here; link to their source in the README.
- After any change: `claude plugin validate .` must pass, and the README table
  must list every skill.
