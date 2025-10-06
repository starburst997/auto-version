# Auto Version

Automatically manage semantic versions based on git tags for production, dev, and RC releases.

## Features

- **Smart versioning**: Automatically calculates next version based on existing tags
- **Multiple release types**: Supports production, dev, and RC workflows
- **Customizable**: Configure git user details and default versions
- **Simple integration**: Drop-in replacement for manual versioning logic

## Inputs

| Input                | Description                                                             | Required | Default                                        |
| -------------------- | ----------------------------------------------------------------------- | -------- | ---------------------------------------------- |
| `release-type`       | Type of release (`production`, `dev`, `rc`)                             | Yes      | -                                              |
| `default-version`    | Default version if no tags exist                                        | No       | `1.0.0`                                        |
| `git-user-name`      | Git user name for tagging                                               | No       | `github-actions[bot]`                          |
| `git-user-email`     | Git user email for tagging                                              | No       | `github-actions[bot]@users.noreply.github.com` |
| `update-major-minor` | Update major/minor tags (e.g., `v1`, `v1.2`) to point to latest version | No       | `false`                                        |

## Outputs

| Output    | Description           | Example                  |
| --------- | --------------------- | ------------------------ |
| `version` | Calculated version    | `1.2.3` or `1.2.3-dev`   |
| `tag`     | Git tag with v prefix | `v1.2.3` or `v1.2.3-dev` |

## Requirements

Before using this action, ensure your workflow has:

1. **Checkout with full history**: Required to access all git tags

   ```yaml
   - uses: actions/checkout@v4
     with:
       fetch-depth: 0
   ```

2. **Write permissions**: Required to push tags to the repository
   ```yaml
   permissions:
     contents: write
   ```

## Usage

### Production Release

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    release-type: production

- name: Use Version
  run: |
    echo "Version: ${{ steps.version.outputs.version }}"
    echo "Tag: ${{ steps.version.outputs.tag }}"
```

### Dev Release

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    release-type: dev
    default-version: "1.0.0"
```

### RC Release

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    release-type: rc
```

### Custom Git User

```yaml
- name: Version and Tag
  id: version
  uses: starburst997/auto-version@v1
  with:
    release-type: production
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
    release-type: production
    update-major-minor: true
```

This updates floating tags based on release type:

- **Production**: `v1` and `v1.2` → point to `v1.2.3`
- **Dev**: `v1-dev` and `v1.2-dev` → point to `v1.2.3-dev`
- **RC**: `v1-rc` and `v1.2-rc` → point to `v1.2.3-rc.1`

Users can then reference `@v1` (production), `@v1-dev` (dev), or `@v1-rc` (rc) to always get the latest version.

## Full Workflow Example

### Production Workflow

```yaml
name: Production Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions: write-all
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Version and Tag
        id: version
        uses: starburst997/auto-version@v1
        with:
          release-type: production

      - name: Build and Push
        run: |
          echo "Building version ${{ steps.version.outputs.version }}"
          # Your build steps here
```

### Dev Workflow

```yaml
name: Dev Release

on:
  push:
    branches: [dev]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions: write-all
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Version and Tag
        id: version
        uses: starburst997/auto-version@v1
        with:
          release-type: dev

      - name: Build and Push
        run: |
          echo "Building version ${{ steps.version.outputs.version }}"
          # Your build steps here
```

## How It Works

### Production (`release-type: production`)

1. Checks for latest dev/RC tags
2. If found, uses that version as the next production version
3. If version already exists, increments minor version (hotfix)
4. If no dev/RC tags, increments minor from latest stable

### Dev (`release-type: dev`)

1. Compares latest stable and dev versions
2. If stable is newer (just released), bumps minor from stable (reset patch)
3. Otherwise, increments patch from latest dev
4. Appends `-dev` suffix

### RC (`release-type: rc`)

1. If no RC exists, creates from latest dev/stable + `-rc.1`
2. If RC exists, increments RC number (e.g., `1.2.3-rc.2`)

## License

MIT
