---
name: electrobun-release
description: Guided step-by-step release workflow for Electrobun apps — verifies config, checks signing identity, builds, notarizes (macOS), and configures the auto-updater.
---

Walk through the full release workflow for an Electrobun app.

## Steps

1. **Read `electrobun.config.ts`** and verify the release section exists. If `release.baseUrl` or `generatePatch` is missing, add them and ask the user for the base URL.

2. **Check app metadata** — confirm `app.version` is correct. Ask: "Is version X.Y.Z correct for this release?"

3. **Platform check** — ask which platform(s) they're releasing for: macOS / Windows / Linux / all.

4. **For macOS — verify code signing:**
   ```bash
   security find-identity -v -p codesigning
   ```
   - If no "Developer ID Application" identity is found, explain they need to enroll in Apple Developer Program.
   - Confirm the identity in `electrobun.config.ts` matches exactly (including the team ID in parentheses).

5. **For macOS — check notarization env vars:**
   ```bash
   echo "APPLE_ID: ${APPLE_ID:-NOT SET}"
   echo "APPLE_PASSWORD: ${APPLE_PASSWORD:-NOT SET}"
   ```
   - If not set, guide the user: "Set APPLE_ID and APPLE_PASSWORD (an app-specific password from appleid.apple.com) before building."

6. **Run the build:**
   ```bash
   electrobun build
   ```
   - If build fails, show the error and diagnose.

7. **After build — verify the output:**
   - macOS: check `build/*.app` is code-signed
     ```bash
     codesign -dv --verbose=4 build/<AppName>.app
     spctl --assess --verbose build/<AppName>.app
     ```
   - Windows: check `build/*.exe` exists

8. **Configure the Updater** — verify or add update check to `src/bun/index.ts`:
   ```typescript
   import { Updater } from "electrobun/bun";

   const updater = new Updater();
   updater.checkForUpdates().then(async (status) => {
     if (status.hasUpdate) {
       console.log("Update available:", status.version);
       await updater.downloadUpdate();
       await updater.applyUpdate();
     }
   }).catch(console.error);
   ```

9. **Summarize** what was done and what the user needs to do next (upload artifacts to `release.baseUrl`, bump version for next release, etc.).
