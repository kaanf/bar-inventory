# Archon repository configuration

Applies to Archon v0.14.0 (binary `8def159a`). Behavior was verified locally on
2026-11-11 with bash-only probe workflows: no AI, no edits, nothing pushed.

## The file

Every repository that Archon runs against carries `.archon/config.yaml`:

```yaml
worktree:
  baseBranch: <integration-branch>
```

For this project the integration branch is `dev` in all three repositories.

## Supported settings

The v0.14.0 configuration reference documents these `worktree` keys: `baseBranch`,
`remote`, `path`, `copyFiles`, `initSubmodules`, `postCreate`, `postCreateTimeout`.
Workflow YAML also has `worktree.enabled`. Only `baseBranch` is used here and verified
by behavior.

The repository config has **no** settings for:

- branch naming patterns or prefixes (such as `feat/*`)
- a PR target separate from `baseBranch`
- auto-merge, deployment, triggers or schedules

Archon keeps these off by not running them. The SDLC pack's queues, `archon trigger`
and `archon serve` all have to be started explicitly.

## Verified behavior

| Situation | Worktree cut from | `$BASE_BRANCH` (PR target) |
| --- | --- | --- |
| No config (before this change) | remote HEAD (`main`) | `main` |
| Config only in the launch checkout, not on the cut-from branch | `dev` | **`main`** |
| Config committed on the branch the worktree is cut from | `dev` | `dev` |
| `--base dev` on the command line | `dev` | `dev` |

The two values come from different places:

- **Cut-from branch:** read from `.archon/config.yaml` in the registered checkout when
  the worktree is created.
- **`$BASE_BRANCH`:** resolved from the config inside the new worktree. The file must
  therefore be committed on the integration branch itself.

**Consequences:**

- **Until the config PR is merged into `dev`,** pass `--base dev` on every run.
- **After the merge,** every worktree cut from `dev` contains the file and resolves `dev`.
- **Feature branch names** are not generated. Pass `--branch feat/<name>`; without it,
  Archon names the branch `archon/task-<workflow>-<timestamp>`.
- **Probe branch names must be unique across all projects.** `archon complete <branch>`
  matches a branch name in every registered project and refuses when it is ambiguous.

## Portability requirements for a project generator

A future generator should write this file from a project manifest. It must not
hard-code project names, owners or branch names.

**The manifest must provide, for each repository:**

- **Identity:** a stable id, the git remote URL and the local path.
- **Branch model:** the integration branch (here `dev`) and the release branch (here
  `main`). The generator must not assume either name.
- **Feature branch prefix** (here `feat/`). Archon cannot enforce it, so the prefix is
  used only when composing `--branch` for runs.
- **Worktree provisioning:** optional `copyFiles`, `postCreate` and `postCreateTimeout`,
  per repository and stack.
- **Remote name:** only when it is not `origin`.

**The generator must:**

1. Emit only keys documented for the installed Archon version. How that version
   handles unknown keys was not verified.
2. Record the Archon version the template was validated against.
3. Validate each generated file by running a bash-only probe workflow. The worktree
   must be cut from the integration branch, and `$BASE_BRANCH` must equal it.
4. Deliver the file through a pull request into the integration branch, and never push
   to protected branches.
5. Never add triggers, schedules, merge queues or server settings.
