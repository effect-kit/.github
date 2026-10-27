# EffectKit package script contract

Shared workflows call root scripts. A repository owns its Turborepo task graph and test
implementation; shared YAML does not embed package paths or accept arbitrary shell commands.

## Required root scripts

| Script            | Contract                                                                      |
| ----------------- | ----------------------------------------------------------------------------- |
| `format`          | Apply formatting.                                                             |
| `format:check`    | Check formatting without changing tracked files.                              |
| `lint`            | Lint packages, tests, examples, and root tooling.                             |
| `lint:fix`        | Apply supported lint fixes.                                                   |
| `typecheck`       | Check source, tests, examples, and configuration.                             |
| `test`            | Run deterministic tests without credentials or external services.             |
| `build`           | Build all workspace packages.                                                 |
| `test:artifacts`  | Check package contents, exports, declarations, and isolated consumers.        |
| `check`           | Run formatting, lint, types, tests, and build without changing tracked files. |
| `ci:check`        | Run fast formatting, lint, types, offline tests, and build for PRs.            |
| `changeset`       | Record a release note and intended version bump.                              |
| `release:version` | Update versions, changelogs, lockfile, and formatting.                        |
| `release:prepare` | Run release-specific source checks and build the packages before packing.    |
| `release:pack`    | Create a local Changesets packed release directory.                           |
| `release:check`   | Verify the supplied packed release plan and archives.                         |
| `release:smoke`   | Compare published packages and provenance with supplied release archives.     |

A typical composition is:

```json
{
  "check": "pnpm format:check && pnpm lint && pnpm typecheck && pnpm test && pnpm build",
  "ci:check": "pnpm check",
  "release:prepare": "pnpm check"
}
```

Keep repeated artifact builds, full consumer/compiler matrices, fuzzing, and live suites
in manually dispatched workflows. `typecheck` must
include test types, even if it delegates to an internal `typecheck:test` script.
Local controlled fixtures are allowed in offline tests; Docker, live network services,
credentials, and publication belong outside this gate. Installation may need registry access.

## Release commands

Use Changesets CLI v3 with public access and the repository's default branch in
`.changeset/config.json`. Keep its `commit` option false; the version action owns the PR.
Release versioning must not rely on install lifecycle scripts or Bun.

`release:prepare` owns checks and builds before packing. It can run the same checks as CI
or a different release-specific set; it must build everything needed for the release.
The shared release workflow runs this command instead of `ci:check`.
`release:check` verifies the packed candidate afterward and is always required.

```sh
pnpm release:check --pack-dir /absolute/path/to/release-candidate
pnpm release:smoke --pack-dir /absolute/path/to/release-candidate --provenance
```

Both commands must accept arguments appended after the root script name. They must handle
the full release plan, including multiple publishable packages, rather than assume one
hard-coded tarball. `release:check` exits nonzero on failure and writes
`<pack-dir>/verification.json`. It must leave the input plan and archives unchanged.
`release:smoke` checks already-published packages; it must never publish or rebuild them.

`release:pack` is for local checks. Shared releases use `changesets/action/pack` to pack
the selected publish plan and retain its immutable artifact for verification and publication.

## Workspace tasks and optional suites

Publishable workspace packages normally expose `build`, `lint`, `typecheck`, `test`,
`check:package`, `test:artifacts`, `test:release`, and `test:release:registry`.
Root scripts aggregate applicable package tasks. Private packages need only applicable tasks.
Define build dependencies and output directories in `turbo.json`; include generated test
reports in outputs when cached tasks are expected to restore them.

Add `dev`, `test:watch`, `test:unit`, `test:integration`, `test:compat`, `test:live`,
`test:providers`, `test:fuzz`, `test:hardening`, `fixtures:check`, `fixtures:pull`, and
`clean` only when useful. Disable Turbo caching for publication and live-service checks.

## Migrating Redis and emails

- Use `ci:check` for fast checks. Keep protocol fuzzing, repeated builds, and full consumer
  matrices in manually dispatched extended-test workflows.
- Fold test type checking into the public `typecheck` command.
- Add `release:version` as an adapter to `version-packages` initially; preserve old commands
  until documentation and consumers have moved.
- Use `release:prepare` for fast release checks and the build. Keep archive validation in
  `release:check`; avoid full provider matrices and live services during publication.
- Make Redis's shared `release:check` perform its current offline `--skip-live` behavior
  internally. Keep that package-specific flag out of shared YAML.
- Retain specialized Redis live/provider and emails compiler-compatibility workflows.
- Configure optional Bun and report paths in callers, then replace duplicated setup steps.

This contract applies to consuming package repositories. The profile repository contains
shared workflow definitions and actions, with no package scripts or build step of its own.
