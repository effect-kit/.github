# EffectKit organization files and automation

This is the source for [effect-kit's GitHub profile](https://github.com/effect-kit).
Publish it as **effect-kit/.github**. The local folder is named `profile`.

- [Public profile](profile/README.md) and [avatar](profile/assets/logo.jpg)
- [Contribution guide](CONTRIBUTING.md), [support](SUPPORT.md), [security](SECURITY.md), and [conduct](CODE_OF_CONDUCT.md)
- [Issue forms](.github/ISSUE_TEMPLATE/) and [PR template](PULL_REQUEST_TEMPLATE.md)
- [Shared workflows](.github/workflows/), [actions](actions/), and [caller templates](workflow-templates/)

## Shared automation

Package CI, Changesets releases, PR title checks, Changeset status, and published-package
smoke checks are reusable across EffectKit repositories. Workspace setup installs pinned
Node/pnpm versions, optional Bun, and frozen dependencies.

Read the [workflow guide](docs/workflows.md) for inputs, caller examples, trusted publishing,
and the first `v1` release. The [package script contract](docs/package-scripts.md) defines
the commands consuming Turborepo repositories must provide.

The templates reference the published `@v1` workflows. Redis and emails use these callers;
new package repositories can copy them and implement the standard scripts. PR CI and release
checks stay fast; extended suites run only through manual dispatch.

This repository needs no dependency installation or build. The shared actions run directly
in GitHub Actions. Dependabot checks GitHub Actions dependencies weekly.

## Funding and license

[Funding](.github/FUNDING.yml) includes GitHub Sponsors for `envoy1084` and an
[ENS profile](https://app.ens.domains/envoy1084.eth) for direct ETH donations to `envoy1084.eth`.

Copyright 2026 Effect Kit Contributors. [Apache-2.0](LICENSE); see [NOTICE](NOTICE).

Only the selected header and logo are public. Drafts, logo options, prompts,
and previews stay in ignored `research/`.
