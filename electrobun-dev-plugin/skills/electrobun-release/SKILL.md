---
name: Electrobun Release
description: Use when distributing Electrobun apps, configuring auto-updates, uploading release artifacts, understanding bsdiff patches, or integrating the Updater API. Activates on release config, update.json, or artifact upload work.
version: 1.0.0
---

# Electrobun Release & Auto-Update

## Release Pipeline Overview

```
1. Bump app.version in electrobun.config.ts
2. electrobun build --env=canary   (or --env=stable)
3. Upload artifacts/ to release.baseUrl
4. Verify update.json is reachable
5. Running apps auto-detect and apply the update
```

## Config Requirements

```typescript
// electrobun.config.ts
app: {
  version: "1.2.3",  // bump this for each release
},
release: {
  baseUrl: "https://cdn.example.com/releases/myapp",
  generatePatch: true,  // creates bsdiff delta (smaller downloads)
},
```

The build system appends `/<os>-<arch>-update.json` to `baseUrl` to form the update manifest URL.
Example: `https://cdn.example.com/releases/myapp/macos-arm64-update.json`

## Canary vs Stable

| Channel | Use for | Version convention |
|---|---|---|
| `canary` | Beta testers, internal QA | `1.2.3-canary.4` or just `1.2.3` |
| `stable` | All users, production | `1.2.3` |

Apps subscribe to a channel. A stable app will not pick up a canary update.

## Artifact Files Explained

After `electrobun build --env=canary`:

```
artifacts/
├── macos-arm64-canary-1.2.3.tar.zst   # Full bundle (zstd compressed)
├── macos-arm64-canary-1.2.3.patch     # bsdiff patch from previous version
├── macos-arm64-update.json            # Manifest fetched by running apps
└── macos-arm64-canary-1.2.3.dmg      # Installer for fresh installs
```

**bsdiff patch**: Electrobun downloads the previous release's `.tar.zst` from `baseUrl`, diffs it against the new one, and writes a binary patch. If the previous release isn't found, patch generation is skipped (full bundle used instead).

## Uploading Artifacts

Upload everything in `artifacts/` to `<baseUrl>/`:

```bash
# S3 example
aws s3 sync artifacts/ s3://my-bucket/releases/myapp/ --acl public-read

# Cloudflare R2 example
rclone sync artifacts/ r2:my-bucket/releases/myapp/

# Generic rsync
rsync -avz artifacts/ user@server:/var/www/releases/myapp/
```

The `update.json` file must be at exactly `<baseUrl>/<os>-<arch>-update.json`.

## Updater API (in-app)

```typescript
import { Updater } from "electrobun/bun";

const updater = new Updater();

// Check for update
const status = await updater.checkForUpdates();
// status.hasUpdate: boolean
// status.version: string (new version if available)
// status.channel: "canary" | "stable"

if (status.hasUpdate) {
  // Download (uses patch if available, falls back to full bundle)
  await updater.downloadUpdate();

  // Apply on next relaunch
  await updater.applyUpdate();

  // Or: apply immediately (relaunches app)
  await updater.applyUpdate({ relaunch: true });
}

// One-liner
await updater.checkAndApply({ silent: true });
```

## Recommended Startup Pattern

```typescript
// src/bun/index.ts
import { Updater } from "electrobun/bun";

// Check in background, don't block app launch
setTimeout(async () => {
  try {
    const updater = new Updater();
    await updater.checkAndApply({ silent: true });
  } catch (e) {
    console.error("Update check failed:", e);
  }
}, 3000); // 3s after launch
```

## Release Checklist

- [ ] Bump `app.version` in `electrobun.config.ts`
- [ ] Set all signing env vars: `ELECTROBUN_DEVELOPER_ID`, `ELECTROBUN_APPLEID`, `ELECTROBUN_APPLEIDPASS`, `ELECTROBUN_TEAMID`
- [ ] Run `electrobun build --env=canary` (or `stable`)
- [ ] Verify `artifacts/` contains: `.tar.zst`, `.patch` (if not first release), `update.json`, `.dmg`
- [ ] Upload all artifacts to `release.baseUrl`
- [ ] Verify: `curl <baseUrl>/<os>-<arch>-update.json` returns valid JSON with correct version
- [ ] Test update in a running older version of the app

## Common Issues

1. **No patch file generated** → Previous release `.tar.zst` not found at `baseUrl`. First release always skips patch.
2. **Update not detected** → `update.json` URL is wrong. Must be `<baseUrl>/macos-arm64-update.json` (no version number in this file's name).
3. **`checkForUpdates()` returns `hasUpdate: false`** → App version matches `update.json` version. Bump `app.version`.
4. **Large patch size** → Binary assets changed. Put large static files in `build.copy` to exclude from bundle diff.
