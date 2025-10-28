# Commits Log & Release Action

Generate changelogs from commits between releases and optionally create GitHub releases.

## Features

- 🏷️ **Smart Tag Filtering** - Automatically finds latest vX.Y.Z tags or specify manually
- 📝 **Changelog Generation** - Generate from commits between two tags
- 🚀 **Optional Release Creation** - Create GitHub releases with generated changelog
- 📊 **Two Formats** - Simple list or grouped by conventional commit types
- 📋 **Draft Release Detection** - Automatically detects and publishes draft releases

## Quick Start

### Basic Usage - Generate Changelog Only

```yaml
- uses: starburst997/commits-logs@v1
  id: changelog

- run: echo "${{ steps.changelog.outputs.changelog }}"
```

This intelligently detects the commit range:

- **If HEAD is tagged**: Uses the 2 most recent tags
- **If HEAD has new commits**: Uses latest tag to HEAD

### Create Release with Changelog

```yaml
- uses: starburst997/commits-logs@v1
  with:
    create-release: true
    create-tag: true
```

This will:

1. Find commits since the latest `vX.Y.Z` tag
2. Generate a changelog
3. Create a new tag (auto-increments patch version)
4. Create a GitHub release with the changelog

## Full Example

```yaml
name: Release

on:
  push:
    branches:
      - main

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Required for tag detection

      - name: Generate changelog and create release
        uses: starburst997/commits-logs@v1
        with:
          format: grouped
          create-tag: true
          create-release: true
          release-name: "v{tag}"
```

## Inputs

| Input                      | Description                                                                                                                                               | Default                    |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------- |
| `from-tag`                 | Starting tag (auto-detects latest vX.Y.Z if not provided)                                                                                                 | _(auto)_                   |
| `to-tag`                   | Ending tag/ref                                                                                                                                            | `HEAD`                     |
| `format`                   | `simple` or `grouped`                                                                                                                                     | `grouped`                  |
| `include-merge-commits`    | Include merge commits                                                                                                                                     | `false`                    |
| `changelog-title`          | Changelog title                                                                                                                                           | `## Changes`               |
| `create-tag`               | Create a new tag                                                                                                                                          | `false`                    |
| `version-bump`             | Version bump type: `patch`, `minor`, or `major`                                                                                                           | `patch`                    |
| `new-tag`                  | Tag name (auto-increments if not provided)                                                                                                                | _(auto)_                   |
| `create-release`           | Create GitHub release                                                                                                                                     | `true`                     |
| `release-name`             | Release name (supports `{tag}` variable)                                                                                                                  | `Release {tag}`            |
| `release-body`             | Custom release body (uses changelog if empty)                                                                                                             | _(changelog)_              |
| `draft`                    | Create draft release                                                                                                                                      | `false`                    |
| `prerelease`               | Mark as prerelease                                                                                                                                        | `false`                    |
| `make-latest`              | Mark as latest                                                                                                                                            | `true`                     |
| `generate-release-notes`   | Use GitHub's automatic release notes                                                                                                                      | `false`                    |
| `auto-pick-notes`          | Automatically choose between GitHub's generateReleaseNotes (if PRs detected) or custom changelog (if no PRs). Overrides generate-release-notes when true. | `true`                     |
| `artifacts`                | Newline-delimited list of artifact paths                                                                                                                  | _(none)_                   |
| `artifact-content-type`    | Content type for artifacts                                                                                                                                | `application/octet-stream` |
| `commit`                   | Commit to tag for release                                                                                                                                 | _(current)_                |
| `discussion-category-name` | Create discussion in specified category                                                                                                                   | _(none)_                   |
| `token`                    | GitHub token for releases                                                                                                                                 | `${{ github.token }}`      |

## Outputs

| Output           | Description                             |
| ---------------- | --------------------------------------- |
| `changelog`      | Generated changelog content             |
| `changelog-file` | Path to changelog file (`CHANGELOG.md`) |
| `from-tag`       | Tag used as starting point              |
| `to-tag`         | Tag used as ending point                |
| `new-tag`        | New tag created (if `create-tag: true`) |
| `release-url`    | URL of created release                  |
| `release-id`     | ID of created release                   |

## Changelog Formats

### Simple Format (default)

```markdown
## Changes

- Add user authentication (a1b2c3d)
- Fix login bug (e4f5g6h)
- Update documentation (i7j8k9l)
```

### Grouped Format

Groups commits by conventional commit prefixes:

```markdown
## Changes

### Features

- user authentication (a1b2c3d)
- two-factor auth support (m1n2o3p)

### Bug Fixes

- login redirect issue (e4f5g6h)
- session timeout bug (q4r5s6t)

### Documentation

- API usage guide (i7j8k9l)

### Other Changes

- Refactor auth module (u7v8w9x)
```

## Advanced Usage

### Specify Tag Range Manually

```yaml
- uses: starburst997/commits-logs@v1
  with:
    from-tag: v1.0.0
    to-tag: v1.1.0
```

### Create Release with Artifacts

```yaml
- uses: starburst997/commits-logs@v1
  with:
    create-release: true
    create-tag: true
    artifacts: |
      dist/*.zip
      dist/*.tar.gz
```

### Minor or Major Version Bump

```yaml
- uses: starburst997/commits-logs@v1
  with:
    create-tag: true
    version-bump: minor # v1.2.3 → v1.3.0
```

### Generate Changelog Between Any Two Tags

```yaml
- uses: starburst997/commits-logs@v1
  with:
    from-tag: v2.0.0
    to-tag: v2.1.0
    format: grouped
```

## How It Works

**Draft Release Detection:**

Before generating a new release, the action automatically checks for existing draft releases:

1. Looks for draft releases with titles matching the current version pattern (e.g., `vX.Y` for a `vX.Y.Z` tag)
2. If a matching draft is found:
   - Publishes the draft release (removes draft status)
   - Updates the release title to the full version (e.g., `v1.2.3`)
   - Updates the release tag to match the latest tag
   - Skips changelog generation and release creation steps
3. If no draft is found, proceeds with normal changelog generation and release creation

This feature is useful for workflows where you maintain a draft release and want to publish it when a new tag is created.

**When `create-tag: false` (default):**

1. Fetches all tags from the repository
2. Filters tags matching `vX.Y.Z` format and sorts by version
3. Intelligently detects commit range:
   - **If latest tag = HEAD**: Uses 2 most recent tags (e.g., `v1.0.0..v1.1.0`)
   - **If HEAD has new commits**: Uses latest tag to HEAD (e.g., `v1.1.0..HEAD`)
4. Checks for draft releases (see above)
5. Generates changelog from detected range (if no draft found)

**When `create-tag: true`:**

1. Fetches all tags from the repository
2. Filters tags matching `vX.Y.Z` format and sorts by version
3. Uses latest tag to HEAD
4. Checks for draft releases (see above)
5. Generates changelog from commits between latest tag and HEAD (if no draft found)
6. Creates new tag with auto-increment:
   - `patch` (default): `v1.2.3` → `v1.2.4`
   - `minor`: `v1.2.3` → `v1.3.0`
   - `major`: `v1.2.3` → `v2.0.0`
7. Optionally creates GitHub release with changelog (if no draft found)

## Requirements

### Full History (Recommended)

For complete changelogs, checkout with full history:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

### Shallow Clone Support

The action also works with shallow clones, using whatever commit history is available:

```yaml
- uses: actions/checkout@v4
  with:
    fetch-depth: 100 # Last 100 commits
```

**Note:** If tags are outside the shallow history, the action will generate a changelog from available commits only (with a warning). This is acceptable for most use cases where 50-100 commits cover the release.

## License

MIT
