---
name: Electrobun Build & Distribution
description: Use when configuring builds, code signing, notarization, auto-updates, or distribution for Electrobun apps. Activates when editing electrobun.config.ts or running build/release tasks.
version: 1.0.0
---

# Electrobun Build & Distribution

## Full ElectrobunConfig Reference

```typescript
import { defineConfig } from "electrobun/config";

export default defineConfig({
  // ── App metadata ──────────────────────────────────────────────────────────
  app: {
    name: "MyApp",               // Display name
    identifier: "com.co.myapp", // Reverse DNS — must be unique
    version: "1.0.0",           // Semver
    description: "My app",
    urlSchemes: ["myapp"],      // Deep links: myapp://path
  },

  // ── Build ─────────────────────────────────────────────────────────────────
  build: {
    bun: {
      entrypoint: "src/bun/index.ts",
      minify: true,
      sourcemap: "external",
    },
    views: {
      // One entry per BrowserView renderer
      mainview: { entrypoint: "src/mainview/index.ts" },
      sidebar: { entrypoint: "src/sidebar/index.ts" },
    },
    copy: [
      { from: "assets/", to: "assets/" },
      { from: "data/db.sqlite", to: "data/db.sqlite" },
    ],

    // ── Platform-specific ────────────────────────────────────────────────
    mac: {
      bundleWGPU: false,        // Include WGPU native lib (~8MB)
      bundleCEF: false,         // Include Chromium Embedded (~120MB)
      defaultRenderer: "native",// "native" | "cef"
      chromiumFlags: [],        // Extra Chromium flags (CEF only)
      codesign: {
        identity: "Developer ID Application: Name (TEAMID)",
        entitlements: "entitlements.plist",
      },
      notarize: {
        teamId: "XXXXXXXXXX",
        // Uses APPLE_ID + APPLE_PASSWORD env vars
      },
      icon: "assets/icon.icns",
    },
    win: {
      bundleWGPU: false,
      bundleCEF: false,
      defaultRenderer: "native",
      icon: "assets/icon.ico",
    },
    linux: {
      bundleWGPU: false,
      bundleCEF: false,
      defaultRenderer: "native",
    },
  },

  // ── Runtime ───────────────────────────────────────────────────────────────
  runtime: {
    exitOnLastWindowClosed: true,
  },

  // ── Build lifecycle scripts ───────────────────────────────────────────────
  scripts: {
    preBuild: "bun run scripts/generate-icons.ts",
    postBuild: "bun run scripts/verify-build.ts",
    postWrap:  "echo Wrap complete",
    postPackage: "bun run scripts/upload-symbols.ts",
  },

  // ── Auto-update / release ─────────────────────────────────────────────────
  release: {
    baseUrl: "https://releases.example.com/myapp",
    generatePatch: true,
  },
});
```

## Build Commands

```bash
# Development
electrobun dev              # Dev mode (no build, uses source directly)
electrobun dev --watch      # Rebuild on file changes

# Production
electrobun build            # Full production build
electrobun build --env=canary  # Canary/staging build

# Run
electrobun run              # Run the last built app
```

## Auto-Updater

```typescript
import { Updater } from "electrobun/bun";

const updater = new Updater();
const status = await updater.checkForUpdates();

if (status.hasUpdate) {
  await updater.downloadUpdate();
  await updater.applyUpdate();
}

// Or: check and apply automatically
await updater.checkAndApply({ silent: false });
```

Update files are served from `release.baseUrl`. With `generatePatch: true`, Electrobun creates a bsdiff patch — updates can be as small as 14KB.

## Code Signing (macOS)

```bash
export APPLE_ID="developer@example.com"
export APPLE_PASSWORD="app-specific-password"

electrobun build

# Verify signing
codesign -dv --verbose=4 build/MyApp.app
spctl --assess --verbose build/MyApp.app
```

Identity format: `"Developer ID Application: Full Name (TEAMID)"`

## Common Build Issues

1. **`bundleWGPU`/`bundleCEF` not set** → runtime "module not found" error. Set for ALL platforms you target.
2. **Icon missing** → build will warn and use a placeholder. Provide `.icns` (mac), `.ico` (win), `.png` (linux).
3. **`views` entry missing** → renderer HTML loads but JS fails silently.
4. **Notarization fails** → most common cause is missing entitlements. Hardened runtime requires `com.apple.security.cs.allow-jit` for Bun.
5. **Patch update too large** → `generatePatch: true` only helps if assets haven't changed. Put large assets in `copy` to avoid re-bundling.
