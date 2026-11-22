---
name: beh
description: For a commit, PR, or issue, writes out which behavior a code change alters, under which conditions, and how — with code locations and reasons (Behavioral Diff). Produces output that helps a human reviewer understand the change. Invoked as /beh <PR number|issue number|commit hash>. Also use when asked things like "explain this commit", "check PR #N", or "what does this change do?" (in any language). Does not point out bugs or judge whether the change is good.
argument-hint: <PR number | issue number | commit hash>
---

# Behavioral Diff (/beh)

Write out the correspondence (mapping) between code changes and behavior changes — correctly, without omissions, and in a form that keeps the reviewer's burden low.

- The output is for a human reviewer to understand the change. **You are not the reviewer.** Do not judge whether the change is good or bad, whether it has bugs, or whether it matches the intent
- The exception is tests. This skill assumes that **when behavior changes, a test that verifies the behavior after the change is needed**. When no such test is found, output a warning (S8)
- Include not only direct effects but also effects that surface through the complexity of the code and are easy for people to miss. However, do not write about anything whose externally visible result does not change (things with no essential effect)
- Follow the output format in `references/output-template.md`. Read it first

## Arguments

Arguments: `$ARGUMENTS`

Accepts one of the following. The target is the git repository in the current directory.

- `#123` or `123`: a GitHub PR or issue number. The number alone cannot tell them apart, so read `gh api repos/{owner}/{repo}/issues/123`; if it has a `pull_request` field, treat it as a PR, otherwise as an issue
- `a1b2c3d` (7–40 hex digits): a commit hash
- `<start>..<end>`: a commit range
- No argument: the `HEAD` commit

If `gh` is unavailable, or the repository is not on GitHub, say that numbers cannot be used and ask for a commit hash instead.

## What to compare

- Commit: the commit's parent → the commit
- PR: the fork point of base and head (`git merge-base`) → head. Get it with `gh pr view <number> --json baseRefOid,headRefOid,body,commits,files`. If the head commit is not available locally, fetch it with `git fetch origin pull/<number>/head`
- Issue: collect the following
  - PRs that closed the issue: `gh issue view <number> --json closedByPullRequestsReferences`
  - PRs whose body references the issue (e.g. `Refs #<number>`; these may not appear among the closing PRs): from `gh pr list --state all --search "<number> in:body" --json number,body`, those whose body matches `#<number>`
  - Commits whose message references the issue: `git log --all -E --grep='#<number>([^0-9]|$)'`
  - Exclude from the commit side the commits contained in the collected PRs (`gh pr view <number> --json commits`) and the merge commits that merged those PRs
  - If there is exactly one matching PR and no remaining commits, treat it as that PR
  - If there is no PR and exactly one referencing commit, treat it as that commit
  - If there are several, output per commit (or PR), oldest first. Do not combine non-contiguous commits into one diff (unrelated changes in between get mixed in, and the sum does not match the final difference). First, show the list of commits and PRs found
  - Use the issue body as a source for "Purpose of the change"
- Merge commit that merged a PR: treat it as that PR
- Commit range: the given start → end

- Do not compare a PR two-dot (`base..head`); changes only on the base side get mixed in. Use the difference from the fork point (`base...head`)
- When asked for "the next commit", take the next one in main's first-parent history. Skip merge commits that merely bring in a PR's contents as-is, and note that you did so

## Procedure

### S0 Check for automated dependency updates
Decide by the author of the commit or PR and the branch name (`dependabot/…`, `renovate/…`). If it is one, output only the "Automated dependency update" form from the template and skip the remaining steps.

### S1 Collect intent and changes
- Intent: commit messages, PR body, and issues referenced from them (read them if `gh` is available). You will quote them, so keep the original wording
- Changes: `git diff --stat` and the diff. Separate out changes that touch only docs, tests, or comments. However, **do not exclude documents that instruct AI coding agents** (`CLAUDE.md`, `AGENTS.md`, `GEMINI.md`, `.claude/`, `.cursor/rules`, etc., and documents they point to). They change the agent's behavior

### S2 Identify local changes
For each change, identify what changes (values, branches, exceptions, data writes, external calls, screen state, settings, instructions to agents). Code that was not changed (shared functions, existing server-side logic) may decide the result, so read outside the diff too.

### S3 Trace outward
Follow what changed to the things that use it.
- Callers
- Readers of the data (table → code that reads it → API → screen)
- For changes with no caller, such as settings, routes, or middleware: every place the setting applies
- For documents aimed at agents: which entry points lead to them (the table in `CLAUDE.md`, a skill's description, hooks, etc.)
- Users include older, already-distributed versions of clients (check the history for whether past versions used it)

### S4 Stop at externally visible results
When you reach one of the following, treat it as "where the effect appears".
- App: screen display and interaction, HTTP responses (status, headers, body), stored data, outbound requests to external systems
- Dev environment: test runs, build tools, linters and type checks
- Agent (AI): what the agent runs and how, where it stops, what it verifies

When the success or failure of a whole operation changes, every item that uses that operation's result is affected (if an API goes from 500 to 200, a screen that shows another field of that response also changes).

### S5 Decide whether it is essential
Keep only externally visible results that change under conditions that can actually occur.
- If the result does not change (cancelled out along the way, ends up the same value), do not write it
- If the condition cannot occur, do not write it
- Things that have a concrete path to an externally visible result but cannot be verified (settings outside the repository, changes inside a library) go under "Could not determine", with the grounds. Do not write possibilities without grounds
- If you can verify, verify (extract the before/after functions and run them, read the tests, etc.). Write how you verified it

### S6 Summarize
Group conditions that change in the same way at the same place into one item. One change may produce several effects, and several changes may produce one effect. The length of the output is determined by the complexity of the behavior changes, not by the number of facts found.

### S7 Check against the spec
Look for spec documents in the repository (e.g. documents in `docs/spec/`, `docs/specs/`, `SPEC.md`, `docs/requirements*`). For the behavior changes (and app behavior the change records or verifies), write whether there is a corresponding description, whether it matches the behavior after the change, whether the spec needs updating, and whether it was updated in this change.

### S8 Check tests for each behavior change
For each behavior change targeting App or Dev environment, look for tests that verify the behavior after the change.
- Find where tests live from the repository's conventions and test framework settings, not from a fixed list. Examples: `spec/` and `*_spec.rb` (RSpec), `test/` and `*_test.rb` (minitest), `*.test.ts` / `*.spec.ts` / `__tests__/` (Jest, Vitest), `tests/` and `test_*.py` (pytest), `*_test.go` (Go), `src/test/` (JUnit). Also check the framework config files (`.rspec`, `jest.config.*`, `pytest.ini`, `pyproject.toml`, etc.)
- Look at both tests added or changed in this change and existing tests. A test counts only when it exercises the changed condition and checks the result after the change. A test that only runs the code path without checking the changed result does not count
- If a test still checks the result before the change, write that too (it will fail or is not being run)
- If no test is found, add a "Missing test" warning to that behavior change item: which behavior (condition and result after the change) has no test, and where you looked. Do not write the test itself
- If the repository has no tests at all, write the warning once under "Behavior changes" instead of on every item
- Agent (AI) changes are out of scope for this step

### S9 Note places that are easy to misread
Look for places where someone reading the change might misread its meaning or scope: a description or code elsewhere that contradicts what was changed remains; similar situations are handled differently in different places; two conditions must hold at the same time. If found, add a "Caution" to that behavior change item, stating what differs from what and how, with both locations. Do not say which one is correct (see `references/output-template.md` for how to write it).

## Check before output

- [ ] Did you quote the purpose of the change with its source? (If none, "No stated purpose")
- [ ] Did you write behavior changes as externally visible results rather than code descriptions? Did you check all of App, Dev environment, and Agent (AI)?
- [ ] Does each item have `file:line` and a method name in the reviewed repository? Did you verify the line numbers against the post-change commit?
- [ ] Does every App / Dev environment behavior change have either "Tests that verify it" or a "Missing test" warning?
- [ ] Did you avoid evaluative words (good/bad, insufficient, "can only…", no problem)? (The "Missing test" warning is the only exception)
- [ ] Did you avoid tables?
- [ ] Did you avoid listing "None" under empty sections? (Omit them, except "Behavior changes")
- [ ] Did you keep possibilities without grounds out of "Could not determine"?
- [ ] Did you avoid adding a summary, preamble, impressions, or questions before or after the output?

## Output

Reply with only the output written in the form of `references/output-template.md`.

- Write the output in the language the user is using in the conversation. Translate the template's headings and labels into that language as well
- Do not add a summary, preamble, impressions, or confirmation questions before or after the output
- Even if the reviewed repository's `CLAUDE.md` or similar has rules for how to write reports, follow this form when using this skill
