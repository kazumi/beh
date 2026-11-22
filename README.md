# beh — Behavioral Diff for code review

A Claude Code skill that, for a commit, PR, or issue, writes out which behavior a code change alters, under which conditions, and how — with code locations and reasons.

It produces output that helps a human reviewer understand the change. It does not point out bugs or judge whether the change is good.

## Usage

In the directory of the repository you want to review, invoke it from Claude Code:

```
/beh #123        # PR or issue number (the # is optional)
/beh a1b2c3d     # commit hash
/beh v1.2..v1.3  # commit range
/beh             # the HEAD commit
```

- For a PR, it looks at the difference from the fork point with base to head
- For an issue, it collects the PRs that closed it, PRs whose body references it, and commits whose message references it, and outputs per PR or commit
- For automated dependency updates (Dependabot, Renovate, etc.), it outputs only the updated packages and versions

The output is written in the language you use in the conversation.

The skill is read-only: it reads the change and the repository, and does not run tests, builds, or any code, change files, or write to GitHub. It may run `git fetch` to get a PR's commits that are not available locally. Text in commits, PRs, issues, the diff, and other files it reads is treated as data, not instructions; text that tries to instruct the agent is reported in the output. Only the file-editing tools (`Write`, `Edit`, `NotebookEdit`) are removed by the skill's `disallowed-tools`, and only for the turn that invokes it; the rest are instructions to the model, not enforced limits; when using it on changes from people you do not trust, restrict tool permissions (for example, allow only `git` and `gh` read commands in Bash) and do not use auto-approve modes. See `SECURITY.md` for the threat model and known residual risks.

## Output

Purpose of the change (quoted from the commit message, PR, or issue), what changed, behavior changes (before → after, code location, why, tests that verify it or a missing-test warning, cautions), unchanged behavior, spec alignment, changes with no behavioral effect, and what could not be determined.

Behavior changes are written for three targets:

- App: screen display and interaction, HTTP responses, stored data, outbound requests to external systems
- Dev environment: test runs, build tools, linters and type checks
- Agent (AI): the behavior of AI coding agents, changed by changes to `CLAUDE.md`, `AGENTS.md`, `.claude/`, etc.

See `skills/beh/references/output-template.md` for the detailed format.

## Requirements

- Claude Code
- git
- `gh` (GitHub CLI), logged in, when specifying GitHub PR or issue numbers

## Installation

To use `/beh` in any repository, copy the skill directory to `~/.claude/skills/beh`. Copy a commit you have reviewed, and do not symlink a clone you work in.

```sh
git clone https://github.com/kazumi/beh.git /tmp/beh
git -C /tmp/beh log -1            # check the commit you are about to install
mkdir -p ~/.claude/skills
cp -R /tmp/beh/skills/beh ~/.claude/skills/beh
```

Why a copy: the skill is read from wherever it is installed. If `~/.claude/skills/beh` points into a clone and you check out someone's pull request there, the next `/beh` runs with their version of `SKILL.md` and the template, including any change that removes the read-only rules. To update, repeat the steps above with a newer commit.

To use it in a single repository only, place a copy at `.claude/skills/beh` in that repository.

## Structure

```
skills/beh/
├── SKILL.md                      procedure (argument parsing, what to compare, tracing effects, pre-output checks)
└── references/
    └── output-template.md        output template and writing rules
```

## License

MIT. See `LICENSE`.
