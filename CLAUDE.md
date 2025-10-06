# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitHub Actions composite action for automatic semantic versioning based on git tags. It auto-detects workflow context (branch or PR) and generates appropriate version tags without manual configuration.

## Architecture

### Core Components

**action.yml** - Single composite action with three main steps:
1. **Calculate Version** - Bash script (lines 42-252) that auto-detects context and calculates next semantic version
2. **Create and Push Tag** - Creates and pushes the calculated version tag
3. **Update Major/Minor Tags** - Optionally maintains floating tags (v1, v1.2, etc.)

### Version Detection Logic

The action uses GitHub context variables to determine the workflow type:

- `github.event_name == 'pull_request'` → PR workflow
  - `github.base_ref == main-branch` → RC versioning (`v1.2.3-rc.1`)
  - `github.base_ref == dev-branch` → PR versioning (`v1.2.3-pr-123.1`)
- Push to `main-branch` → Production versioning (`v1.2.3`)
- Push to `dev-branch` → Dev versioning (`v1.2.3-dev`)

### Version Calculation Strategy

**Production (main branch):**
- Looks for latest dev/RC tags and strips suffix
- If version exists, increments minor (hotfix scenario)
- Falls back to incrementing minor from latest stable

**Dev (dev branch):**
- Compares latest stable vs dev versions
- If stable is newer (just released), bumps minor and resets patch
- Otherwise increments patch from latest dev
- Appends `-dev` suffix

**RC (PR to main):**
- Uses latest dev or stable as base
- Counts existing RC tags for base version
- Format: `v1.2.3-rc.N`

**PR (PR to dev):**
- Uses latest dev or stable as base
- Counts existing PR tags for this PR number
- Format: `v1.2.3-pr-{PR_NUMBER}.N`

## Key Implementation Details

### Tag Filtering Pattern
Uses `grep -v "dev\|rc\|pr"` to filter stable production tags from pre-release tags. All git commands include `|| true` to prevent pipeline failures when no tags exist.

### Error Handling
- All git tag commands use `|| true` to handle empty results gracefully
- Uses `set -eo pipefail` for proper error propagation
- Exits with clear error message for unsupported branches

### Floating Tags (update-major-minor)
When enabled, force-pushes major/minor tags:
- Production: `v1`, `v1.2` → `v1.2.3`
- Dev: `v1-dev`, `v1.2-dev` → `v1.2.3-dev`
- RC: `v1-rc`, `v1.2-rc` → `v1.2.3-rc.1`
- PR: No floating tags (skipped)

This enables users to reference `@v1` in GitHub Actions to always get the latest version.

## Testing the Action

Since this is a composite action, test locally by:

1. Create test tags: `git tag v1.0.0-dev && git push origin v1.0.0-dev`
2. Test in a workflow using `uses: ./` to reference the local action
3. Verify version calculation in workflow logs

## Workflow Requirements

Actions using this must have:
- `fetch-tags: true` in checkout (fetches all tags without full history)
- `permissions: contents: write` to push tags

## Self-Versioning

This repository uses itself for versioning via `.github/workflows/release.yml`:
- Runs on push to main
- Uses `update-major-minor: true` to maintain `v1` floating tag
- Integrates with `starburst997/commits-logs@v1` for changelog generation
