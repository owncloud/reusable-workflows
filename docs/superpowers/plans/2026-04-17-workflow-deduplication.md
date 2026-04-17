# Workflow Deduplication Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extract three composite actions into `.github/actions/` to eliminate duplicated steps across `php-codestyle.yml`, `php-unit.yml`, `js-unit.yml`, `translation-sync.yml`, and `calens.yml`.

**Architecture:** Three composite actions (`setup-owncloud-env`, `setup-php-server`, `github-app-pr`) live in `.github/actions/` and are referenced by workflows via `uses: ./.github/actions/<name>`. Each action consolidates a cluster of steps that are currently copy-pasted across multiple workflows. Callers reference them as local actions since they live in the same repo.

**Tech Stack:** GitHub Actions composite actions (YAML), `shivammathur/setup-php`, `actions/checkout`, `peter-evans/create-pull-request`, `actions/create-github-app-token`

---

### Task 1: Create `setup-owncloud-env` composite action

**Files:**
- Create: `.github/actions/setup-owncloud-env/action.yml`

Consolidates: validate app-repository, clone owncloud/core (with PHP 7.4 ref fallback), checkout app, optionally checkout additional-app.

- [ ] **Step 1: Create the action file**

Create `.github/actions/setup-owncloud-env/action.yml` with this exact content:

```yaml
name: Setup ownCloud Environment
description: Checkout owncloud/core, the app under test, and an optional additional app

inputs:
  app-name:
    description: App directory name under apps/
    required: true
  app-repository:
    description: Override repo for app checkout (must be owncloud/*)
    required: false
    default: ''
  core-ref:
    description: Ref to use for owncloud/core checkout
    required: false
    default: ''
  core-ref-php74:
    description: Override core ref when PHP version is 7.4
    required: false
    default: ''
  php-version:
    description: Current PHP version (used to select core ref when 7.4)
    required: false
    default: ''
  additional-app:
    description: Optional second app to check out
    required: false
    default: ''
  additional-app-github-token:
    description: Token for private additional-app repo
    required: false
    default: ''

runs:
  using: composite
  steps:
    - name: Validate app-repository
      if: ${{ inputs.app-repository != '' }}
      shell: bash
      env:
        APP_REPO: ${{ inputs.app-repository }}
      run: |
        if ! echo "$APP_REPO" | grep -qE '^owncloud/[a-zA-Z0-9._-]+$'; then
          echo "Error: app-repository must be within the owncloud/ namespace, got: $APP_REPO"
          exit 1
        fi

    - name: Clone owncloud/core
      uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      with:
        repository: owncloud/core
        ref: ${{ inputs.php-version == '7.4' && inputs.core-ref-php74 || inputs.core-ref }}

    - name: Checkout apps/${{ inputs.app-name }}
      uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      with:
        path: apps/${{ inputs.app-name }}
        repository: ${{ inputs.app-repository }}

    - name: Checkout apps/${{ inputs.additional-app }}
      if: ${{ inputs.additional-app != '' }}
      uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      with:
        path: apps/${{ inputs.additional-app }}
        repository: owncloud/${{ inputs.additional-app }}
        token: ${{ inputs.additional-app-github-token || github.token }}
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/actions/setup-owncloud-env/action.yml'))" && echo "VALID"
```

Expected output: `VALID`

- [ ] **Step 3: Commit**

```bash
git add .github/actions/setup-owncloud-env/action.yml
git commit -m "feat: add setup-owncloud-env composite action"
```

---

### Task 2: Create `setup-php-server` composite action

**Files:**
- Create: `.github/actions/setup-php-server/action.yml`

Consolidates: `shivammathur/setup-php` (single version pin), optional `apt-get install`, `make` (core deps), `php occ maintenance:install` with database switching, enable core apps.

- [ ] **Step 1: Create the action file**

Create `.github/actions/setup-php-server/action.yml` with this exact content:

```yaml
name: Setup PHP and ownCloud Server
description: Install PHP, optional Linux packages, core dependencies, and ownCloud server

inputs:
  php-version:
    description: PHP version to install
    required: true
  additional-packages:
    description: Extra apt-get packages (space-separated)
    required: false
    default: ''
  database:
    description: 'Database identifier, e.g. sqlite, mysql:8.0, mariadb:10.6, postgres:10.21'
    required: false
    default: sqlite
  database-name:
    description: Database name
    required: false
    default: owncloud
  database-user:
    description: Database user
    required: false
    default: owncloud
  database-pass:
    description: Database password
    required: false
    default: owncloud
  database-host:
    description: Database host
    required: false
    default: '127.0.0.1'

runs:
  using: composite
  steps:
    - name: Setup PHP
      uses: shivammathur/setup-php@accd6127cb78bee3e8082180cb391013d204ef9f # v2.37.0
      with:
        php-version: ${{ inputs.php-version }}
        extensions: apcu, ctype, curl, exif, fileinfo, gd, iconv, imagick, intl, json, mbstring, memcached, pdo, posix, simplexml, xml, zip, smbclient, ldap, krb5
        ini-values: "memory_limit=1024M"
      env:
        fail-fast: true
        KRB5_LINUX_LIBS: libkrb5-dev

    - name: Install Linux packages
      if: ${{ inputs.additional-packages != '' }}
      shell: bash
      env:
        ADDITIONAL_PACKAGES: ${{ inputs.additional-packages }}
      run: sudo apt-get install $ADDITIONAL_PACKAGES

    - name: Install Core Dependencies
      shell: bash
      run: make

    - name: Install Server
      shell: bash
      env:
        DATA_DIRECTORY: ${{ github.workspace }}/data
        DB: ${{ inputs.database }}
        DB_NAME: ${{ inputs.database-name }}
        ADMIN_LOGIN: admin
        ADMIN_PASSWORD: admin
        DB_HOST: ${{ inputs.database-host }}
        DB_USERNAME: ${{ inputs.database-user }}
        DB_PASSWORD: ${{ inputs.database-pass }}
      run: |
        DB_TYPE=${DB%:*}
        if [ "$DB_TYPE" = "postgres" ]; then DB_TYPE="pgsql"; fi
        if [ "$DB_TYPE" = "mariadb" ]; then DB_TYPE="mysql"; fi
        install_cmd="maintenance:install -vvv \
          --database=${DB_TYPE} \
          --database-name=${DB_NAME} \
          --admin-user=${ADMIN_LOGIN} \
          --admin-pass=${ADMIN_PASSWORD} \
          --data-dir=${DATA_DIRECTORY}"
        if [ "${DB_TYPE}" != "sqlite" ]; then
          install_cmd+=" --database-host=${DB_HOST} \
            --database-user=${DB_USERNAME} \
            --database-pass=${DB_PASSWORD}"
        fi
        php occ ${install_cmd}
        echo "enabling apps"
        php occ app:enable files_sharing
        php occ app:enable files_trashbin
        php occ app:enable files_versions
        php occ app:enable provisioning_api
        php occ app:enable federation
        php occ app:enable federatedfilesharing
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/actions/setup-php-server/action.yml'))" && echo "VALID"
```

Expected output: `VALID`

- [ ] **Step 3: Commit**

```bash
git add .github/actions/setup-php-server/action.yml
git commit -m "feat: add setup-php-server composite action"
```

---

### Task 3: Create `github-app-pr` composite action

**Files:**
- Create: `.github/actions/github-app-pr/action.yml`

Consolidates `actions/create-github-app-token` and `peter-evans/create-pull-request` into a single pinned location. Fixes version drift (v3.1.1 vs v1.11.6). The `condition` string input gates both steps so callers control branch conditions externally.

- [ ] **Step 1: Create the action file**

Create `.github/actions/github-app-pr/action.yml` with this exact content:

```yaml
name: Create GitHub App PR
description: Generate a GitHub App token and create or update a pull request

inputs:
  app-id:
    description: GitHub App ID
    required: true
  app-private-key:
    description: GitHub App private key
    required: true
  branch:
    description: PR branch name
    required: true
  base:
    description: Base branch for the PR (defaults to repo default branch)
    required: false
    default: ''
  commit-message:
    description: Commit message for the PR
    required: true
  title:
    description: PR title
    required: true
  body:
    description: PR body
    required: true
  reviewers:
    description: Comma-separated reviewer logins
    required: false
    default: ''
  team-reviewers:
    description: Comma-separated team slugs
    required: false
    default: ''
  sign-commits:
    description: Whether to sign commits
    required: false
    default: 'true'
  condition:
    description: Set to 'false' to skip both steps (pass a GitHub Actions expression result)
    required: false
    default: 'true'

runs:
  using: composite
  steps:
    - name: Generate GitHub App token
      id: app-token
      if: ${{ inputs.condition == 'true' }}
      uses: actions/create-github-app-token@1b10c78c7865c340bc4f6099eb2f838309f1e8c3 # v3.1.1
      with:
        app-id: ${{ inputs.app-id }}
        private-key: ${{ inputs.app-private-key }}

    - name: Create or update PR
      if: ${{ inputs.condition == 'true' }}
      uses: peter-evans/create-pull-request@5f6978faf089d4d20b00c7766989d076bb2fc7f1 # v8.1.1
      with:
        token: ${{ steps.app-token.outputs.token }}
        branch: ${{ inputs.branch }}
        base: ${{ inputs.base }}
        commit-message: ${{ inputs.commit-message }}
        title: ${{ inputs.title }}
        body: ${{ inputs.body }}
        delete-branch: true
        sign-commits: ${{ inputs.sign-commits }}
        reviewers: ${{ inputs.reviewers }}
        team-reviewers: ${{ inputs.team-reviewers }}
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/actions/github-app-pr/action.yml'))" && echo "VALID"
```

Expected output: `VALID`

- [ ] **Step 3: Commit**

```bash
git add .github/actions/github-app-pr/action.yml
git commit -m "feat: add github-app-pr composite action"
```

---

### Task 4: Update `php-codestyle.yml`

**Files:**
- Modify: `.github/workflows/php-codestyle.yml`

Replace the 8 duplicated steps (validate, 3× checkout, Linux packages, PHP setup, core deps, server install) with two composite action calls. Keep: setup additional app, setup app, and the three test steps.

- [ ] **Step 1: Replace the file**

Overwrite `.github/workflows/php-codestyle.yml` with this exact content:

```yaml
on:
  workflow_call:
    inputs:
      app-name:
        description: 'Name of the app'
        required: true
        type: string
      php-versions:
        description: JSON array of PHP versions
        required: true
        type: string
      core-ref:
        description: 'git reference to checkout for core'
        required: false
        type: string
        default: ''
      core-ref-php74:
        description: 'git reference to checkout for core if PHP 7.4 is used'
        required: false
        type: string
        default: ''
      app-repository:
        description: 'Repository of the app (optional, defaults to the current repository)'
        required: false
        type: string
        default: ''
      disable-phpstan:
        description: 'Whether to disable PHPStan checks'
        required: false
        type: boolean
        default: false
      disable-phan:
        description: 'Whether to disable Phan checks'
        required: false
        type: boolean
        default: false
      additional-app:
        description: 'Additional app to enable for testing'
        required: false
        type: string
        default: ''
      additional-app-github-token:
        description: 'GitHub PAT to access the additional app repo - required if private'
        required: false
        type: string
        default: ''
      additional-packages:
        description: Additional Linux packages to be installed - space separated
        required: false
        type: string
        default: ''

permissions:
  contents: read

jobs:
  php-codestyle:
    name: PHP Code Style
    runs-on: ubuntu-latest
    permissions:
      contents: read
    strategy:
      fail-fast: true
      matrix:
        php: ${{ fromJSON(inputs.php-versions) }}

    steps:
      - name: Setup ownCloud environment
        uses: ./.github/actions/setup-owncloud-env
        with:
          app-name: ${{ inputs.app-name }}
          app-repository: ${{ inputs.app-repository }}
          core-ref: ${{ inputs.core-ref }}
          core-ref-php74: ${{ inputs.core-ref-php74 }}
          php-version: ${{ matrix.php }}
          additional-app: ${{ inputs.additional-app }}
          additional-app-github-token: ${{ inputs.additional-app-github-token }}

      - name: Setup PHP and server
        uses: ./.github/actions/setup-php-server
        with:
          php-version: ${{ matrix.php }}
          additional-packages: ${{ inputs.additional-packages }}

      - name: Setup additional app
        if: ${{ inputs.additional-app != '' }}
        env:
          ADDITIONAL_APP: ${{ inputs.additional-app }}
        run: |
          ( cd "apps/${ADDITIONAL_APP}" && make vendor || true)
          php occ a:e "${ADDITIONAL_APP}"

      - name: Setup app
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          ( cd "apps/${APP_NAME}" && make vendor || true)
          php occ a:e "${APP_NAME}"

      - name: Run PHP Code Style Checks
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          cd "apps/${APP_NAME}"
          make test-php-style

      - name: Run PHP Stan
        if: ${{ !inputs.disable-phpstan }}
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          cd "apps/${APP_NAME}"
          make test-php-phpstan

      - name: Run PHP Phan
        if: ${{ !inputs.disable-phan }}
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          cd "apps/${APP_NAME}"
          make test-php-phan
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/php-codestyle.yml'))" && echo "VALID"
```

Expected output: `VALID`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/php-codestyle.yml
git commit -m "refactor: use composite actions in php-codestyle workflow"
```

---

### Task 5: Update `php-unit.yml`

**Files:**
- Modify: `.github/workflows/php-unit.yml`

Replace the 8 duplicated steps with two composite action calls. Pass `database: ${{ matrix.database }}` to `setup-php-server` so it handles the sqlite/mysql/mariadb/postgres switching. Keep: setup additional app, setup app, PHPUnit, integration tests, tail log.

- [ ] **Step 1: Replace the file**

Overwrite `.github/workflows/php-unit.yml` with this exact content:

```yaml
on:
  workflow_call:
    inputs:
      app-name:
        description: 'Name of the app'
        required: true
        type: string
      php-versions:
        description: JSON array of PHP versions
        required: true
        type: string
      core-ref:
        description: 'git reference to checkout for core'
        required: false
        type: string
        default: ''
      core-ref-php74:
        description: 'git reference to checkout for core if PHP 7.4 is used'
        required: false
        type: string
        default: ''
      app-repository:
        description: 'Repository of the app (optional, defaults to the current repository)'
        required: false
        type: string
        default: ''
      do-integration-tests:
        description: 'Whether to run PHP integration tests'
        required: false
        type: boolean
        default: false
      additional-app:
        description: 'Additional app to enable for testing'
        required: false
        type: string
        default: ''
      additional-app-github-token:
        description: 'GitHub PAT to access the additional app repo - required if private'
        required: false
        type: string
        default: ''
      additional-packages:
        description: Additional Linux packages to be installed - space separated
        required: false
        type: string
        default: ''

permissions:
  contents: read

jobs:
  php-unit:
    name: PHP Unit
    runs-on: ubuntu-latest
    permissions:
      contents: read
    strategy:
      fail-fast: true
      matrix:
        php: ${{ fromJSON(inputs.php-versions) }}
        database: [sqlite]
        include:
          - php: ${{ fromJSON(inputs.php-versions)[0] }}
            database: "mysql:8.0"
          - php: ${{ fromJSON(inputs.php-versions)[0] }}
            database: "mariadb:10.6"
          - php: ${{ fromJSON(inputs.php-versions)[0] }}
            database: "postgres:10.21"

    services:
      mysql:
        image: >-
          ${{ (startsWith(matrix.database,'mysql:') || startsWith(matrix.database,'mariadb:')) && matrix.database || '' }}
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: owncloud
          MYSQL_USER: owncloud
          MYSQL_PASSWORD: owncloud
        options: >-
          --health-cmd="mysqladmin ping -u root -ppassword"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3
        ports:
          - 3306:3306
      postgres:
        image: ${{ startsWith(matrix.database,'postgres:') && matrix.database || '' }}
        env:
          POSTGRES_DB: owncloud
          POSTGRES_USER: owncloud
          POSTGRES_PASSWORD: owncloud
        ports:
          - 5432:5432

    steps:
      - name: Setup ownCloud environment
        uses: ./.github/actions/setup-owncloud-env
        with:
          app-name: ${{ inputs.app-name }}
          app-repository: ${{ inputs.app-repository }}
          core-ref: ${{ inputs.core-ref }}
          core-ref-php74: ${{ inputs.core-ref-php74 }}
          php-version: ${{ matrix.php }}
          additional-app: ${{ inputs.additional-app }}
          additional-app-github-token: ${{ inputs.additional-app-github-token }}

      - name: Setup PHP and server
        uses: ./.github/actions/setup-php-server
        with:
          php-version: ${{ matrix.php }}
          additional-packages: ${{ inputs.additional-packages }}
          database: ${{ matrix.database }}

      - name: Setup additional app
        if: ${{ inputs.additional-app != '' }}
        env:
          ADDITIONAL_APP: ${{ inputs.additional-app }}
        run: |
          ( cd "apps/${ADDITIONAL_APP}" && make vendor || true)
          php occ a:e "${ADDITIONAL_APP}"

      - name: Setup app
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          ( cd "apps/${APP_NAME}" && make vendor || true)
          php occ a:e "${APP_NAME}"

      - name: Run PHPUnit
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          cd "apps/${APP_NAME}"
          make test-php-unit

      - name: Run PHP Integration tests
        if: ${{ inputs.do-integration-tests }}
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          cd "apps/${APP_NAME}"
          make test-php-integration

      - name: Tail log
        if: always()
        run: |
          ls -la data
          cat data/owncloud.log
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/php-unit.yml'))" && echo "VALID"
```

Expected output: `VALID`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/php-unit.yml
git commit -m "refactor: use composite actions in php-unit workflow"
```

---

### Task 6: Update `js-unit.yml`

**Files:**
- Modify: `.github/workflows/js-unit.yml`

Replace the 6 duplicated steps (validate, 2× checkout, Linux packages, PHP setup, core deps + server install) with two composite action calls. Note: `js-unit` has no `additional-app` or `core-ref-php74` inputs — omit those from the composite action calls. Combine the former "Install Dependencies" and "Setup App" steps into a single "Setup app" step (the `make` is now in `setup-php-server`).

- [ ] **Step 1: Replace the file**

Overwrite `.github/workflows/js-unit.yml` with this exact content:

```yaml
on:
  workflow_call:
    inputs:
      app-name:
        description: 'Name of the app'
        required: true
        type: string
      php-version:
        description: PHP version the server should be setup with
        required: true
        type: string
      core-ref:
        description: git reference to checkout for core
        required: false
        type: string
        default: ''
      app-repository:
        description: 'Repository of the app (optional, defaults to the current repository)'
        required: false
        type: string
        default: ''
      additional-packages:
        description: Additional Linux packages to be installed - space separated
        required: false
        type: string
        default: ''

permissions:
  contents: read

jobs:
  js-unit:
    name: Javascript Unit
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Setup ownCloud environment
        uses: ./.github/actions/setup-owncloud-env
        with:
          app-name: ${{ inputs.app-name }}
          app-repository: ${{ inputs.app-repository }}
          core-ref: ${{ inputs.core-ref }}
          php-version: ${{ inputs.php-version }}

      - name: Setup PHP and server
        uses: ./.github/actions/setup-php-server
        with:
          php-version: ${{ inputs.php-version }}
          additional-packages: ${{ inputs.additional-packages }}

      - name: Setup app
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          ( cd "apps/${APP_NAME}" && make vendor || true)
          php occ a:e "${APP_NAME}"

      - name: Run jsunit
        env:
          APP_NAME: ${{ inputs.app-name }}
        run: |
          cd "apps/${APP_NAME}"
          make test-js
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/js-unit.yml'))" && echo "VALID"
```

Expected output: `VALID`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/js-unit.yml
git commit -m "refactor: use composite actions in js-unit workflow"
```

---

### Task 7: Update `translation-sync.yml`

**Files:**
- Modify: `.github/workflows/translation-sync.yml`

Replace "Generate GitHub App token" + "Create or update translations PR" with a single `github-app-pr` composite action call. The `condition` input gates both steps (previously only PR creation was conditional; gating token generation too is a minor improvement). Note: `team_reviewers` input uses underscore in this workflow — pass it to `team-reviewers` (hyphen) in the action.

- [ ] **Step 1: Replace the last two steps**

The full updated file for `.github/workflows/translation-sync.yml`:

```yaml
on:
  workflow_call:
    inputs:
      mode:
        description: 'Translation mode: old, make, or native'
        required: true
        type: string
      branch:
        description: 'Branch to open the translations PR against'
        required: false
        type: string
        default: master
      sub_path:
        description: 'Path within the repo where translation files live (e.g. l10n or .)'
        required: false
        type: string
        default: l10n
      package_manager:
        description: 'Package manager for make mode: npm or pnpm'
        required: false
        type: string
        default: npm
      reviewers:
        description: 'Comma-separated list of GitHub usernames to request a review from'
        required: false
        type: string
        default: ''
      team_reviewers:
        description: 'Comma-separated list of GitHub team slugs to request a review from'
        required: false
        type: string
        default: ''
    secrets:
      TX_TOKEN:
        required: true
      TRANSLATION_APP_ID:
        required: true
      TRANSLATION_APP_PRIVATE_KEY:
        required: true

permissions:
  contents: read

jobs:
  sync:
    name: Translation Sync
    runs-on: ubuntu-latest
    permissions:
      contents: read
    container:
      image: owncloudci/transifex@sha256:b34396c63ce246a34ceb441b8c39067f7f46090b6de5f51dbf656e9be69d7185

    steps:
      - name: Checkout
        uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          ref: ${{ inputs.branch }}
          fetch-depth: 0
          persist-credentials: false

      - name: Create translation directory
        if: inputs.mode == 'old'
        env:
          SUB_PATH: ${{ inputs.sub_path }}
        run: mkdir -p "$SUB_PATH"

      - name: Translation reader (make)
        if: inputs.mode == 'make'
        env:
          NO_INSTALL: "true"
          SUB_PATH: ${{ inputs.sub_path }}
          PACKAGE_MANAGER: ${{ inputs.package_manager }}
        run: |
          cd "$SUB_PATH"
          if [ "$PACKAGE_MANAGER" = 'pnpm' ]; then
            npm install --silent --global --force "$(jq -r '.packageManager' < package.json)"
            pnpm config set store-dir ./.pnpm-store
            pnpm install
          fi
          make l10n-read

      - name: Translation reader (old)
        if: inputs.mode == 'old'
        env:
          SUB_PATH: ${{ inputs.sub_path }}
          REPO_NAME: ${{ github.event.repository.name }}
        run: |
          cd "$SUB_PATH"
          l10n "$REPO_NAME" read

      - name: Translation push (make)
        if: inputs.mode == 'make' && github.ref == format('refs/heads/{0}', inputs.branch)
        env:
          TX_TOKEN: ${{ secrets.TX_TOKEN }}
          SUB_PATH: ${{ inputs.sub_path }}
        run: |
          cd "$SUB_PATH"
          make l10n-push

      - name: Translation push (old)
        if: inputs.mode == 'old' && github.ref == format('refs/heads/{0}', inputs.branch)
        env:
          TX_TOKEN: ${{ secrets.TX_TOKEN }}
          SUB_PATH: ${{ inputs.sub_path }}
        run: |
          cd "$SUB_PATH"
          tx push -s --skip

      - name: Translation pull (make)
        if: inputs.mode == 'make'
        env:
          TX_TOKEN: ${{ secrets.TX_TOKEN }}
          SUB_PATH: ${{ inputs.sub_path }}
        run: |
          cd "$SUB_PATH"
          make l10n-pull

      - name: Translation pull (old/native)
        if: inputs.mode != 'make'
        env:
          TX_TOKEN: ${{ secrets.TX_TOKEN }}
          SUB_PATH: ${{ inputs.sub_path }}
        run: |
          cd "$SUB_PATH"
          tx pull -a --skip --minimum-perc=75 -f

      - name: Translation writer (make)
        if: inputs.mode == 'make'
        env:
          SUB_PATH: ${{ inputs.sub_path }}
        run: |
          cd "$SUB_PATH"
          make l10n-write

      - name: Translation writer (old)
        if: inputs.mode == 'old'
        env:
          SUB_PATH: ${{ inputs.sub_path }}
          REPO_NAME: ${{ github.event.repository.name }}
        run: |
          cd "$SUB_PATH"
          l10n "$REPO_NAME" write

      - name: Translation cleanup (make)
        if: inputs.mode == 'make'
        env:
          SUB_PATH: ${{ inputs.sub_path }}
        run: |
          cd "$SUB_PATH"
          make l10n-clean

      - name: Translation cleanup (old)
        if: inputs.mode == 'old'
        env:
          SUB_PATH: ${{ inputs.sub_path }}
        run: |
          cd "$SUB_PATH"
          find . -name "*.po" -type f -delete
          find . -name "*.pot" -type f -delete
          find . -name "or_IN.*" -type f -print0 | xargs -r -0 git rm -f
          find . -name "uz.*" -type f -print0 | xargs -r -0 git rm -f
          find . -name "yo.*" -type f -print0 | xargs -r -0 git rm -f
          find . -name "ne.*" -type f -print0 | xargs -r -0 git rm -f

      - name: Create or update translations PR
        uses: ./.github/actions/github-app-pr
        with:
          app-id: ${{ secrets.TRANSLATION_APP_ID }}
          app-private-key: ${{ secrets.TRANSLATION_APP_PRIVATE_KEY }}
          branch: chore/translations-update
          commit-message: "chore: update translations from transifex"
          title: "chore: update translations from transifex"
          body: "Automated translation update from Transifex. This pull request is updated on each sync run — merging it will close it and a fresh one will be opened on the next change."
          reviewers: ${{ inputs.reviewers }}
          team-reviewers: ${{ inputs.team_reviewers }}
          condition: ${{ github.ref == format('refs/heads/{0}', inputs.branch) }}
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/translation-sync.yml'))" && echo "VALID"
```

Expected output: `VALID`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/translation-sync.yml
git commit -m "refactor: use github-app-pr composite action in translation-sync"
```

---

### Task 8: Update `calens.yml`

**Files:**
- Modify: `.github/workflows/calens.yml`

Replace "Generate GitHub App token" + "Create or update changelog PR" with a single `github-app-pr` composite action call. Pass `base: ${{ inputs.branch }}` (calens-specific — translation-sync doesn't set this).

- [ ] **Step 1: Replace the file**

Overwrite `.github/workflows/calens.yml` with this exact content:

```yaml
on:
  workflow_call:
    inputs:
      branch:
        description: 'Branch to open the changelog PR against'
        required: false
        type: string
        default: master
      target:
        description: 'Target file for the generated changelog'
        required: false
        type: string
        default: CHANGELOG.md
    secrets:
      TRANSLATION_APP_ID:
        required: true
      TRANSLATION_APP_PRIVATE_KEY:
        required: true

jobs:
  changelog:
    name: Generate Changelog
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
        with:
          fetch-depth: 0
          persist-credentials: false

      - name: Generate changelog
        uses: actionhippie/calens@244f3e5c328b842a740113859b87bbebf697f63b # v1.13.1
        with:
          target: ${{ inputs.target }}

      - name: Show diff
        run: git diff

      - name: Output
        env:
          TARGET: ${{ inputs.target }}
        run: cat "$TARGET"

      - name: Create or update changelog PR
        uses: ./.github/actions/github-app-pr
        with:
          app-id: ${{ secrets.TRANSLATION_APP_ID }}
          app-private-key: ${{ secrets.TRANSLATION_APP_PRIVATE_KEY }}
          branch: chore/changelog-update
          base: ${{ inputs.branch }}
          commit-message: "chore: update changelog"
          title: "chore: update changelog"
          body: "Automated changelog update via Calens. This pull request is updated on each push to `${{ inputs.branch }}` — merging it will close it and a fresh one will be opened on the next change."
          condition: ${{ github.ref == format('refs/heads/{0}', inputs.branch) }}
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/calens.yml'))" && echo "VALID"
```

Expected output: `VALID`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/calens.yml
git commit -m "refactor: use github-app-pr composite action in calens"
```
