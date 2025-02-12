# Test Solidus Extension GitHub action

A GitHub action for testing Solidus extensions

## Example configuration

```yaml
name: Test

on:
  push:
    branches:
      - main
  pull_request:
  schedule:
    - cron: "0 0 * * 4" # every Thursday

concurrency:
  group: test-${{ github.ref_name }}
  cancel-in-progress: ${{ github.ref_name != 'main' }}

permissions:
  contents: read

jobs:
  rspec:
    name: Solidus ${{ matrix.solidus-branch }}, Rails ${{ matrix.rails-version }} and Ruby ${{ matrix.ruby-version }} on ${{ matrix.database }}
    runs-on: ubuntu-24.04
    strategy:
      fail-fast: true
      matrix:
        rails-version:
          - "7.0"
          - "7.1"
          - "7.2"
        ruby-version:
          - "3.1"
          - "3.4"
        solidus-branch:
          - "v4.1"
          - "v4.2"
          - "v4.3"
          - "v4.4"
        database:
          - "postgresql"
          - "mysql"
          - "sqlite"
        exclude:
          - rails-version: "7.2"
            solidus-branch: "v4.3"
          - rails-version: "7.2"
            solidus-branch: "v4.2"
          - rails-version: "7.2"
            solidus-branch: "v4.1"
          - rails-version: "7.1"
            solidus-branch: "v4.2"
          - rails-version: "7.1"
            solidus-branch: "v4.1"
          - ruby-version: "3.4"
            rails-version: "7.0"
    env:
      TEST_RESULTS_PATH: coverage/coverage.xml
    steps:
      - uses: actions/checkout@v4
      - name: Run extension tests
        uses: solidusio/test-solidus-extension@main
        with:
          database: ${{ matrix.database }}
          rails-version: ${{ matrix.rails-version }}
          ruby-version: ${{ matrix.ruby-version }}
          solidus-branch: ${{ matrix.solidus-branch }}
      - name: Upload coverage reports to Codecov
        uses: codecov/codecov-action@v5
        continue-on-error: true
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ${{ env.TEST_RESULTS_PATH }}

```
