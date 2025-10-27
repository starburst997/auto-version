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

### Outputs

The action provides several outputs for use in subsequent workflow steps:

- **version** - Calculated version (e.g., `1.2.3`, `1.2.3-dev`, `0.0.0-rc.1`)
- **tag** - Git tag with `v` prefix (e.g., `v1.2.3`)
- **environment** - Environment type (`production`, `dev`, `staging`, `pr-###`)
- **suffix** - Version suffix (empty string, `dev`, `rc`, `pr-###`)
- **future-version** - For RC/PR environments, the semantic version that would be used when merged (not `0.0.0`)
- **stable-version** - Latest stable production version without pre-release suffix (always set, defaults to `default-version` if no stable tags exist)

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

## Documentation Requirements

**CRITICAL RULE**: Any changes to inputs, outputs, or functionality MUST be documented in ALL THREE locations:

1. **action.yml** - The source of truth for inputs/outputs with:
   - Input/output definition with description
   - Required flag and default values
   - Proper value mapping in steps

2. **README.md** - User-facing documentation with:
   - Updated inputs/outputs tables (must match action.yml exactly)
   - Usage examples showing new features
   - Clear descriptions and default values

3. **docs/index.html** - Website documentation with:
   - Updated reference cards for inputs/outputs (must match action.yml exactly)
   - Workflow examples if applicable
   - Consistent styling with existing content

**Mandatory checklist when adding/modifying inputs or outputs:**
- [ ] Update action.yml with the input/output definition
- [ ] Add to the inputs/outputs table in README.md with matching description
- [ ] Add to the corresponding reference card in docs/index.html with matching description
- [ ] Include examples demonstrating usage in README.md if applicable
- [ ] Update CLAUDE.md if implementation details or version calculation logic changes
- [ ] Ensure all three locations have consistent descriptions and examples

**NEVER skip any of these three documentation locations. Missing documentation in any location is considered incomplete work.**
