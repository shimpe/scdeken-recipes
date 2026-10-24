# sc-deken recipes

This repository is the package list behind [sc-deken](https://github.com/sc-deken/sc-deken),
a package manager for SuperCollider plugins and quarks. Maintainers review recipes
and advisories here; CI builds and signs the binaries and publishes the signed
index at <https://sc-deken.github.io/sc-deken-recipes/index.json>.

## Add a package

The easy way: open an issue with the **Add a package** form and paste the GitHub
address of the plugin or quark. A bot drafts the recipe and opens a pull request;
the build results for every platform appear as comments on that pull request.

The manual way:

    scdeken recipe new https://github.com/owner/name --out recipes/name.toml
    scdeken recipe check recipes/name.toml --build

then open a pull request with the new file. A maintainer reads the recipe, the
build flags and the code at the pinned commit before approving.

## How versions get approved

A recipe lists the versions a maintainer has reviewed, each pinned to a commit:

    [versions.approved]
    "0.1.0" = "7d99fe3301712ebd666eab659197317f2a73a8bb"

Nothing is offered as a reviewed download until its version is in that table.
Every night the **Index** workflow looks for new upstream tags and opens one pull
request per new tag (`bot/<package>-<version>`) with a link to the code
changes. Merging it approves the version. If an upstream tag later points at a
different commit, the bot opens a `Moved tag: <package> <version>` issue.

When approved versions reach `main`, the **Publish** workflow builds them on
Linux (x86_64, arm64), macOS (universal) and Windows (x86_64), creates a release
named `<package>-<version>` with the zips, file lists and `SHA256SUMS`, and
refreshes the index.

## Advisories

Advisories in `advisories/` retract malicious, vulnerable or broken versions.
See [advisories/README.md](advisories/README.md). Changes there are reviewed by
the code owner and published by the Index workflow as soon as they reach `main`.

## Setting up the repository (maintainers)

1. **Signing key.** Generate it once, somewhere safe outside any repository:

       scdeken index keygen --out-dir <secure folder>

   This writes `scdeken.pub` and the password-protected `scdeken.key`. Never
   commit the key.
2. **Secrets.** Create a GitHub *environment* named `release`, restrict it to the
   `main` branch, and add two secrets to it: `SCDEKEN_SIGNING_KEY` (the text of
   `scdeken.key`) and `SCDEKEN_SIGNING_PASSWORD`. Only the Index job and the
   release step of Publish use this environment; pull request builds never see it.
3. **Public key.** Paste the contents of `scdeken.pub` into `EmbeddedPublicKey`
   in `internal/sign/key.go` of the sc-deken repository and ship a new release.
4. **Pages.** In Settings > Pages choose *GitHub Actions* as the source.
5. **Actions settings.** Allow GitHub Actions to create pull requests
   (Settings > Actions > General).
6. **Optional `BOT_TOKEN`.** Pull requests created with the default token do not
   start the PR build workflow. Add a repository secret `BOT_TOKEN` (a
   fine-grained token with contents, pull requests and issues write access) and
   the bot uses it instead.
7. **Tool version.** `.scdeken-version` holds the sc-deken tag or branch the
   workflows install with `go install`. It starts as `main`; after the first
   sc-deken release, pin it to that tag (for example `v0.1.0`).

## License

MIT, see [LICENSE](LICENSE).
