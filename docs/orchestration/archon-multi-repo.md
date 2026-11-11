# Archon multi-repository setup (proposal)

Status: **proposal**, except the per-repository base branch config (section 1), which
is applied. Based on Archon v0.14.0 documentation and local validation on 2026-11-11.

Each statement below is marked:

- **Verified**: tested locally or stated in the official v0.14.0 docs.
- **Proposed**: a convention or custom workflow we would have to build. Archon does not
  provide it.

## Repositories

| Role | Local path | GitHub |
| --- | --- | --- |
| Planning (BMAD) | `bar-inventory-workspace/` | `kaanf/bar-inventory` |
| Backend | `bar-inventory-api/` | `kaanf/bar-inventory-api` |
| Mobile | `bar-inventory-kmp/` | `kaanf/bar-inventory-kmp` |

The parent directory `~/Projects/bar-inventory` is not a git repository.

## What Archon supports natively

### Per-repository runs with worktree isolation (verified)

Every git repository is its own Archon *repo project*. A run gets a separate git
worktree on its own branch under `~/.archon/workspaces/<owner>/<repo>/worktrees/`.
Your main checkout is not touched.

Locally verified on `bar-inventory-api`: a bash-only workflow launched with
`--branch archon-smoke/worktree --base dev` ran in an isolated worktree cut from
`dev` (`416d8f0`). The main checkout stayed on `dev` and clean. `archon complete`
removed the worktree and local branch, and nothing was pushed.

### Base branch defaults to `main` (verified, now configured)

When no base is configured, Archon uses the remote's symbolic HEAD. For all three
repositories that is `main`. Each repository now sets `worktree.baseBranch: dev`.
See [archon-repo-config.md](archon-repo-config.md) for the exact behavior, including
when `--base dev` is still needed.

`--base` sets both the branch a worktree is cut from and the PR target
(`$BASE_BRANCH`). `--from` overrides only the cut-from branch.

### Folder project for cross-repository reading (verified in docs)

`~/Projects/bar-inventory` can be registered as a *folder project* with `--folder`.
Runs then execute in place at the root, so an agent can read all three repositories.

Limits (from the docs):

- Archon creates no worktree, branch or PR for a folder project.
- `--branch` and `--from` are rejected.
- Git work in each child repository is the agent's job, done through workflow prompts.
- A `workflow:` child with `isolation: worktree` fails in a folder project.

### GitHub integration (verified)

The `forge-github` plugin (v0.14.0) resolves repositories and reads PR checks and
state through the existing `gh` login (`credentialSource: gh auth token`). No token
had to be added.

### Things that are present but must stay off

- **SDLC pack queues:** `archon-merge-queue` merges PRs and `archon-build-queue` claims
  issues on its own.
- **Workflow triggers:** schedules and issue events. The pack binds none by default.
- **`archon serve`:** the background server.

Using any of these would conflict with the current policy: no auto-merge, no
scheduled agents, and manual release approval.

## What Archon does not provide

- **Coordinated branches or PRs across repositories.** The multi-repo guide says
  Archon does not create a composed multi-repo worktree.
- **Dependency ordering between repositories** (for example, API before KMP).
  Neither the docs nor the CLI have a construct that blocks a run in one project on
  work in another.
- **Child runs in another repository.** A `workflow:` node runs in the parent's
  project. Its target is a static workflow name, not a different codebase.

## Proposed configuration

### 1. Per-repository `.archon/config.yaml` (applied)

Each of the three repositories has this file, added through a `feat/archon-config`
PR into `dev`:

```yaml
# .archon/config.yaml
worktree:
  baseBranch: dev
```

Add `copyFiles` and `postCreate` once the API and KMP projects exist and their setup
commands are known. Verified behavior and portability requirements:
[archon-repo-config.md](archon-repo-config.md).

### 2. How work runs (proposed convention)

| Phase | Where | Tool |
| --- | --- | --- |
| Discovery, PRD, architecture, epics | `bar-inventory-workspace` | BMAD skills in Claude Code, by hand |
| Backend story | `bar-inventory-api` repo project | `archon workflow run archon-plan` / `archon-implement` / `archon-pr` with `--branch feat/<story> --base dev` |
| Mobile story | `bar-inventory-kmp` repo project | same as backend |
| Cross-repo questions | folder project at `~/Projects/bar-inventory` | read-only workflows such as `archon-investigate` |

PRs stay per repository. Merges into `dev` and releases into `main` remain manual.

### 3. Dependency-aware backend/mobile flow (proposed custom work)

Archon has no built-in dependency handling across repositories, so this needs custom
pieces:

1. In the workspace, BMAD epics and stories record each story's repository and its
   dependencies, for example "KMP story X depends on API story Y".
2. Each story becomes a GitHub issue in its own repository, with a "Depends on
   kaanf/bar-inventory-api#N" line. This is our own convention, not something
   Archon reads.
3. A custom gate workflow, or a `bash:` node at the start of a KMP workflow, runs
   `gh pr view` / `gh issue view` against the API repository. It refuses to continue
   until the dependency is merged into `dev`.
4. The API contract, such as OpenAPI, is published from the API repo and consumed
   by KMP at a pinned version. This is an architecture decision for BMAD to make
   later.

None of this is built yet.

## Open questions

- Whether to register the parent folder as a folder project at all, or keep runs
  strictly per repository.
- Whether to install the Archon skill (`archon skill add`) into each repository's
  `.claude/skills/` and `.agents/skills/`. Doing so writes files into the repositories.
- Telemetry: Archon sends anonymous usage events by default. Opt out with
  `DO_NOT_TRACK=1` or `ARCHON_TELEMETRY_DISABLED=1`, which would require an env
  change.
