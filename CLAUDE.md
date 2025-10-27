# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitHub Action (composite action) that generates changelogs from commit history and optionally creates GitHub releases. It's written entirely in bash scripts embedded within the action.yml file.

## Architecture

The action consists of a single composite action definition (`action.yml`) with three main bash script steps:

1. **Get tags for changelog** (`steps.get-tags`)
   - Fetches and filters tags matching `vX.Y.Z` format
   - Intelligently determines commit range based on whether creating a new tag or just generating changelog
   - When `create-tag: false`: Auto-detects if HEAD is tagged (uses 2 most recent tags) or has new commits (uses latest tag to HEAD)
   - When `create-tag: true`: Always uses latest tag to HEAD
   - Supports manual tag specification via inputs

2. **Generate changelog** (`steps.generate`)
   - Parses git log between determined tag range
   - Supports two formats:
     - **Simple**: One line per commit `- message (hash)`
     - **Grouped**: Commits grouped by conventional commit types (feat, fix, docs, chore, other)
   - Uses AWK for parsing in grouped format
   - Handles shallow clones gracefully with warnings
   - Outputs to `CHANGELOG.md` file

3. **Create new tag** (`steps.create-new-tag`, conditional)
   - Only runs if `create-tag: true`
   - Auto-increments version from latest tag based on bump type (patch/minor/major)
   - Creates and pushes git tag

4. **Create release** (`steps.release`, conditional)
   - Only runs if `create-release: true`
   - Delegates to `ncipollo/release-action@v1`
   - Uses generated changelog as release body by default

## Development Workflow

### Testing Changes

Since this is a composite action with no build step, test by:
1. Pushing changes to a branch
2. Referencing the branch in a workflow: `uses: starburst997/commits-logs@branch-name`
3. The action uses itself in `.github/workflows/release.yml` for releases

### Release Process

The repository uses its own action for releases (`.github/workflows/release.yml`):
- Triggered on push to `main`
- Uses `starburst997/auto-version@v1` for versioning
- Runs the action itself (`uses: ./`) to create releases with grouped changelog format

## Key Technical Details

### Tag Detection Logic
- Tags are filtered using `git tag -l "v*.*.*"` and sorted with `sort -V` (version sort)
- The action intelligently detects whether to generate changelog for existing tags or new commits
- Supports shallow clones by checking if tags are reachable with `git rev-parse`

### Changelog Generation
- Simple format uses: `git log --pretty=format:"- %s (%h)"`
- Grouped format uses AWK to parse conventional commit prefixes
- Merge commits excluded by default with `--no-merges` (unless `include-merge-commits: true`)

### Version Bumping
- Parses semantic version from latest `vX.Y.Z` tag
- Increments major/minor/patch based on `version-bump` input
- Defaults to `v0.1.0` if no tags exist

## Important Constraints

- This is a composite action (not JavaScript/Docker)
- All logic is bash scripts in `action.yml`
- No separate source files or build process
- Must work in GitHub Actions environment only
- Depends on `ncipollo/release-action@v1` for release creation

## Documentation Requirements

**CRITICAL**: When adding or modifying inputs/outputs in `action.yml`, you MUST update ALL of the following documentation files:

1. **README.md** - Update the inputs/outputs tables with:
   - Parameter name and type
   - Description
   - Default value
   - Whether it's required

2. **index.html** - Update the documentation page with:
   - Input/output descriptions in the appropriate sections
   - Code examples showing usage of new parameters
   - Any relevant notes about behavior or constraints

**Never** add or modify inputs/outputs in `action.yml` without updating both README.md and index.html in the same change. The documentation must always stay synchronized with the code.
