# Shared GitHub Workflows

These are shared GitHub workflows used by my repositories.

## poetry-build-and-test

Implements the standard build and test process for my Python libraries and utilities.

Example workflow:

```yaml
name: Test Suite
on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - master
  schedule:
    - cron: '05 17 15 * *'  # 15th of the month at 5:05pm UTC
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  build-and-test:
    uses: pronovic/gha-shared-workflows/.github/workflows/poetry-build-and-test.yml@v1
    secrets: inherit
    with:
      matrix-os-version: "[ 'ubuntu-latest', 'macos-latest', 'windows-latest' ]"
      matrix-python-version: "[ '3.7', '3.8', '3.9', '3.10' ]"
      poetry-version: "1.2.0"
      poetry-plugins: "poetry-plugin-export@==1.1.2"
```

The following input parameters are accepted:

|Input|Type|Required|Description|
|-----|----|--------|-----------|
|`matrix-os-version`|String|Yes|JSON array as a string, a list of operating systems for the matrix build|
|`matrix-python-version`|String|Yes|JSON array as a string, a list of Python versions for the matrix build|
|`poetry-version`|String|Yes|Version of Poetry to use for the build (>=1.2.0)|
|`poetry-plugins`|String|No|Comma-separated list of Poetry plugins to install, defaults to none|
|`poetry-cache-venv`|Boolean|No|Whether to cache the Poetry virtualenv; defaults to `true`|
|`poetry-cache-install`|Boolean|No|Whether to cache the Poetry install; defaults to `true`|
|`timeout-minutes`|Number|No|Job timeout in minutes|
|`test-suite-command`|String|No|Shell command used to execute the test suite|

The default test suite command is:

```
poetry run tox -c .toxrc -e 'checks,coverage'
```

If you need a different command and it's more complicate than a single line like this, it's best to extract a script to somewhere in the repository and invoke that in the `test-suite-command`.

The matrix versions are passed in as JSON strings because GitHub Actions does not support workflow inputs of type array.  As of this writing (October of 2022), passing in JSON like this is the [most highly rated solution solution](https://github.com/community/community/discussions/11692?sort=top#discussioncomment-3541856).
