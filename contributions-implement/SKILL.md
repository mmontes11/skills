---
name: contributions-implement
description: >
  Implement a chosen contribution in a fork of an open-source project: verify the user's fork
  already exists (stop and ask the user to fork if it does not), strictly follow the upstream
  repository's conventions (CONTRIBUTING.md, AGENTS.md, CLAUDE.md, PR template, commit and
  sign-off format), work on a branch of the fork without opening any fork PR, write
  equivalence-first tests, run the upstream CI/CD checks locally before pushing (the fork will
  not run them), sign commits per the upstream DCO/CLA convention, and post the PR body — built
  from the upstream PR template — in a markdown pasteable format on the Multica issue. Use
  whenever a Multica issue asks to "work on the contribution", "implement <candidate> in
  <owner/repo>", "prepare the branch for <repo>", or to "turn the scouted contribution into a
  PR-ready diff" — even if the issue only names the target repo and the change to make.
license: Apache-2.0
metadata:
  author: mmontes11
  version: "1.0"
compatibility: Requires the multica CLI, a local toolchain for the target repo's language, and GitHub access (GitHub MCP tools and the authenticated gh CLI) to both the upstream repo and the user's fork.
allowed-tools: Read, Grep, Bash(multica:*), Bash(gh:*), Bash(git:*)
---

# Implement a contribution in a fork

One contribution per run. The run's **Multica issue** names the target repository (owner/repo) and
the change to make. The deliverables are a **pushed branch in the user's fork** and a **Multica
comment carrying a pasteable PR body** that follows the upstream PR template. No PR is opened
anywhere — the user opens the upstream PR after validating the branch.

## Scope

- **In:** verify the fork; read the upstream conventions; tests first; implement; run the upstream
  CI checks locally; commit with the correct sign-off; push the fork branch; write the PR body;
  post the deliverable comment.
- **Out (hard):** creating the fork, opening a PR against the fork, opening a PR against upstream,
  merging anything, force-pushing.

## Hard gate: the fork must already exist

The fork is created by the **user**, never by you. Before any other work, verify a fork of the
upstream repo exists under the user's account (e.g. `gh api repos/<user>/<repo>` returning `fork:
true`, or the fork metadata via the GitHub MCP tools). If it does not exist, **stop immediately and
ask the user to fork the repo** — do not fork it yourself and do not start other work before the
fork is there. A fork you create silently is not the same deliverable: the user chose to own the
fork relationship.

## Procedure

1. **Read the upstream conventions before writing any code.**
   - Agent-facing docs at the root — `CONTRIBUTING.md`, `AGENTS.md`, `CLAUDE.md` (whichever exist) —
     plus the issue and PR templates (`.github/PULL_REQUEST_TEMPLATE*`) and any contribution or
     AI-PR policy in the tracker.
   - Commit conventions (Conventional Commits or otherwise), DCO vs CLA, and the CI expectations:
     which commands the upstream workflows run, and which tool versions are pinned (Makefile
     `make lint` targets, `.golangci.yml`, workflow `go-version`, etc.).
   - The code style of the exact files you will touch: naming, error handling, test framework, and
     existing helpers to reuse rather than reinvent.
   If upstream forbids fully AI-generated PRs, or requires a CLA the user has not signed, record it
   and stop — do not push on.
2. **Prepare the branch.** Clone or fetch the fork. Base the new branch on the **upstream default
   branch** (fetch upstream explicitly; do not assume the fork's head is current). Name it
   `feature-<short-name>` or `fix-<short-name>` — never a generic agent-generated branch name.
3. **Tests first, equivalence first.** Before changing behavior, add tests for the paths the change
   touches — especially branches that have no coverage today. Run them against the **unmodified**
   code: they must pass unmodified. A test that only passes after the change proves the new
   structure works; a test that passed before proves the behavior is the same, which is what a
   reviewer needs to trust a refactor.
4. **Implement the change.** One logical change, minimal diff — no unrelated refactors, renames,
   deletions, or formatting tweaks. Match the repo's conventions from step 1 exactly.
5. **Run the upstream CI/CD checks locally, before pushing.** The fork will not run upstream
   CI/CD, so nothing else will. Reproduce what the upstream workflows run: the build, the package's
   test suite (with race/sanitizer flags where the repo uses them), and the repo's lint target with
   the **pinned tool versions** (a lint pass on a newer linter is not the same evidence). Read the
   actual terminal output and confirm it is green; do not assume success from a quiet exit.
6. **Commit with the correct sign-off.** Respect the upstream sign-off convention: for DCO repos,
   sign with the user's identity so the commit carries `Signed-off-by: Name <email>`
   (`git commit -s`). Never change the git identity for the task, and never commit secrets.
7. **Re-read the diff, then push.** Review the generated diff for scope creep, then push the branch
   to the fork's origin. **Do not open a PR against the fork** — a fork PR shows up in the
   upstream's PR timeline and pollutes it with updates the maintainers will never review. The
   pushed branch alone is the deliverable.
8. **Write the PR body from the upstream PR template.** Same sections and checkboxes as the
   template. Include: what it does, the motivation (linked to the upstream issue if one exists),
   the exact testing commands with their results, and an honest AI-assistance disclosure.
9. **Post the deliverable comment** on the Multica issue
   (`multica issue comment add <issue-id> --content-file <path>`). It must carry:
   - The branch name and the fork URL with a link to the branch diff.
   - The verification summary: the exact commands run (tests, lint, CI-equivalent checks) and
     their results.
   - The PR **title and body in a fenced markdown block**, pasteable as-is for the upstream PR.
   - A closing line: **no PR was opened anywhere — the branch sits in the fork, and opening the
     upstream PR is the user's call.**
10. **Close out.** Set the issue status per the platform workflow once the deliverable is posted.

## Conventions this run followed (beyond the steps above)

- **Toolchain pinned to the repo.** Verify the language toolchain the repo requires (e.g. the
  `go.mod`/`GOTOOLCHAIN` requirement, the linter version pinned by the Makefile or CI) before
  running anything — a green build on the wrong toolchain is false evidence.
- **Narrow reads, bounded context.** Read the narrowest slice needed — one path, one line range,
  one diff — and summarize learnings before fetching more. Never dump whole files or unpaginated
  logs into the conversation.
- **One branch per logical change.** If the work spans several independent changes, do one per
  branch and report them separately, so the user can open, sequence, or discard them independently.
- **Verification is evidence, not ritual.** Every command's real output must be inspected
  (tests passing, lint "0 issues", build silence), and the summary must quote what was run and
  what it reported — a "tests pass" claim without the command and the observed result is not a
  verification.
- **Stop at the deliverable.** The moment the branch is pushed and the comment is posted, the work
  is done. Wait for the human to validate and open the upstream PR; do not chase CI, reviews, or
  merges on your own.
