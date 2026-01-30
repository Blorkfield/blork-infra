# blork-infra

Shared CI/CD workflows for Blorkfield repositories.

## Workflows

### ci.yml

Build, typecheck, and auto-merge workflow.

**Inputs:**
| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `node_version` | string | `'22'` | Node.js version |
| `run_typecheck` | boolean | `true` | Run typecheck step |
| `typecheck_command` | string | `'pnpm typecheck'` | Typecheck command |
| `enable_automerge` | boolean | `true` | Auto-merge PRs after CI passes |

### publish.yml

Release PR creation, npm publishing (optional), and Docker publishing (optional).

**Inputs:**
| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `node_version` | string | `'22'` | Node.js version |
| `publish_npm` | boolean | `false` | Publish to npm |
| `npm_access` | string | `'public'` | npm access level |
| `publish_docker` | boolean | `false` | Publish Docker image |
| `docker_image_name` | string | `''` | Docker image name (used as `ghcr.io/blorkfield/<name>`) |
| `force_publish` | boolean | `false` | Force publish regardless of release detection |

### auto-pr.yml

Automatically create PRs and changesets based on branch naming conventions.

**Inputs:**
| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `package_name` | string | *required* | Package name for changeset |
| `watched_paths` | string | `'^(src/\|package\.json\|tsconfig\.json)'` | Regex pattern for paths to watch |

**Branch naming conventions:**
| Prefix | Type | Version Bump |
|--------|------|--------------|
| `feature/`, `feat/` | feat | minor |
| `fix/`, `bugfix/`, `hotfix/` | fix | patch |
| `breaking/` | feat | major |
| `chore/` | chore | patch |
| `docs/`, `doc/` | docs | patch |
| `refactor/` | refactor | patch |
| `perf/`, `performance/` | perf | patch |
| `test/`, `tests/` | test | patch |
| `style/` | style | patch |
| `ci/` | ci | patch |
| `build/` | build | patch |

## Setup

### 1. Required Secrets (Organization Level)

These should already be configured as organization secrets:
- `BOT_APP_ID` - GitHub App ID for bot operations
- `BOT_PRIVATE_KEY` - GitHub App private key

### 2. Required Repository Configuration

Each consuming repository needs:

#### Changeset Configuration

Create `.changeset/config.json`:
```json
{
  "$schema": "https://unpkg.com/@changesets/config@3.0.0/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "fixed": [],
  "linked": [],
  "access": "public",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": []
}
```

Create `.changeset/README.md`:
```markdown
# Changesets

This project uses [changesets](https://github.com/changesets/changesets) for versioning.

Changesets are auto-generated based on branch naming conventions.
```

Install changesets as a dev dependency:
```bash
pnpm add -D @changesets/cli
```

### 3. npm Trusted Publishers (For Repos Publishing to npm)

**This is critical for OIDC-based publishing with provenance.**

For each package that publishes to npm, configure trusted publishers on npmjs.com:

1. Go to your package settings on npmjs.com
2. Navigate to "Publishing access" → "Trusted publishers"
3. Add a new trusted publisher with:
   - **Repository owner:** `Blorkfield`
   - **Repository name:** `<your-repo-name>` (e.g., `overlay-core`)
   - **Workflow:** `publish.yml`
   - **Environment:** *(leave blank)*

**Important:** The trusted publisher is configured for the *calling* repository, not blork-infra. The OIDC token identifies where the workflow was triggered from.

No `NPM_TOKEN` secret is required - authentication is handled via OIDC.

## Usage Examples

### Repository with npm + Docker (like overlay-core)

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    uses: Blorkfield/blork-infra/.github/workflows/ci.yml@main
    secrets: inherit
```

```yaml
# .github/workflows/publish.yml
name: Publish
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      force_publish:
        type: boolean
        default: false

jobs:
  publish:
    uses: Blorkfield/blork-infra/.github/workflows/publish.yml@main
    secrets: inherit
    with:
      publish_npm: true
      publish_docker: true
      docker_image_name: 'overlay-core'
      force_publish: ${{ inputs.force_publish || false }}
```

```yaml
# .github/workflows/auto-pr.yml
name: Auto PR
on:
  push:
    branches-ignore:
      - main
      - 'release/**'

jobs:
  auto-pr:
    uses: Blorkfield/blork-infra/.github/workflows/auto-pr.yml@main
    secrets: inherit
    with:
      package_name: '@blorkfield/overlay-core'
      watched_paths: '^(src/|test-app/|fonts/|public/|package\.json|tsconfig\.json|tsup\.config\.ts|Dockerfile|nginx\.conf)'
```

### Repository with Docker Only (like obs-overlay)

```yaml
# .github/workflows/publish.yml
name: Publish
on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      force_publish:
        type: boolean
        default: false

jobs:
  publish:
    uses: Blorkfield/blork-infra/.github/workflows/publish.yml@main
    secrets: inherit
    with:
      publish_npm: false
      publish_docker: true
      docker_image_name: 'obs-overlay'
      force_publish: ${{ inputs.force_publish || false }}
```

## Pipeline Flow

```
┌─────────────────────────────────────────────────────────────┐
│              Push to Feature Branch                         │
│            (feat/*, fix/*, etc.)                            │
└─────────────────────────┬───────────────────────────────────┘
                          │
                   auto-pr.yml
                          │
            ┌─────────────────────────────┐
            │  • Parse branch name        │
            │  • Create changeset         │
            │  • Create PR to main        │
            └─────────────┬───────────────┘
                          │
                   PR triggers ci.yml
                          │
            ┌─────────────────────────────┐
            │  • Build                    │
            │  • Typecheck                │
            │  • Auto-merge (squash)      │
            └─────────────┬───────────────┘
                          │
                   Merge to main
                          │
            ┌─────────────────────────────┐
            │  publish.yml triggers       │
            │  • Detects changesets       │
            │  • Creates release PR       │
            └─────────────┬───────────────┘
                          │
                   Release PR merges
                   (commit: "chore(release):")
                          │
            ┌─────────────────────────────┐
            │  publish.yml triggers       │
            │  • Detects release merge    │
            │  • npm publish (if enabled) │
            │  • Docker push (if enabled) │
            └─────────────────────────────┘
```
