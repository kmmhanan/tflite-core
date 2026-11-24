# GitHub Actions guidelines

- When using a third-party action (one not hosted in a Google-managed org), a fixed version of the action MUST be used by [specifying a specific commit](https://help.github.com/en/github/automating-your-workflow-with-github-actions/workflow-syntax-for-github-actions#jobsjob_idstepsuses), rather than a branch like "main", or a tagged release, which can be overwritten by any maintainer of the action.
- Docker and [GitHub runner](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources) images should always be run at a fixed version rather than "latest".
- Exception: the reusable workflows and actions from our own [KMM-ZONE/open-flows](https://github.com/KMM-ZONE/open-flows) repository are referenced by their major version tag (`@v1`). We maintain them ourselves, minor and patch releases are backward compatible, and the publish workflow reports newer major versions in its job summary. Keep tag and branch protection enabled in that repository, because whoever can move its `v1` tag changes what runs here.
- If an action is granted additional GitHub access (by being passed the GITHUB_TOKEN variable), all main development branches (typically, "main") must use [branch protection](https://help.github.com/en/github/administering-a-repository/about-protected-branches) with "Require pull request reviews before merging" enabled and/or "Restrict who can push to matching branches" limited to repository admins.

## Publishing to pub.dev

The `publish.yml` workflow publishes `tflite_core` when a `v<version>` tag is pushed. It calls the reusable workflow `publish-flutter.yml` from [KMM-ZONE/open-flows](https://github.com/KMM-ZONE/open-flows) at `@v1`. That workflow checks that the tag matches the package version, runs formatting and analysis on `lib` and `test`, tests, and a pub.dev dry run, then publishes using GitHub Actions OIDC trusted publishing. No pub.dev token is stored in GitHub secrets.

Before using the workflow for releases:

1. Set up and verify the `kmmhanan.com` publisher on pub.dev.
2. Publish the first `tflite_core` version manually from a signed-in user account with `flutter pub publish`. Pub does not currently allow the first version of a new package to be published directly to a verified publisher.
3. Transfer the package from your user account to the `kmmhanan.com` publisher in the package's pub.dev admin page. This transfer is irreversible.
4. Enable GitHub Actions automated publishing from the package's **Admin** page. Use repository `kmmhanan/tflite-core` and tag pattern `v{{version}}`.
5. For later versions, update the package version and changelog, then push a matching version tag. For example, with `version: 0.12.6` in `pubspec.yaml`, push `v0.12.6`.

The GitHub repository and tag pattern are configured on pub.dev; no workflow filename, GitHub secret, pub.dev API key, or GitHub environment is required for this workflow. See the [pub.dev automated publishing guide](https://dart.dev/tools/pub/automated-publishing).

### Rehearsing a release

Run **Actions → Publish to pub.dev → Run workflow**. A manual run does everything except the upload (the `dry_run` switch is on by default).

### Creating a release

To create a GitHub Release and trigger the publish workflow manually, run **Actions → Create release → Run workflow** on the `main` branch. It reads the version from `pubspec.yaml` (open-flows `check-version` action), creates and pushes the corresponding annotated `v<version>` tag, then creates a GitHub Release with generated notes (open-flows `github-release` action; a pre-release version is marked as a pre-release). If the version is not newer than the latest tag, nothing is created and the run says why. The tag push triggers `publish.yml`.

This workflow uses a repository secret named `RELEASE_TOKEN` so pushing its tag triggers other workflows (GitHub suppresses those triggers when using the default `GITHUB_TOKEN`). Create a fine-grained personal access token scoped to this repository with **Contents: Read and write**, then add it under **Settings → Secrets and variables → Actions**. The token does not authenticate to pub.dev; pub.dev publishing still uses OIDC trusted publishing. Only trusted users should be able to run this release workflow.

### Updates from open-flows

`publish.yml` and `create-release.yml` reference open-flows as `@v1`, which always means the newest 1.x release, so minor and patch updates arrive without any change here.

When a new **major** version exists, the publish run writes a note to its job summary (no warning, no failure). Read the release notes, then change `@v1` to `@v2` in both workflows. Set `inform_new_version` in `publish.yml` to `none`, `major-only`, `major-minor-only` or `all` (the default) to choose what you are told.

If a release ever causes trouble, freeze the previous one by replacing `@v1` with an exact tag such as `@v1.2.0` in both workflows, then switch back once it is fixed.

### Differences from the previous in-repo publish workflow

- The old workflow skipped publishing when the tag's version was not newer than the latest tag. Now **Create release** refuses a version that is not newer, and the publish workflow skips a version that is already on pub.dev. A tag pushed by hand for an older, unpublished version would therefore be published.
- Analysis and formatting still cover only `lib` and `test` (`analyze_paths` and `format_paths`).
- Flutter is installed with the workflow's default cache key instead of the custom `cache-key` used before.
