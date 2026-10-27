# Shared GitHub automation

This repository is published as `effect-kit/.github`. The extra `.github` in a caller such as
`effect-kit/.github/.github/workflows/ci.yml@v1` is intentional: repository name, then directory.
All reusable workflows run on GitHub-hosted Ubuntu. They require explicit caller files.

## Adoption and versioning

1. Review the shared workflows and actions.
2. Publish a reviewed commit and a `v1` tag before using the supplied templates. The tag and
   remote files do not exist merely because templates exist in a local checkout.
3. Give each consuming repository the [standard scripts](package-scripts.md), `.node-version`,
   a pinned `packageManager` field, and `pnpm-lock.yaml`. Add `.bun-version` when enabling Bun.
4. Copy the desired YAML files from `workflow-templates/` to the package's `.github/workflows/`.
   GitHub's New workflow UI can also instantiate them from this public repository.
5. Configure npm trusted publishing before adopting the release caller.

Templates use the floating `v1` tag. Release compatible updates there and reserve breaking
interfaces for a new major. Shared workflows reference this repository's actions at `@v1`;
keep action interfaces compatible throughout that major. Pin a reviewed SHA when a consumer
needs a frozen workflow; its internal `@v1` action references must also be pinned for a fully
frozen dependency graph. Test candidate branches with a consumer before moving a stable tag.

`$default-branch` in the CI template is replaced by GitHub when creating a workflow through
the UI. Replace it yourself when copying the file manually. Other callers derive the default
branch from the event metadata.

## Workspace setup action

```yaml
- uses: actions/checkout@v7
  with:
    persist-credentials: false
- uses: effect-kit/.github/actions/setup@v1
  with:
    cache: pnpm
    bun: "false"
```

The action operates in the checked-out workspace root. pnpm comes from `packageManager`;
Node comes from `.node-version`. It installs with `--frozen-lockfile`.

| Input            | Default   | Meaning                                                      |
| ---------------- | --------- | ------------------------------------------------------------ |
| `node-version`   | Empty     | Override `.node-version`, for example `24.x`.                |
| `bun`            | `"false"` | Install the version in `.bun-version` when `"true"`.         |
| `cache`          | Empty     | `pnpm` enables the pnpm store cache; empty disables caching. |
| `ignore-scripts` | `"false"` | Disable install lifecycle scripts when `"true"`.             |

Checkout happens in the caller's repository even inside a reusable workflow. Shared actions
therefore use an explicit remote reference; `./actions/setup` would resolve in the consumer.

## Package CI

`ci.yml` checks out the caller, sets up the workspace, and runs `pnpm ci:check`. It grants only
`contents: read`. Optional reports upload even when checks fail, with unique artifact names
for concurrent matrix jobs. Reports are retained for 14 days.

Inputs: `node-version` (empty), `bun` (false), `timeout-minutes` (30), and `report-paths`
(empty; newline-separated paths). Runtime matrices belong in callers:

```yaml
jobs:
  check:
    strategy:
      fail-fast: false
      matrix:
        node: ["24.22.0", "24.x"]
    uses: effect-kit/.github/.github/workflows/ci.yml@v1
    with:
      node-version: ${{ matrix.node }}
      bun: true
```

The CI template includes PRs, default-branch pushes, merge queues, and manual dispatch.
CI never publishes. Keep repository-specific compatibility and service tests in separate
jobs using the setup action. Choose required status checks after inspecting the migrated
job names. GitHub-token-created version PRs do not automatically trigger ordinary CI;
the manual CI entry point can check their branch. Release checks the merged commit with
the repository's `release:prepare` and packed-archive verification commands.

## Releases

`release.yml` requires `workflow_dispatch` from the caller's default branch. It serializes
releases per repository without cancelling publication. Do not reuse its concurrency group
in the caller. Inputs are `bun` (false, verification only), `timeout-minutes` (30, verification),
and `report-paths` (empty, additional newline-separated evidence paths).

The workflow uses Changesets v2 actions with Changesets CLI v3:

1. **Plan:** choose `version`, `publish`, or no action.
2. **Version:** run `pnpm release:version`, then open or update the version/changelog PR.
3. **Prepare:** run `pnpm release:prepare`, pack the selected release plan, download its artifact,
   run `release:check`, require its verification report, and check the original file checksums.
4. **Publish:** publish that same immutable packed artifact with OIDC and provenance,
   then create Git tags and GitHub releases.

Each package defines `release:prepare` to choose its release checks and build steps.
It may reuse `ci:check` or run a different set. `release:check --pack-dir <directory>`
always runs after packing to verify the exact archives that will be published.

The caller authorizes `contents: write`, `pull-requests: write`, and `id-token: write` on
the calling job. The shared workflow scopes those permissions to the jobs that need them:
only publication gets OIDC. No npm token or blanket `secrets: inherit` is required.
Versioning and publication install dependencies with lifecycle scripts disabled and caching off.

Merge the version PR after review, then dispatch Release again. Publication never waits
for registry availability. Failed runs keep available verification evidence. Packed archives
and verification reports are retained for 30 days; download them earlier if needed longer.
The prepare job summary records the run and packed artifact IDs.

Outputs: `published` (`"true"`/`"false"` when publication runs), `published-packages`
(a JSON array), and `pack-artifact-id`. Outputs from skipped jobs are empty.

### npm setup

For each npm package, configure a GitHub Actions trusted publisher with organization
`effect-kit`, its own repository (for example `emails`), and workflow filename `release.yml`.
Leave the environment empty: these workflows do not select a GitHub environment.
For a new package, complete its initial npm publication and trusted-publisher setup first.
Ensure package repository metadata points to the caller repository.

Allow GitHub Actions to create pull requests in the consuming repository's Actions settings.
Keep the workflow filename `release.yml`: npm checks the calling workflow identity, and
the smoke workflow validates this path. Organization Actions policies must permit the
shared repository and its third-party action dependencies.

## Published-package smoke checks

After npm makes a version available, manually dispatch the `release-smoke.yml` template
with `release-run-id` and `pack-artifact-id` from the release summary. Optional inputs are
`bun` (false) and `timeout-minutes` (20). Caller permissions are `contents: read` and
`actions: read`; there is no publication permission.

The workflow requires a successful manual release from this repository's default branch,
the `release.yml` caller, and an unexpired `changeset-pack-*` artifact from that run.
It checks out the release's original SHA, downloads that artifact, and runs:

```sh
pnpm release:smoke --pack-dir "$RUNNER_TEMP/release-candidate" --provenance
```

It does not rebuild or republish archives. Retry after registry propagation if necessary.
For partial publication failures, inspect the saved artifacts and existing npm versions
before recovery; the shared smoke workflow accepts only successful release runs.

## PR metadata

`pr-title.yml` uses [amannn/action-semantic-pull-request](https://github.com/amannn/action-semantic-pull-request)
to check Conventional Commit PR titles, including scopes and `!` for breaking changes.
Example: `feat(redis): add a command`. It uses the action's default rules; it does not
apply a separate 100-character limit. No PR code is checked out or installed.
Use the same title rules in repository commit tooling when squash merging.

The title caller uses `pull_request_target` so the check works for forked PRs and the
workflow comes from the base branch. Both caller and shared workflow grant only
`pull-requests: read`. Keep this metadata check separate from workflows that run PR code.

`changeset-status.yml` uses the Changesets PR-status action and writes its result to the
job summary. It is informational: documentation-only PRs need not have a Changeset.
It supports normal `pull_request` events, including forks, without comments or write tokens.
It checks out the base commit and retains read-only checkout credentials so the action can
fetch PR refs. It does not install dependencies or invoke submitted package scripts.

Both callers should include the `edited` PR activity so metadata changes refresh the result.

## Maintaining this repository

Edit workflows, actions, and templates directly; this repository has no local package tooling,
test suite, or build step. Dependabot checks GitHub Actions dependencies weekly.

References: [GitHub workflow reuse](https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows),
[Changesets actions](https://github.com/changesets/action), and
[npm trusted publishing](https://docs.npmjs.com/trusted-publishers/).
