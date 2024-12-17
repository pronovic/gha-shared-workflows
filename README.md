# Shared GitHub Workflows

These are shared GitHub workflows used by my repositories.

## PyPI Trusted Publishers

As of this writing (in late 2024), the recommended best practice for publishing to PyPI is to use so-called [Trusted Publishers](https://docs.pypi.org/trusted-publishers/).  This allows the publishing process to use short-lived identity tokens, which are more secure than maintaining a long-lived PyPI API key in your GitHub repository secrets.  GitHub Actions is a supported publisher.  Unfortunately, as discussed on the [Troubleshooting](https://docs.pypi.org/trusted-publishers/troubleshooting/) page and detailed in [pypi/warehouse issue #11096](https://github.com/pypi/warehouse/issues/11096), the trusted publishing mechanism does not yet support reusable GitHub Actions.  Starting with `@v7` of the poetry-release workflow, I do rely on the official [PyPI Publish](https://github.com/pypa/gh-action-pypi-publish) shared action to publish artifacts, rather than `poetry publish`.  However, there's no way to support Trusted Publishing with full attestations for the time being.

## poetry-build-and-test

This workflow implements the standard build and test process for my Python libraries and utilities.

The following input parameters are accepted:

|Input|Type|Required|Default|Description|
|-----|----|--------|-------|-----------|
|`matrix-os-version`|String|Yes||JSON array as a string, a list of operating systems for the matrix build|
|`matrix-python-version`|String|Yes||JSON array as a string, a list of Python versions for the matrix build|
|`poetry-version`|String|No|_see below_|Version of Poetry to use for the build (>=1.8.0)|
|`plugin-version`|String|No|_see below_|Version of the [poetry-dynamic-versioning](https://github.com/mtkennerly/poetry-dynamic-versioning) plugin to use for the build (>=1.2.0)|
|`poetry-plugins`|String|No|_none_|Comma-separated list of Poetry plugins to install (in addition to [poetry-dynamic-versioning](https://github.com/mtkennerly/poetry-dynamic-versioning))|
|`poetry-cache-venv`|Boolean|No|`true`|Whether to cache the Poetry virtualenv|
|`poetry-cache-install`|Boolean|No|`true`|Whether to cache the Poetry install|
|`test-suite-command`|String|No|_see below_|Shell command used to execute the test suite|
|`timeout-minutes`|Number|No|`30`|Job timeout in minutes|
|`enable-coveralls`|Boolean|No|`false`|Whether to enable coverage reporting to coveralls.io|
|`persist-python-version`|String|No|_none_|Which matrix Python version to persist artifacts for, if any|

In order to run the release workflow (discussed below), you must use `persist-python-version` to persist build artifacts from at exactly one matrix build.  Normally, I do this for the oldest supported Python version on the Linux platform, which is what the example below shows.

The default values for `poetry-version` and `plugin-version` will change as new versions are released. In general, these values are set to to the latest version of each tool that I have tested.  This way, I only need maintain my preferred version in one place.  If you want to upgrade to new versions of these tools on your own schedule, then set these values explicitly in your workflow rather than relying on the defaults.

Starting with `v2` of the shared workflow, the default test suite command is:

```
./run suite
```

If you need a different command, and it's more complicated than a single line like this, you should extract a script to somewhere in the repository and invoke that script in the `test-suite-command`.  However, in any repo that follows the standard `run` script convention, it's best just to adjust the `suite` task to do what you need.

> _Note:_ The matrix versions are passed in as JSON strings because GitHub Actions does not support workflow inputs of type array.  As of this writing (October of 2022), passing in JSON like this is the [most highly rated solution](https://github.com/community/community/discussions/11692?sort=top#discussioncomment-3541856).

## poetry-release

This workflow implements the standard release process for my Python libraries and utilities.  It is designed to be triggered by pushing a new tag to GitHub, as shown in the example workflow below.  It uses the [GH Release](https://github.com/marketplace/actions/gh-release) action to create a new GitHub release, and can optionally publish to PyPI via the official [PyPI Publish](https://github.com/pypa/gh-action-pypi-publish) action.

The workflow relies on build artifacts published by the build process.  In your build workflow (discussed above), you must use `persist-python-version` to persist build artifacts from at exactly one matrix build.  Normally, I do this for the oldest supported Python version on the Linux platform, which is what the example below shows.

The following input parameters are accepted:

|Input|Type|Required|Default|Description|
|-----|----|--------|-------|-----------|
|`os-version`|String|Yes||Operating system to use for the build|
|`timeout-minutes`|Number|No|`30`|Job timeout in minutes|
|`publish-pypi`|Boolean|No|`false`|Whether to publish artifacts to PyPI|

## Example Workflow

```yaml
# On GHA, the Linux runners are *much* faster and more reliable, so we only run the full matrix build there
name: Test Suite
on:
  push:
    branches:
      - master
    tags:
      - "v*"
  pull_request:
    branches:
      - master
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  linux-build-and-test:
    name: "Linux"
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v7
    secrets: inherit
    with:
      matrix-os-version: "[ 'ubuntu-latest' ]"
      matrix-python-version: "[ '3.10', '3.11', '3.12', '3.13' ]"  # run Linux tests on all supported Python versions
      enable-coveralls: true  # only report to coveralls.io for tests that run on Linux
      persist-python-version: "3.10"  # persist artifacts for the oldest supported Python version
  macos-build-and-test:
    name: "MacOS"
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v7
    secrets: inherit
    with:
      matrix-os-version: "[ 'macos-latest' ]"
      matrix-python-version: "[ '3.13' ]"  # only run MacOS tests on latest Python
  windows-build-and-test:
    name: "Windows"
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v7
    secrets: inherit
    with:
      matrix-os-version: "[ 'windows-latest' ]"
      matrix-python-version: "[ '3.13' ]"  # only run Windows tests on latest Python
  release:
    name: "Release"
    if: github.ref_type == 'tag'
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-release.yml@v7
    needs: [ linux-build-and-test, macos-build-and-test, windows-build-and-test ]
    secrets: inherit
    with:
      publish-pypi: true
```
