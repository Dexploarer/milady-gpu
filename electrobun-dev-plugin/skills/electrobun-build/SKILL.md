---
name: Electrobun Build
description: Use when running electrobun build, choosing a build environment, configuring code signing or notarization, understanding build artifacts, or debugging build failures.
version: 1.0.0
---

# Electrobun Build

## Commands

```bash
electrobun build                   # dev build (no sign, no package)
electrobun build --env=dev         # same as above
electrobun build --env=canary      # release build: sign + notarize + DMG + patch
electrobun build --env=stable      # same as canary, tagged as stable channel
```

## Build Environments Compared

| Feature | dev | canary | stable |
|---|---|---|---|
| Codesign | No | Yes (if configured) | Yes (if configured) |
| Notarize | No | Yes (if configured) | Yes (if configured) |
| DMG/installer | No | Yes | Yes |
| bsdiff patch | No | Yes (if `release.baseUrl`) | Yes (if `release.baseUrl`) |
| Self-extracting bundle | No | Yes | Yes |
| Output dir | `build/dev-*/` | `build/canary-*/` | `build/stable-*/` |
| Artifacts written | No | Yes → `artifacts/` | Yes → `artifacts/` |

## Code Signing (macOS)

Required env vars before running `--env=canary` or `--env=stable`:

```bash
export ELECTROBUN_DEVELOPER_ID="Developer ID Application: Your Name (TEAMID)"
export ELECTROBUN_APPLEID="developer@example.com"
export ELECTROBUN_APPLEIDPASS="xxxx-xxxx-xxxx-xxxx"  # app-specific password
export ELECTROBUN_TEAMID="XXXXXXXXXX"
```

Get app-specific password: https://appleid.apple.com → Security → App-Specific Passwords

Verify identity is available:
```bash
security find-identity -v -p codesigning | grep "Developer ID Application"
```

Skip signing even when configured:
```bash
ELECTROBUN_SKIP_CODESIGN=1 electrobun build --env=canary
```

## Required `electrobun.config.ts` for Release Builds

```typescript
build: {
  mac: {
    codesign: true,
    notarize: true,
    icons: "icon.iconset",   // 1024×1024 .iconset folder
  }
},
release: {
  baseUrl: "https://your-cdn.com/releases/myapp",
  generatePatch: true,
}
```

## Build Step Sequence

1. `scripts.preBuild` hook runs
2. Bundle bun entrypoint (`Bun.build()`)
3. Bundle each view entrypoint
4. Copy `build.copy` files
5. Assemble `.app` bundle (macOS), directory (Windows/Linux)
6. Copy launcher, native wrapper, CEF/WGPU libraries
7. Write `Info.plist`, `build.json`, `version.json`
8. `scripts.postBuild` hook runs
9. Codesign all binaries (canary/stable only)
10. Notarize + staple (canary/stable only)
11. `scripts.postWrap` hook runs
12. Tar the `.app` bundle
13. Compress with zstd
14. Build self-extracting wrapper
15. Download previous release → generate bsdiff patch
16. Create DMG (macOS) / archive (Linux/Windows)
17. `scripts.postPackage` hook runs
18. Write all artifacts to `build.artifactFolder/` (default: `artifacts/`)

## Hook Scripts

Scripts run with these env vars injected:

```bash
ELECTROBUN_BUILD_ENV      # dev | canary | stable
ELECTROBUN_OS             # macos | win | linux
ELECTROBUN_ARCH           # arm64 | x64
ELECTROBUN_BUILD_DIR      # /abs/path/to/build/
ELECTROBUN_APP_NAME       # sanitized filename (e.g. "MyApp")
ELECTROBUN_APP_VERSION    # from app.version
ELECTROBUN_APP_IDENTIFIER # from app.identifier
ELECTROBUN_ARTIFACT_DIR   # /abs/path/to/artifacts/
```

Example `scripts.postBuild`:
```bash
#!/bin/bash
echo "Built $ELECTROBUN_APP_NAME v$ELECTROBUN_APP_VERSION for $ELECTROBUN_OS-$ELECTROBUN_ARCH"
```

## Artifact Outputs (canary/stable)

```
artifacts/
├── macos-arm64-canary-1.2.3.tar.zst     # full compressed bundle
├── macos-arm64-canary-1.2.3.patch       # bsdiff delta from previous
├── macos-arm64-update.json              # auto-updater manifest
└── macos-arm64-canary-1.2.3.dmg        # installer
```

`update.json` format:
```json
{
  "version": "1.2.3",
  "channel": "canary",
  "url": "https://cdn.example.com/releases/myapp/macos-arm64-canary-1.2.3.tar.zst",
  "patchUrl": "https://cdn.example.com/releases/myapp/macos-arm64-canary-1.2.3.patch",
  "sha256": "abc123...",
  "patchSha256": "def456..."
}
```

## Version Overrides

Pin specific runtime versions in config (use sparingly — test before release):

```typescript
build: {
  cefVersion: "CEF_128+chromium-128.0.6613.84",
  wgpuVersion: "0.2.3",
  bunVersion: "1.4.2",
}
```

## Common Build Failures

1. **"No Developer ID Application certificate found"** → Check `ELECTROBUN_DEVELOPER_ID` matches exactly, including the TEAMID in parentheses
2. **Notarization timeout** → Apple servers slow; retry. Set `ELECTROBUN_SKIP_CODESIGN=1` to test the rest of the pipeline
3. **`icons` not found** → `build.mac.icons` must point to a `.iconset` folder containing `icon_1024x1024.png`
4. **bsdiff patch too large** → Large asset files changed. Move them to `build.copy` so they aren't re-bundled
5. **"Cannot find bun entrypoint"** → `build.bun.entrypoint` path doesn't exist. Check relative path from project root
