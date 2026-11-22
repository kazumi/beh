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

The skill is meant to only read: it does not run code, change files, or write to GitHub (except `git fetch` to get a PR's commits, which updates `.git`), and treats text in the target as data, not instructions. Provided as is under the MIT License. It is a set of instructions to a model; the model may not follow them, including the read-only rules.

## Output

Purpose of the change (quoted from the commit message, PR, or issue), what changed, behavior changes (before → after, code location, why, tests that verify it or a missing-test warning, cautions), unchanged behavior, spec alignment, changes with no behavioral effect, and what could not be determined.

Behavior changes are written for three targets:

- App: screen display and interaction, HTTP responses, stored data, outbound requests to external systems
- Dev environment: test runs, build tools, linters and type checks
- Agent (AI): the behavior of AI coding agents, changed by changes to `CLAUDE.md`, `AGENTS.md`, `.claude/`, etc.

See `references/output-template.md` for the detailed format.

## Requirements

- Claude Code
- git
- `gh` (GitHub CLI), logged in, when specifying GitHub PR or issue numbers

## Installation

```sh
git clone https://github.com/kazumi/beh.git ~/.claude/skills/beh
```

To update, run `git pull` in `~/.claude/skills/beh`. To use it in a single repository only, clone it to `.claude/skills/beh` in that repository.

## Structure

```
SKILL.md                          procedure (argument parsing, what to compare, tracing effects, pre-output checks)
references/
└── output-template.md            output template and writing rules
```

## License

MIT. See `LICENSE`.
