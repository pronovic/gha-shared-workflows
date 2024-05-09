# Shared GitHub Workflows

These are shared GitHub workflows used by my repositories.

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
|`timeout-minutes`|Number|No|`30`|Job timeout in minutes|
|`enable-coveralls`|Boolean|No|`true`|Whether to enable coverage reporting to coveralls.io|
|`test-suite-command`|String|No|_see below_|Shell command used to execute the test suite|

The default values for `poetry-version` and `plugin-version` will change as new versions are released. In general, these values are set to to the latest version of each tool that I have tested.  This way, I only need maintain my preferred version in one place.  If you want to upgrade to new versions of these tools on your own schedule, then set these values explicitly in your workflow rather than relying on the defaults.

The default test suite command for `v2` of the shared workflow is:

```
./run suite
```

If you need a different command, and it's more complicated than a single line like this, you should extract a script to somewhere in the repository and invoke that script in the `test-suite-command`.  However, in any repo that follows the standard `run` script convention, it's best just to adjust the `suite` task to do what you need.

The matrix versions are passed in as JSON strings because GitHub Actions does not support workflow inputs of type array.  As of this writing (October of 2022), passing in JSON like this is the [most highly rated solution solution](https://github.com/community/community/discussions/11692?sort=top#discussioncomment-3541856).

## poetry-release

This workflow implements the standard release process for my Python libraries and utilities.  It is designed to be triggered by pushing a new tag to GitHub, as shown in the example workflow below.  It uses the [GH Release](https://github.com/marketplace/actions/gh-release) action to create a new GitHub release, and can optionally publish to PyPI via `poetry publish`.

The following input parameters are accepted:

|Input|Type|Required|Default|Description|
|-----|----|--------|-------|-----------|
|`os-version`|String|Yes||Operating system to use for the build|
|`python-version`|String|Yes||Version of Python to use for the build|
|`poetry-version`|String|No|_see below_|Version of Poetry to use for the build (>=1.8.0)|
|`plugin-version`|String|No|_see below_|Version of the [poetry-dynamic-versioning](https://github.com/mtkennerly/poetry-dynamic-versioning) plugin to use for the build (>=1.2.0)|
|`poetry-plugins`|String|No|_none_|Comma-separated list of Poetry plugins to install (in addition to [poetry-dynamic-versioning](https://github.com/mtkennerly/poetry-dynamic-versioning))|
|`poetry-cache-venv`|Boolean|No|`true`|Whether to cache the Poetry virtualenv|
|`poetry-cache-install`|Boolean|No|`true`|Whether to cache the Poetry install|
|`timeout-minutes`|Number|No|`30`|Job timeout in minutes|
|`publish-pypi`|Boolean|No|`false`|Whether to publish artifacts to PyPI|

The default values for `poetry-version` and `plugin-version` will change as new versions are released. In general, these values are set to to the latest version of each tool that I have tested.  This way, I only need maintain my preferred version in one place.  If you want to upgrade to new versions of these tools on your own schedule, then set these values explicitly in your workflow rather than relying on the defaults.

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
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v6
    secrets: inherit
    with:
      matrix-os-version: "[ 'ubuntu-latest' ]"
      matrix-python-version: "[ '3.9', '3.10', '3.11', '3.12' ]"  # run Linux tests on all supported Python versions
      enable-coveralls: true  # only report to coveralls.io for tests that run on Linux
  macos-build-and-test:
    name: "MacOS"
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v6
    secrets: inherit
    with:
      matrix-os-version: "[ 'macos-latest' ]"
      matrix-python-version: "[ '3.12' ]"  # only run MacOS tests on latest Python
      enable-coveralls: false
  windows-build-and-test:
    name: "Windows"
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v6
    secrets: inherit
    with:
      matrix-os-version: "[ 'windows-latest' ]"
      matrix-python-version: "[ '3.12' ]"  # only run Windows tests on latest Python
      enable-coveralls: false
  release:
    name: "Release"
    if: github.ref_type == 'tag'
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-release.yml@v6
    needs: [ linux-build-and-test, macos-build-and-test, windows-build-and-test ]
    secrets: inherit
    with:
      python-version: "3.9"  # run release with oldest supported Python version
      publish-pypi: true
```
