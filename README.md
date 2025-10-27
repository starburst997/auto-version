# Auto Version

Automatically manage semantic versions based on git tags for production, dev, and RC releases.

## Why?

Managing versions across different environments can be tedious. This action automates the entire versioning workflow:

- **Dev branch commits** → `v1.2.3-dev` - Automatically increment patch for each dev deployment
- **PR to dev** → `v0.0.0-pr-123.1` - Transient versions for PR preview environments
- **PR to main** → `v0.0.0-rc.1` - Transient release candidates for staging environments
- **Main branch** → `v1.2.3` - Production releases that strip the suffix from dev/rc versions
- **After production release** → Next dev commit bumps minor version and starts a new development cycle

This workflow enables:

- Easy deployment to Kubernetes with distinct version tags
- Automated semantic versioning without manual intervention
- Clear version progression from dev → rc → production
- No version conflicts across environments

## How It Works

The action automatically detects the workflow context and determines the version strategy:

### Push to Main Branch (Production)

1. Checks for latest dev tags
2. If found, uses that version as the next production version (strips suffix)
3. If version already exists, increments patch (hotfix scenario)
4. If no dev tags, increments patch from latest stable
5. Creates tag: `v1.2.3`

### Push to Dev Branch

1. Compares latest stable and dev versions
2. If stable is newer (just released to production), bumps minor from stable and resets patch
3. Otherwise, increments patch from latest dev
4. Appends `-dev` suffix
5. Creates tag: `v1.2.3-dev`

### Pull Request to Main Branch (Release Candidate)

1. Uses 0.0.0 as base for transient RC versions
2. Counts existing RC tags
3. Increments RC number
4. Creates tag: `v0.0.0-rc.1`, `v0.0.0-rc.2`, etc.

### Pull Request to Dev Branch (PR Preview)

1. Uses 0.0.0 as base for transient PR versions
2. Counts existing PR tags for this PR number
3. Increments PR build number
4. Creates tag: `v0.0.0-pr-123.1`, `v0.0.0-pr-123.2`, etc.

## Features

- **Smart versioning**: Automatically calculates next version based on existing tags
- **Multiple release types**: Supports production, dev, and RC workflows
- **Commit message commands**: Use `/major` or `/minor` in commits to control version bumps
- **Yearly versioning**: Optional year-based major versions (2025.0.0 format)
- **File editing**: Automatically update files with new version (e.g., action.yml for GitHub Actions)
- **Flexible git operations**: Optional push control for advanced workflows
- **Customizable**: Configure git user details and default versions
- **Simple integration**: Drop-in replacement for manual versioning logic

## Inputs

| Input                  | Description                                                                                                  | Required | Default                              |
| ---------------------- | ------------------------------------------------------------------------------------------------------------ | -------- | ------------------------------------ |
| `main-branch`          | Name of the main/production branch                                                                           | No       | `main`                               |
| `dev-branch`           | Name of the development branch                                                                               | No       | `dev`                                |
| `default-version`      | Default version if no tags exist                                                                             | No       | `1.0.0`                              |
| `git-user-name`        | Git user name for tagging and commits                                                                        | No       | `github-actions[bot]`                |
| `git-user-email`       | Git user email for tagging and commits                                                                       | No       | `...@users.noreply.github.com`       |
| `update-major-minor`   | Update major/minor tags (e.g., `v1`, `v1.2`) to point to latest version                                      | No       | `false`                              |
| `yearly`               | Use year as major version (e.g., 2025.0.0) and auto-update on new year                                       | No       | `false`                              |
| `git-push`             | Push changes and tags to remote repository                                                                   | No       | `true`                               |
| `no-tags`              | Skip tag creation and pushing (only calculate version)                                                       | No       | `false`                              |
| `edit-file`            | File to update with new version (e.g., `action.yml`). If empty, skipped                                      | No       | `` (empty)                           |
| `edit-search-pattern`  | Search pattern for version replacement                                                                       | No       | `${{ github.repository }}:`          |
| `edit-commit-message`  | Commit message template for file edit (use `{version}` placeholder)                                          | No       | `chore: update version to {version}` |
| `update-version-files` | Automatically update version in common version files (package.json, pyproject.toml, etc.) without committing | No       | `true`                               |
| `version-file-paths`   | Additional paths to search for version files (space or newline separated). Useful for monorepos              | No       | `` (empty)                           |
| `patch-reset-value`    | Value to reset PATCH to when incrementing MAJOR or MINOR                                                     | No       | `0`                                  |

## Outputs

| Output           | Description                                                             | Example                                 |
| ---------------- | ----------------------------------------------------------------------- | --------------------------------------- |
| `version`        | Calculated version                                                      | `1.2.3` or `1.2.3-dev`                  |
| `tag`            | Git tag with v prefix                                                   | `v1.2.3` or `v1.2.3-dev`                |
| `environment`    | Environment type                                                        | `production`, `dev`, `staging`, `pr-42` |
| `suffix`         | Version suffix (empty for production)                                   | `dev`, `rc`, `pr-42`, or empty          |
| `future-version` | Actual semantic version for transient environments (RC/PR) before 0.0.0 | `1.2.3` (when version is `0.0.0-rc.1`)  |

## Requirements

Before using this action, ensure your workflow has:

1. **Checkout with tags**: Required to access all git tags for version calculation

   ```yaml
   - uses: actions/checkout@v4
     with:
       fetch-tags: true
   ```

   This fetches all tags and their referenced commits (much faster than `fetch-depth: 0` which fetches entire history).

2. **Write permissions**: Required to push tags to the repository
   ```yaml
   permissions:
     contents: write
   ```

## Usage

The action automatically detects the context (branch or PR) and generates the appropriate version.

### Basic Usage

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1

- name: Use Version
  run: |
    echo "Version: ${{ steps.version.outputs.version }}"
    echo "Tag: ${{ steps.version.outputs.tag }}"
    echo "Environment: ${{ steps.version.outputs.environment }}"
    echo "Future Version: ${{ steps.version.outputs.future-version }}"
```

### Custom Branch Names

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    main-branch: "master"
    dev-branch: "develop"
```

### Custom Git User

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    git-user-name: "Release Bot"
    git-user-email: "bot@example.com"
```

### Update Major/Minor Tags

Enable automatic updating of major and minor version tags (useful for GitHub Actions):

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    update-major-minor: true
```

This updates floating tags based on release type:

- **Production**: `v1` and `v1.2` → point to `v1.2.3`
- **Dev**: `v1-dev` and `v1.2-dev` → point to `v1.2.3-dev`
- **RC/PR**: Floating tags skipped (transient versions use 0.0.0)

Users can then reference `@v1` (production) or `@v1-dev` (dev) to always get the latest version.

### File Editing

Automatically update files with the new version (useful for GitHub Actions that reference Docker images):

```yaml
- name: Version and Tag with File Update
  id: version
  uses: starburst997/auto-version@v1
  with:
    update-major-minor: true
    edit-file: "action.yml"
    edit-commit-message: "chore: update docker image to {version}"
```

This will update lines in `action.yml` that match the repository pattern:

```yaml
# Before
image: "docker://ghcr.io/user/repo:v1"

# After (if version is v1.2.3)
image: "docker://ghcr.io/user/repo:v1.2.3"
```

### Automatic Version File Updates

By default, the action automatically updates version numbers in common project files **without committing** the changes. This is useful for build processes that need the correct version in compiled artifacts:

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  # update-version-files defaults to true
```

Supported files:

- **JavaScript/TypeScript**: `package.json`
- **Python**: `pyproject.toml`, `setup.py`
- **Rust**: `Cargo.toml`
- **Go**: `VERSION` file
- **Ruby**: `*.gemspec` files
- **PHP**: `composer.json`
- **.NET**: `*.csproj` files (Version, AssemblyVersion, FileVersion)
- **Gradle**: `gradle.properties`
- **Maven**: `pom.xml`

To disable this behavior:

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    update-version-files: false
```

For monorepos, you can specify additional paths to search for version files:

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    version-file-paths: |
      packages/app
      packages/lib
      services/api
```

Or as a single line:

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    version-file-paths: "packages/app packages/lib services/api"
```

### Yearly Versioning

Use the current year as the major version:

```yaml
- name: Version with Year
  uses: starburst997/auto-version@v1
  with:
    yearly: true # Produces versions like 2025.0.0
```

### Commit Message Commands

Control version bumps via commit messages:

```yaml
# Commit with "/minor" in message → bumps minor version
# Commit with "/major" in message → bumps major version (ignored in yearly mode)
# Regular commit → bumps patch version (default)

- name: Version and Tag
  uses: starburst997/auto-version@v1
```

### Advanced Git Control

For workflows that need to control when git operations happen:

```yaml
- name: Version and Tag (No Push)
  id: version
  uses: starburst997/auto-version@v1
  with:
    git-push: false # Don't push yet
    edit-file: "action.yml"

# Run tests, builds, etc.
- name: Run Tests
  run: npm test

# Only push if everything succeeds
- name: Push Changes
  run: git push --follow-tags
```

## Full Workflow Examples

### Production Workflow

Automatically creates production versions when merging to main:

```yaml
name: Production Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-tags: true

      - name: Auto Version and Tag
        id: version
        uses: starburst997/auto-version@v1
        with:
          update-major-minor: true
          edit-file: "action.yml"
          edit-commit-message: "chore: update to {version}"

      - name: Create GitHub Release
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create ${{ steps.version.outputs.tag }} \
            --title "Release ${{ steps.version.outputs.version }}" \
            --generate-notes

      - name: Build and Deploy
        run: |
          echo "Building version ${{ steps.version.outputs.version }}"
          # Your build steps here
```

### Dev Workflow

Automatically creates dev versions on every push to dev branch:

```yaml
name: Dev Deploy

on:
  push:
    branches: [dev]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-tags: true

      - name: Auto Version and Tag
        id: version
        uses: starburst997/auto-version@v1

      - name: Build and Deploy
        run: |
          echo "Building version ${{ steps.version.outputs.version }}"
          # Your build steps here
```

### PR Workflow

Automatically creates RC versions for PRs to main, and PR-specific versions for PRs to dev:

```yaml
name: PR Deploy

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches:
      - main
      - dev

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-tags: true

      - name: Auto Version and Tag
        id: version
        uses: starburst997/auto-version@v1

      - name: Build and Deploy
        run: |
          echo "Building version ${{ steps.version.outputs.version }}"
          # PR to main: v0.0.0-rc.1 (staging)
          # PR to dev: v0.0.0-pr-123.1 (preview)
```

## License

MIT
