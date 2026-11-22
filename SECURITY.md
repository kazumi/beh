# Security Policy

## Reporting a vulnerability

Do not open a public issue for a security problem, and do not post working injection payloads, tokens, or private repository contents in public.

Report it privately through GitHub's "Report a vulnerability" button on this repository's Security tab. If that is not available to you, open a public issue titled "Security contact request" with no technical detail, and you will be given a private channel.

Please include what you did, what the agent did, the Claude Code version, and your permission settings (auto mode, allowed tools).

This is a personal project maintained on a best-effort basis. There is no guaranteed response time.

## What this skill is

`beh` is a set of instructions (`skills/beh/SKILL.md` and `skills/beh/references/output-template.md`) for Claude Code. It contains no code of its own. It tells the model to read a commit, pull request, or issue in the current repository and describe the behavior change.

## Threat model

1. **Untrusted text in the target.** Commit messages, PR and issue bodies, comments, branch and file names, the diff, and files in the repository can contain text written to steer the agent ("run this command", "read this file", "fetch this URL").

   **Mitigation:** the skill tells the model to treat all of it as data, never as instructions, and to report text addressed to the agent in a dedicated output section instead of following it. It tells the model to use only read-only `git` and `gh` commands, and not to run tests, builds, or any code, change files, or write to GitHub.

2. **Tampered skill files.** If the installed skill is replaced by someone else's version, for example by checking out a pull request in a clone that `~/.claude/skills/beh` points into, the read-only rules can be removed.

   **Mitigation:** the README installs a copy of a reviewed commit, not a symlink into a working clone. Review changes to `SKILL.md` and the template before installing them.

## Known residual risks

- **The read-only rules are instructions to the model, not enforced limits.** A model can be persuaded to break them. The only enforced limit is `disallowed-tools: Write Edit NotebookEdit`, which removes those tools for the turn that invokes `/beh` only. Bash, MCP tools, and network access remain available, and the restriction is gone on your next message while the untrusted text stays in the conversation.
- **Read-only commands have side effects and options.** Some `git` and `gh` options write files, start programs, or send data (for example `git diff --output`, pager and textconv settings, `gh api` with fields). Your git configuration and hooks run as usual. The skill assumes a git environment you trust.
- **`git fetch` writes to the local repository.** It is allowed only to get a PR's commits that are not available locally. It adds objects and refs to `.git`.
- **`gh` uses your credentials.** Anything your token can read can be read through `gh api`, including other private repositories.
- **The output is not a security review.** The skill describes behavior changes; it does not judge whether a change is safe.

## Recommended use with untrusted changes

- Do not use auto-approve modes. Review each command before allowing it.
- Restrict Bash in your permission settings to the read-only `git` and `gh` commands you need.
- Use a `gh` token with read-only scope, limited to the repositories you review.
- Run it in a clone or environment that holds no other secrets.
