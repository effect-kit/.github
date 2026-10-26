# EffectKit organization files

This is the source for [effect-kit's GitHub profile](https://github.com/effect-kit).
Publish it as **effect-kit/.github**. The local folder is named `profile`.

- [Public profile](profile/README.md) and [avatar](profile/assets/logo.jpg)
- [Contribution guide](CONTRIBUTING.md), [support](SUPPORT.md), [security](SECURITY.md), and [conduct](CODE_OF_CONDUCT.md)
- [Issue forms](.github/ISSUE_TEMPLATE/) and [PR template](PULL_REQUEST_TEMPLATE.md)

## Check

```sh
pnpm install --frozen-lockfile
pnpm check
pnpm lint:workflows
```

Requires Node 24.22+ and pnpm 12.14.0. The workflow linter uses actionlint 1.7.12.
On Linux x64 it can download a checksum-verified copy; other platforms need it installed.

## Funding and license

[Funding](.github/FUNDING.yml) points to `envoy1084`. The personal Sponsors
account still needs activation.

Copyright 2026 Effect Kit Contributors. [Apache-2.0](LICENSE); see [NOTICE](NOTICE).

Only the selected header and logo are public. Drafts, logo options, prompts,
and previews stay in ignored `research/`.
