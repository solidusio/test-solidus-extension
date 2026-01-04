# Test Solidus Extension GitHub action

GitHub actions and workflows for testing Solidus extensions.

## Usage

### Reusable Workflow (Recommended)

The simplest way to test your extension. The reusable workflow handles database services and provides a sensible default test matrix:

```yaml
name: Test

on:
  push:
    branches: [main]
  pull_request:

jobs:
  Test:
    uses: solidusio/test-solidus-extension/.github/workflows/test.yml@main
```

#### Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `rails_versions` | Rails versions (JSON array) | `["8.0", "7.2"]` |
| `ruby_versions` | Ruby versions (JSON array) | `["3.4"]` |
| `solidus_branches` | Solidus branches (JSON array) | `["v4.6", "v4.5"]` |
| `databases` | Databases (JSON array) | `["postgresql", "mysql", "sqlite"]` |

#### Custom Matrix

Override any input to customize the test matrix:

```yaml
jobs:
  test:
    uses: solidusio/test-solidus-extension/.github/workflows/test.yml@main
    with:
      rails_versions: '["7.2"]'
      databases: '["postgresql", "sqlite"]'
```

### Composite Action

For full control over your workflow, use the composite action directly. This requires you to define database services yourself.

#### Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `ruby-version` | Ruby version to use | Yes | `3.3` |
| `rails-version` | Rails version to use | Yes | `7.2` |
| `solidus-branch` | Solidus branch to use | Yes | `main` |
| `database` | Database to use (`sqlite`, `postgresql`, `mysql`, or `mariadb`) | Yes | `sqlite` |

#### Database Services

The composite GitHub Action **cannot define services directly**. If you need to test against PostgreSQL or MySQL/MariaDB, you must define the database service in your workflow file.

The action will install the necessary database client libraries automatically based on the `database` input.

#### Database Credentials

The action automatically sets the correct `DB_USERNAME` based on the database:

- **PostgreSQL**: `postgres` (default superuser)
- **MySQL/MariaDB**: `root` (default superuser)

Configure your database services with passwordless access for simplicity (see example below).

> [!WARNING]
> Rails 8.0 has a [known issue](https://github.com/rails/rails/issues/53673) with MySQL/MariaDB that causes empty `Mysql2::Error` messages. Until a fix is released, exclude the `mariadb` + Rails 8.0 combination from your test matrix.

#### Example Configuration

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
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_HOST_AUTH_METHOD: trust
        options: >-
          --health-cmd="pg_isready"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 5432:5432
      mysql:
        image: mysql:8
        env:
          MYSQL_ALLOW_EMPTY_PASSWORD: "yes"
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 3306:3306
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
