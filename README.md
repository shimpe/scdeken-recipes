# sc-deken recipes

This repository is the package list behind [sc-deken](https://github.com/shimpe/scdeken),
a package manager for SuperCollider plugins and quarks. Maintainers review recipes
and advisories here; CI builds and signs the binaries and publishes the signed
index at <https://shimpe.github.io/scdeken-recipes/index.json>.

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
Keep the `advisories/` folder even when it holds no advisory (its README keeps it
in git): the index build stops with an error when the folder is missing, so a
wrong path can never publish a package list without its advisories.

## Setting up the repository (maintainers)

1. **Signing key.** Generate it once, somewhere safe outside any repository:

       scdeken index keygen --out-dir <secure folder>

   This asks for the password twice and writes `scdeken.pub` and the
   password-protected `scdeken.key`. Never commit the key. On Windows, quote a
   folder name with spaces and leave out the trailing backslash:
   `--out-dir 'C:\Users\me\signing keys'` (see the maintainer guide).
2. **Secrets.** First create a GitHub *environment* named `release` and restrict
   it to the `main` branch (Settings > Environments > Deployment branches);
   only **then** add its two secrets: `SCDEKEN_SIGNING_KEY` (the text of
   `scdeken.key`) and `SCDEKEN_SIGNING_PASSWORD`. Secrets added before the
   restriction are open to every branch for that moment. Only the Index
   workflow's sign job and the release step of Publish use this environment;
   pull request builds never see it. Every click is listed in the maintainer
   guide: [Storing the key in GitHub, step by step](https://github.com/shimpe/scdeken/blob/main/docs/maintainer-guide.md#storing-the-key-in-github-step-by-step).
3. **Public key.** Paste the contents of `scdeken.pub` into `EmbeddedPublicKey`
   in `internal/sign/key.go` of the sc-deken repository and ship a new release.
4. **Pages.** In Settings > Pages choose *GitHub Actions* as the source.
5. **Actions settings.** Allow GitHub Actions to create pull requests:
   Settings > Actions > General, scroll to **Workflow permissions**, tick
   **Allow GitHub Actions to create and approve pull requests**, and click
   **Save**. Leave the permissions radio button on read-only; the workflows ask
   for what they need per job. Do the same in the sc-deken tool repository,
   whose release workflow opens a pull request too (details in the maintainer
   guide: [Letting workflows open pull requests](https://github.com/shimpe/scdeken/blob/main/docs/maintainer-guide.md#letting-workflows-open-pull-requests)).
   Then create the label **`add-package`**
   (Issues > Labels > New label). The "Add a package" form, which the
   **Suggest a package** button in the SuperCollider window opens, tags new
   issues with it, and the add-package workflow only runs for issues that carry
   it. GitHub silently drops a form's label when the label does not exist.
6. **Branch protection (required).** "Merging a pull request means the version is
   approved" is only true if `main` is protected. Create a ruleset for `main` that:
   - requires a pull request before merging;
   - requires review from Code Owners (`.github/CODEOWNERS`) — see the note below;
   - requires the status check **`pr-build-result`** (the only check with a fixed
     name: it fails when a recipe check or the advisory validation fails; a
     platform that fails to build is reported but does not block, because
     Publish leaves that platform out and releases the others);
   - has **no bypass** for GitHub Actions, the bot account or admins you do not
     want to trust blindly;
   - blocks force pushes and branch deletion.
   Without this, anyone who can push can approve a version, and the publish and
   index workflows will build and sign it.

   *With a single maintainer*, "require review from Code Owners" blocks the
   owner's own pull requests, because nobody else can approve them. Choose one,
   knowingly: allow only the owner to bypass the review rule, or require
   0 approvals until a second maintainer joins (the pull request and the
   `pr-build-result` check are still required). Turn the review back on as soon
   as there are two maintainers.
7. **Optional `BOT_TOKEN`.** Pull requests created with the default token do not
   start the PR build workflow. If you want them to, create a *fine-grained*
   personal access token owned by a **separate bot account**, limited to **this
   repository only**, with contents, pull requests and issues write access, and
   store it as the repository secret `BOT_TOKEN`. Be aware of the consequence:
   with `BOT_TOKEN` set, anyone who opens an "Add a package" issue makes CI
   build the repository they named (the build runs with no secrets, but it does
   run their code on GitHub's runners).
8. **Tool version.** `.scdeken-version` holds the sc-deken version the workflows
   install with `go install`, including the job that signs the index. Keep it
   pinned to a reviewed tag (for example `v0.1.0`) or a full 40-character commit
   id of the sc-deken repository, never a branch, and change it only through a
   reviewed pull request. (It says `main` only until sc-deken is first published.)

## Rebuilding a version

Publish builds each platform separately; a platform that fails is left out of
the release and the others are published. To try a version again (for example
after fixing a runner problem), run the **Publish** workflow by hand (Actions >
Publish > Run workflow) with the recipe file (`recipes/<name>.toml`) and the
approved version. It adds the platforms that now build to the existing release.

## A new SuperCollider release

`sc-versions.toml` says which SuperCollider release has which plugin API (the
`api3` in download names such as `windows-x86_64-api3.zip`). The window uses it
to pick downloads that load in the user's SuperCollider; a release newer than
the table gets local builds instead. CI builds every API in the table once per
platform, against the newest release listed for it.

When SuperCollider releases a new version, check its `sc_api_version` (in
`include/plugin_interface/SC_InterfaceTable.h`). Same API: raise `version` in
that row. New API: add a row with `min` set to that release, then rebuild the
versions that should get downloads for it as described above. Step by step:
[maintainer guide, SuperCollider versions and the plugin API](https://github.com/shimpe/scdeken/blob/main/docs/maintainer-guide.md#supercollider-versions-and-the-plugin-api).

## License

MIT, see [LICENSE](LICENSE).
