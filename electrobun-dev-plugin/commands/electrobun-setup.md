---
name: electrobun-setup
description: Run immediately after electrobun init. Creates all the scaffolding a production-ready Electrobun app needs — CI/CD workflows, release pipeline, test infrastructure, Mintlify docs, CLAUDE.md, shared type directories, gitignore entries, and pre-push hooks — so the project never drifts from best practices.
argument-hint: [template-name]
---

Set up a production-ready Electrobun project scaffold on top of a freshly initialized template.

## Step 1: Detect Project State

Read `electrobun.config.ts` to extract:
- `app.name`
- `app.identifier`
- Template type (infer from directory structure)

```bash
ls src/
```

Classify template type:
- **gpu**: has no `src/<viewname>/` dirs, has GPU-related code → `wgpu`, `wgpu-mlp`, `wgpu-babylon`, `wgpu-threejs`
- **webview**: has one `src/<viewname>/` dir → `hello-world`, `svelte`, `vue`, `vanilla-vite`, etc.
- **multi-window**: has multiple `src/<viewname>/` dirs → `multi-window`, `multitab-browser`
- **tray**: has tray code, minimal/no window → `tray-app`
- **sqlite**: has SQLite usage → `sqlite-crud`, `notes-app`

Announce:
> Detected template type: **[type]** | App: **[name]** | Identifier: **[identifier]**

---

## Step 2: User Questionnaire

Ask each question. Record answers. Do not apply anything yet.

```
Setup Wizard — answer Y/N or press Enter for default

1. Set up GitHub Actions CI?          [Y/n]
   (Runs bun check + type check on every PR)

2. Set up GitHub Actions release workflow?  [Y/n]
   (Builds and signs artifacts on version tags)

3. Set up release.baseUrl for updates?  [Y/n]
   If yes: Where will you host update artifacts?
   Options: r2 / s3 / github-releases / ssh / custom
   Enter URL: ___

4. Enable macOS code signing?          [Y/n]
   (Requires Apple Developer account)

5. Enable Windows build in CI?         [Y/n]

6. Enable Linux build in CI?           [Y/n]

7. Set up Mintlify documentation?      [Y/n]

8. Set up test infrastructure?         [Y/n]
   (vitest + coverage config, test scaffold directory)

9. Set up milady-ai contribution workflow?  [Y/n]
   (PR template, trust-compatible labels, Biome config)

10. Add CLAUDE.md with project-specific rules?  [Y/n]

11. Set up pre-push git hook?          [Y/n]
    (Runs type check before push)
```

Wait for all answers. Then confirm:
> Ready to create the following files: [list chosen files]. Proceed? [Y/n]

---

## Step 3: Apply Setup

Work through each section the user approved. Skip sections where user answered N.

### Section A: Core Files (always)

**`.gitignore` additions** — append if these patterns aren't already present:
```
# Electrobun build output
build/
artifacts/
node_modules/
*.env
.env.*
!.env.example
```

**`src/shared/`** — create shared types directory:
```bash
mkdir -p src/shared
```

Create `src/shared/.gitkeep` (placeholder so directory is tracked).

**`src/assets/`** — create assets directory:
```bash
mkdir -p src/assets
```

---

### Section B: GitHub Actions CI (if yes)

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:
  contents: read

jobs:
  check:
    name: Type Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest
      - run: bun install --frozen-lockfile
      - run: bun tsc --noEmit
```

If test infrastructure was also requested, add:
```yaml
      - run: bun test --coverage
```

---

### Section C: Release Workflow (if yes)

Collect platform selections (macOS always included; add Windows/Linux if user said yes).

Create `.github/workflows/release-electrobun.yml`. Template:

```yaml
name: Build & Release (Electrobun)

on:
  push:
    tags:
      - "v*"
  workflow_dispatch:
    inputs:
      tag:
        description: "Release tag (e.g. v1.0.0-alpha.1)"
        required: false
        type: string
      draft:
        description: "Create as draft release"
        required: false
        type: boolean
        default: true

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

permissions:
  contents: write

env:
  BUN_VERSION: latest

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.v.outputs.version }}
      env: ${{ steps.v.outputs.env }}
    steps:
      - uses: actions/checkout@v4
      - id: v
        run: |
          TAG="${{ inputs.tag || github.ref_name }}"
          VERSION="${TAG#v}"
          if echo "$VERSION" | grep -qE '(alpha|beta|rc|nightly)'; then
            ENV="canary"
          else
            ENV="stable"
          fi
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "env=$ENV" >> "$GITHUB_OUTPUT"

  build:
    needs: prepare
    strategy:
      matrix:
        include:
          - runner: macos-14
            os: macos
            arch: arm64
          - runner: macos-15-intel
            os: macos
            arch: x64
          # WINDOWS_ENABLED
          # LINUX_ENABLED
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version: ${{ env.BUN_VERSION }}
      - run: bun install --frozen-lockfile

      - name: Import Apple certificate (macOS)
        if: matrix.os == 'macos'
        run: |
          echo $MACOS_CERTIFICATE | base64 --decode > certificate.p12
          security create-keychain -p "" build.keychain
          security import certificate.p12 -k build.keychain \
            -P $MACOS_CERTIFICATE_PWD -T /usr/bin/codesign
          security list-keychains -d user -s build.keychain
          security set-keychain-settings -t 3600 -u build.keychain
          security unlock-keychain -p "" build.keychain
        env:
          MACOS_CERTIFICATE: ${{ secrets.MACOS_CERTIFICATE }}
          MACOS_CERTIFICATE_PWD: ${{ secrets.MACOS_CERTIFICATE_PWD }}

      - name: Install Linux deps
        if: matrix.os == 'linux'
        run: |
          sudo apt-get update
          sudo apt-get install -y build-essential cmake pkg-config \
            libgtk-3-dev libwebkit2gtk-4.1-dev libayatana-appindicator3-dev \
            librsvg2-dev fuse libfuse2

      - name: Build
        run: electrobun build --env=${{ needs.prepare.outputs.env }}
        env:
          ELECTROBUN_DEVELOPER_ID: ${{ secrets.ELECTROBUN_DEVELOPER_ID }}
          ELECTROBUN_APPLEID: ${{ secrets.ELECTROBUN_APPLEID }}
          ELECTROBUN_APPLEIDPASS: ${{ secrets.ELECTROBUN_APPLEIDPASS }}
          ELECTROBUN_TEAMID: ${{ secrets.ELECTROBUN_TEAMID }}

      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: artifacts-${{ matrix.os }}-${{ matrix.arch }}
          path: artifacts/

  release:
    needs: [prepare, build]
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: artifacts-*
          merge-multiple: true
          path: artifacts/

      - uses: softprops/action-gh-release@v1
        with:
          name: ${{ needs.prepare.outputs.version }}
          draft: ${{ inputs.draft || false }}
          files: artifacts/*
```

Insert Windows runner if user said yes (replace `# WINDOWS_ENABLED`):
```yaml
          - runner: windows-2025
            os: win
            arch: x64
```

Insert Linux runner if user said yes (replace `# LINUX_ENABLED`):
```yaml
          - runner: ubuntu-24.04
            os: linux
            arch: x64
          - runner: ubuntu-24.04-arm
            os: linux
            arch: arm64
```

---

### Section D: Update baseUrl (if yes)

Update `electrobun.config.ts` to add `release.baseUrl`:

```typescript
release: {
  baseUrl: "<user's chosen URL>/",
  generatePatch: true,
},
```

Prefix the URL based on their host choice:
- `github-releases`: `https://github.com/<owner>/<repo>/releases/download/`
- `r2`: `https://<accountid>.r2.cloudflarestorage.com/<bucket>/`
- `s3`: `https://<bucket>.s3.amazonaws.com/`
- `ssh` / `custom`: whatever URL they entered

---

### Section E: GitHub Secrets Checklist

Create `.github/SECRETS.md` (not committed — add to .gitignore — this is a local reference file):

```markdown
# Required GitHub Secrets

## Code Signing (macOS)
- MACOS_CERTIFICATE — base64-encoded .p12 Developer ID certificate
- MACOS_CERTIFICATE_PWD — certificate password
- ELECTROBUN_DEVELOPER_ID — "Developer ID Application: Name (TEAMID)"
- ELECTROBUN_APPLEID — Apple ID email
- ELECTROBUN_APPLEIDPASS — app-specific password from appleid.apple.com
- ELECTROBUN_TEAMID — 10-char Apple Team ID

## Artifact Upload (choose one)
### Cloudflare R2
- R2_ENDPOINT
- R2_ACCESS_KEY_ID
- R2_SECRET_ACCESS_KEY
- R2_BUCKET

### AWS S3
- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY
- AWS_DEFAULT_REGION
- S3_BUCKET
```

Add `.github/SECRETS.md` to `.gitignore`.

---

### Section F: Mintlify Documentation (if yes)

Create `docs/` directory structure:

```
docs/
├── mint.json
└── index.mdx
```

`docs/mint.json`:
```json
{
  "name": "<app.name>",
  "description": "Documentation for <app.name>",
  "navigation": [
    {
      "group": "Getting Started",
      "pages": ["index"]
    }
  ],
  "theme": "mint",
  "colors": {
    "primary": "#0969da"
  }
}
```

`docs/index.mdx`:
```mdx
---
title: "Getting Started"
description: "Welcome to <app.name>"
---

## Overview

<app.name> is a desktop app built with Electrobun.

## Quick Start

```bash
bun install
bun start
```

## Development

Run the development server:

```bash
bun run dev
```
```

---

### Section G: Test Infrastructure (if yes)

**For GPU templates** — skip Kitchen Sink tests, use direct assertions.

**For webview templates:**

Create `vitest.config.ts`:
```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    coverage: {
      provider: "v8",
      reporter: ["text", "json", "html"],
      thresholds: {
        lines: 25,
        functions: 25,
        statements: 25,
        branches: 15,
      },
    },
  },
});
```

Create `src/tests/` directory:
```bash
mkdir -p src/tests
```

Create `src/tests/index.ts`:
```typescript
// Test entry point — import test files here
// import "./example.test";
```

Create `src/tests/example.test.ts`:
```typescript
import { describe, it, expect } from "vitest";

describe("Sanity check", () => {
  it("should pass", () => {
    expect(true).toBe(true);
  });
});
```

Add test script to `package.json` if not present:
```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

---

### Section H: milady Contribution Setup (if yes)

**`.biome.json`:**
```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "organizeImports": { "enabled": true },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "suspicious": {
        "noExplicitAny": "error"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "space",
    "indentWidth": 2,
    "lineWidth": 100
  }
}
```

**`.github/PULL_REQUEST_TEMPLATE.md`:**
```markdown
## Summary

<!-- What does this PR do and why? -->

## Changes

<!-- List of files changed and what was changed -->

## Tests

<!-- What tests were added or updated? -->

## Follow-up

<!-- Optional: bullet points become GitHub issues automatically -->
```

**`.github/labeler.yml`** (for auto-labeling):
```yaml
bugfix:
  - head-branch: ["^fix/", "^bugfix/", "^hotfix/"]
feature:
  - head-branch: ["^feat/", "^feature/"]
docs:
  - changed-files:
    - any-glob-to-any-file: ["docs/**"]
chore:
  - head-branch: ["^chore/"]
```

---

### Section I: CLAUDE.md (if yes)

Create `CLAUDE.md` at project root:

```markdown
# CLAUDE.md

Guidance for Claude Code in the <app.name> Electrobun project.

## Commands

```bash
bun start              # Start dev server (electrobun dev)
bun run dev            # Dev with file watching
bun run build:canary   # Build for canary environment
bun run build:stable   # Build for stable environment
bun test               # Run tests
bun run test:coverage  # Run tests with coverage
```

## Architecture

- Main process: `src/bun/index.ts`
- Renderer views: `src/<viewname>/`
- Shared RPC types: `src/shared/`
- Config: `electrobun.config.ts`
- Tests: `src/tests/`
- Docs: `docs/`

## Key Patterns

- BrowserView RPC uses `BrowserView.defineRPC<MyRPC>()` (bun) and `new Electroview<MyRPC>()` (renderer)
- View URL scheme must match config key: `views.mainview` → `mainview://index.html`
- All GPU objects must be in `KEEPALIVE` array to prevent GC
- Do not use `win.webview.openDevTools()` — call `view.openDevTools()` on the BrowserView instance

## Plugin

This project uses the `electrobun-dev` Claude Code plugin.
- Start with `/electrobun-guide` to see all available skills and commands
- Use `/electrobun-workflow` to check your current pipeline stage
- Use `/electrobun-align` if the project drifts from conventions
```

---

### Section J: Pre-push Hook (if yes)

Create `.github/hooks/pre-push`:
```bash
#!/bin/sh
# Run type check before push
echo "Running type check..."
bun tsc --noEmit
if [ $? -ne 0 ]; then
  echo "Type check failed. Push aborted."
  exit 1
fi
echo "Type check passed."
```

Create install script `scripts/install-hooks.sh`:
```bash
#!/bin/sh
cp .github/hooks/pre-push .git/hooks/pre-push
chmod +x .git/hooks/pre-push
echo "Git hooks installed."
```

Tell user:
> Run `bash scripts/install-hooks.sh` to activate the pre-push hook locally.

---

## Step 4: Summary

After applying all selected sections, print:

```
✅ ELECTROBUN SETUP COMPLETE

Project: <app.name> (<app.identifier>)
Template type: <type>

Created:
  <list every file created>

Next steps:
  1. Run: bun install (if not done)
  2. Run: bun start  (verify template runs)
  3. Run: bash scripts/install-hooks.sh (if pre-push hook selected)
  4. Add GitHub Secrets (see .github/SECRETS.md — do not commit this file)
  5. Run /electrobun-workflow to see your current pipeline stage

Plugin tip: Run /electrobun-guide at any time for a full skills/commands reference.
```
