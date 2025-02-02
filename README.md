# Shared GitHub Workflows

These are shared GitHub workflows used by my repositories.

## PyPI Trusted Publishers

As of this writing (in late 2024), the recommended best practice for publishing to PyPI is to use so-called [Trusted Publishers](https://docs.pypi.org/trusted-publishers/), also discussed in the [Python Packaging User Guide](https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/) and in this [postmortem of the Ultralytics supply chain attack](https://blog.pypi.org/posts/2024-12-11-ultralytics-attack-analysis/).  The Trusted Publishers mechanism allows the publishing process to use short-lived identity tokens, which are more secure than maintaining a long-lived PyPI API key in your GitHub repository secrets.

GitHub Actions is a supported publisher.  Unfortunately, as discussed on the [Troubleshooting](https://docs.pypi.org/trusted-publishers/troubleshooting/) page and detailed in [pypi/warehouse issue #11096](https://github.com/pypi/warehouse/issues/11096), the trusted publishing mechanism does not yet support reusable GitHub Actions.  Starting with `@v7` of the poetry-release workflow, I do rely on the official [PyPI Publish](https://github.com/pypa/gh-action-pypi-publish) shared action to publish artifacts, rather than `poetry publish`.  However, there's no way to support Trusted Publishing with full attestations for the time being.

## Poetry Version

The default Poetry version used in these workflows will change over time as new versions of Poetry are released. In general, the default value is set to to the latest version of Poetry that I have tested.  This way, I only need to maintain my preferred version in one place.  If you want to upgrade to a new version of Poetry on your own schedule, then set `poetry-version` explicitly in your workflow, rather than relying on the default.

If needed, I will release a new major version of the shared workflows whenever a new major version of Poetry is released.  For instance, Poetry v2 was released in January of 2025.  I adjusted the shared workflows to support the new project plugins feature supported in Poetry v2, and released v8 with the new interface.

## poetry-build-and-test

This workflow implements the standard build and test process for my Python libraries and utilities.

The following input parameters are accepted:

|Input|Type|Required|Default|Description|
|-----|----|--------|-------|-----------|
|`matrix-os-version`|String|Yes||JSON array as a string, a list of operating systems for the matrix build|
|`matrix-python-version`|String|Yes||JSON array as a string, a list of Python versions for the matrix build|
|`poetry-version`|String|No|_see below_|Version of Poetry to use for the build (>=2.0.0)|
|`poetry-cache-venv`|Boolean|No|`true`|Whether to cache the Poetry virtualenv|
|`poetry-cache-install`|Boolean|No|`true`|Whether to cache the Poetry install|
|`test-suite-command`|String|No|_see below_|Shell command used to execute the test suite|
|`timeout-minutes`|Number|No|`30`|Job timeout in minutes|
|`enable-coveralls`|Boolean|No|`false`|Whether to enable coverage reporting to coveralls.io|
|`persist-python-version`|String|No|_none_|Which matrix Python version to persist artifacts for, if any|

In order to run the release workflow (discussed below), you must set `persist-python-version` to persist build artifacts from at _exactly one_ matrix build.  Normally, I do this for the oldest supported Python version on the Linux platform, which is what the example below shows.

Starting with `v2` of the shared workflow, the default test suite command is:

```
./run suite
```

If you need a different command, and it's more complicated than a single line like this, you should extract a script to somewhere in the repository and invoke that script in the `test-suite-command`.  However, in any repo that follows the standard `run` script convention, it's best just to adjust the `suite` task to do what you need.

> _Note:_ The matrix versions are passed in as JSON strings because GitHub Actions does not support workflow inputs of type array.  See [this discussion](https://github.com/community/community/discussions/11692?sort=top#discussioncomment-3541856).

## poetry-release

This workflow implements the standard release process for my Python libraries and utilities.  It is designed to be triggered by pushing a new tag to GitHub, as shown in the example workflow below.  It uses the [GH Release](https://github.com/marketplace/actions/gh-release) action to create a new GitHub release, and can optionally publish to PyPI via the official [PyPI Publish](https://github.com/pypa/gh-action-pypi-publish) action.

The workflow relies on build artifacts published by the build process.  In your build workflow (discussed above), you must set `persist-python-version` to persist build artifacts from at _exactly one_ matrix build.  Normally, I do this for the oldest supported Python version on the Linux platform, which is what the example below shows.

The following input parameters are accepted:

|Input|Type|Required|Default|Description|
|-----|----|--------|-------|-----------|
|`timeout-minutes`|Number|No|`30`|Job timeout in minutes|
|`publish-pypi`|Boolean|No|`false`|Whether to publish artifacts to PyPI|

## Example Workflow

```yaml
# On GHA, the Linux runners are *much* faster and more reliable, so we only run the full matrix build there
name: Test Suite
on:
  push:
    branches:
      - main
    tags:
      - "v*"
  pull_request:
    branches:
      - main
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  linux-build-and-test:
    name: "Linux"
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v8
    secrets: inherit
    with:
      matrix-os-version: "[ 'ubuntu-latest' ]"
      matrix-python-version: "[ '3.10', '3.11', '3.12', '3.13' ]"  # run Linux tests on all supported Python versions
      enable-coveralls: true  # only report to coveralls.io for tests that run on Linux
      persist-python-version: "3.10"  # persist artifacts for the oldest supported Python version
  macos-build-and-test:
    name: "MacOS"
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v8
    secrets: inherit
    with:
      matrix-os-version: "[ 'macos-latest' ]"
      matrix-python-version: "[ '3.13' ]"  # only run MacOS tests on latest Python
  windows-build-and-test:
    name: "Windows"
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v8
    secrets: inherit
    with:
      matrix-os-version: "[ 'windows-latest' ]"
      matrix-python-version: "[ '3.13' ]"  # only run Windows tests on latest Python
  release:
    name: "Release"
    if: github.ref_type == 'tag'
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-release.yml@v8
    needs: [ linux-build-and-test, macos-build-and-test, windows-build-and-test ]
    secrets: inherit
    with:
      publish-pypi: true
```
