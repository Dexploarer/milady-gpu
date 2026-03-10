# electrobun-dev Plugin Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a full-spectrum Claude Code plugin (`electrobun-dev`) that provides auto-activating skills, scaffolding commands, specialized agents, and a linting hook for Electrobun desktop app development.

**Architecture:** A plugin installed at `~/.claude/plugins/electrobun-dev/` consisting of markdown/JSON/shell files only — no compilation step. Skills auto-activate based on context; commands are invoked explicitly; agents are dispatched by Claude; the hook runs on every Write/Edit tool call.

**Tech Stack:** Claude Code plugin format (YAML-frontmatter markdown, JSON config, bash); Electrobun framework knowledge embedded as skill content.

---

## Chunk 1: Plugin Scaffold

### Task 1: Create plugin root and manifest

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/.claude-plugin/plugin.json`

- [ ] **Step 1: Create the plugin directory structure**

```bash
mkdir -p ~/.claude/plugins/electrobun-dev/.claude-plugin
mkdir -p ~/.claude/plugins/electrobun-dev/commands
mkdir -p ~/.claude/plugins/electrobun-dev/agents
mkdir -p ~/.claude/plugins/electrobun-dev/skills/electrobun
mkdir -p ~/.claude/plugins/electrobun-dev/skills/electrobun-rpc
mkdir -p ~/.claude/plugins/electrobun-dev/skills/electrobun-webgpu
mkdir -p ~/.claude/plugins/electrobun-dev/skills/electrobun-build
mkdir -p ~/.claude/plugins/electrobun-dev/hooks/scripts
```

- [ ] **Step 2: Write the plugin manifest**

Write to `~/.claude/plugins/electrobun-dev/.claude-plugin/plugin.json`:
```json
{
  "name": "electrobun-dev",
  "version": "1.0.0",
  "description": "Agentic development companion for Electrobun desktop apps — skills, commands, agents, and hooks for the full development lifecycle",
  "author": {
    "name": "electrobun-dev"
  },
  "keywords": ["electrobun", "desktop", "webgpu", "bun", "rpc", "wgsl"]
}
```

- [ ] **Step 3: Validate the manifest JSON**

```bash
jq . ~/.claude/plugins/electrobun-dev/.claude-plugin/plugin.json
```
Expected: JSON prints cleanly with no errors.

---

## Chunk 2: Skills

### Task 2: Write the core Electrobun skill

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/skills/electrobun/SKILL.md`

- [ ] **Step 1: Write `skills/electrobun/SKILL.md`**

```markdown
---
name: Electrobun Core
description: Use when working on any Electrobun desktop app — BrowserWindow, BrowserView, events, app lifecycle, ApplicationMenu, Tray, and electrobun.config.ts. Activates automatically when editing Electrobun project files.
version: 1.0.0
---

# Electrobun Core Patterns

Electrobun is a cross-platform desktop app framework (macOS/Windows/Linux) using Bun as runtime and a native system webview (or CEF) as renderer. The bun process and renderer run as separate processes; they communicate via RPC (see electrobun-rpc skill).

## Project Structure

```
src/
├── bun/          # Main process (Bun side)
│   └── index.ts  # Entry point
└── mainview/     # Renderer process
    ├── index.html
    ├── index.css
    └── index.ts
```

## electrobun.config.ts

```typescript
import { defineConfig } from "electrobun/config";

export default defineConfig({
  app: {
    name: "MyApp",
    identifier: "com.example.myapp",
    version: "1.0.0",
    urlSchemes: ["myapp"], // enables myapp:// deep links
  },
  build: {
    bun: { entrypoint: "src/bun/index.ts" },
    views: {
      mainview: { entrypoint: "src/mainview/index.ts" },
    },
    mac: {
      bundleWGPU: false,
      bundleCEF: false,
      defaultRenderer: "native", // or "cef"
      codesign: { identity: "Developer ID Application: ..." },
      notarize: { teamId: "XXXXXXXXXX" },
      icon: "assets/icon.icns",
    },
    win: { bundleWGPU: false, bundleCEF: false, icon: "assets/icon.ico" },
    linux: { bundleWGPU: false, bundleCEF: false },
    copy: [{ from: "assets/", to: "assets/" }],
  },
  runtime: { exitOnLastWindowClosed: true },
  scripts: {
    preBuild: "bun run generate-assets.ts",
    postBuild: "echo Build complete",
  },
  release: {
    baseUrl: "https://releases.example.com/myapp",
    generatePatch: true,
  },
});
```

## BrowserWindow

```typescript
import { BrowserWindow } from "electrobun/bun";

const win = new BrowserWindow({
  title: "My App",
  frame: { x: 100, y: 100, width: 1200, height: 800 },
  url: "http://localhost:3000",        // OR
  html: "<h1>Hello</h1>",             // inline HTML
  preload: "src/mainview/preload.ts",  // injected before content
  renderer: "native",                  // "native" | "cef"
  titleBarStyle: "hiddenInset",        // "default" | "hidden" | "hiddenInset"
  transparent: false,
  sandbox: false,                      // true = no RPC, limited events
});
```

The primary webview is `win.webview` (a BrowserView instance).

## BrowserView (additional views)

```typescript
import { BrowserView } from "electrobun/bun";

const sidebar = new BrowserView({
  url: "src/sidebar/index.html",
  frame: { x: 0, y: 0, width: 300, height: 800 },
  renderer: "native",
  rpc: sidebarRPC,
  navigationRules: {
    allowedUrls: ["http://localhost:*"],
    deniedUrls: ["*"],
  },
});

win.addBrowserView(sidebar);
```

Each BrowserView is a separate renderer process.

## Events

```typescript
import { Electrobun } from "electrobun/bun";

Electrobun.events.on("open-url", (e) => {
  console.log("Deep link:", e.data.url);
});

Electrobun.events.on("application-menu-clicked", (e) => {
  const { action, role } = e.data;
  win.webview.rpc?.send.menuAction({ action, role });
});

win.on("close", () => { /* window closed */ });
win.webview.on("dom-ready", () => { /* safe to interact with page */ });
win.webview.on("will-navigate", (e) => { /* can cancel navigation */ });
win.webview.on("did-navigate", (e) => { /* navigation complete */ });
win.webview.on("page-title-updated", (e) => { /* title changed */ });
```

## ApplicationMenu

```typescript
import { ApplicationMenu } from "electrobun/bun";

ApplicationMenu.setMenu([
  {
    label: "File",
    submenu: [
      { label: "New", action: "file-new", accelerator: "CmdOrCtrl+N" },
      { label: "Open", action: "file-open", accelerator: "CmdOrCtrl+O" },
      { type: "separator" },
      { label: "Quit", role: "quit" },
    ],
  },
  {
    label: "Edit",
    submenu: [
      { label: "Undo", role: "undo" },
      { label: "Redo", role: "redo" },
      { type: "separator" },
      { label: "Cut", role: "cut" },
      { label: "Copy", role: "copy" },
      { label: "Paste", role: "paste" },
    ],
  },
]);
```

## Tray

```typescript
import { Tray } from "electrobun/bun";

const tray = new Tray({
  icon: "assets/tray-icon.png",
  tooltip: "My App",
});

tray.setMenu([
  { label: "Show", action: "tray-show" },
  { label: "Quit", role: "quit" },
]);
// Note: tray click events do NOT fire on Linux (AppIndicator limitation)
```

## Platform Quirks

- **Linux**: ApplicationMenu renders as app menu bar. Tray click events don't fire with AppIndicator.
- **Windows**: `titleBarStyle: "hiddenInset"` has no effect — use `"hidden"`.
- **macOS**: Full native feel. All APIs work as documented.
- **CEF renderer**: Requires `bundleCEF: true` in config; adds ~120MB to bundle size.

## CLI Commands

```bash
bunx electrobun init            # Scaffold new project
electrobun dev                  # Dev mode
electrobun dev --watch          # Dev with hot reload
electrobun build                # Production build
electrobun build --env=canary   # Canary build
electrobun run                  # Run built app
```
```

- [ ] **Step 2: Verify the file was written**

```bash
head -5 ~/.claude/plugins/electrobun-dev/skills/electrobun/SKILL.md
```
Expected: Shows the YAML frontmatter starting with `---` and `name: Electrobun Core`.

---

### Task 3: Write the RPC skill

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/skills/electrobun-rpc/SKILL.md`

- [ ] **Step 1: Write `skills/electrobun-rpc/SKILL.md`**

```markdown
---
name: Electrobun RPC
description: Use when working with Electrobun's RPC system — defineElectrobunRPC, Electroview.defineRPC, ElectrobunRPCSchema types, bun-to-renderer and renderer-to-bun communication. Activates automatically when RPC code is present.
version: 1.0.0
---

# Electrobun RPC System

Type-safe bidirectional communication between the Bun process and webview renderers. Uses WebSockets for transport with AES-GCM encryption. Two call types: `requests` (expects a response) and `messages` (fire-and-forget).

## Schema Definition (shared type)

Define the schema in a shared file both sides import:

```typescript
// src/shared/rpc-schema.ts
import type { ElectrobunRPCSchema } from "electrobun/bun";

export type AppRPC = ElectrobunRPCSchema<{
  requests: {
    // renderer asks bun for data
    getNotes: { args: {}; response: Note[] };
    saveNote: { args: { id: string; title: string; content: string }; response: { success: boolean } };
    deleteNote: { args: { id: string }; response: void };
    // bun asks renderer for data
    getSelectedText: { args: {}; response: string };
  };
  messages: {
    // bun sends to renderer (no response)
    menuAction: { action: string; role?: string };
    noteUpdatedExternally: { id: string };
    // renderer sends to bun (no response)
    userIdleDetected: {};
  };
}>;
```

## Bun Side — BrowserView.defineRPC

```typescript
import { BrowserView } from "electrobun/bun";
import type { AppRPC } from "../shared/rpc-schema";

const appRPC = BrowserView.defineRPC<AppRPC>({
  maxRequestTime: 10000, // ms — set higher for file dialogs, heavy DB ops
  handlers: {
    requests: {
      getNotes: async () => {
        return await db.getAllNotes();
      },
      saveNote: async ({ id, title, content }) => {
        await db.saveNote({ id, title, content });
        return { success: true };
      },
      deleteNote: async ({ id }) => {
        await db.deleteNote(id);
      },
    },
    messages: {
      userIdleDetected: () => {
        console.log("User idle, saving state...");
      },
    },
  },
});

// Pass rpc to BrowserView
const view = new BrowserView({ url: "...", rpc: appRPC });

// Send a message to renderer
view.rpc.send.menuAction({ action: "file-new" });

// Make a request to renderer
const selectedText = await view.rpc.request.getSelectedText({});
```

## Renderer Side — Electroview.defineRPC

```typescript
// src/mainview/index.ts
import { Electroview } from "electrobun/browser";
import type { AppRPC } from "../shared/rpc-schema";

const rpc = Electroview.defineRPC<AppRPC>({
  maxRequestTime: 10000,
  handlers: {
    requests: {
      getSelectedText: () => {
        return window.getSelection()?.toString() ?? "";
      },
    },
    messages: {
      menuAction: ({ action, role }) => {
        handleMenuAction(action ?? role ?? "unknown");
      },
      noteUpdatedExternally: ({ id }) => {
        reloadNote(id);
      },
    },
  },
});

// Make a request to bun
const notes = await rpc.request.getNotes({});

// Send a message to bun
rpc.send.userIdleDetected({});
```

## Key Rules

- **`requests`**: bidirectional, async, returns a value. Use for: data fetching, file I/O, native dialogs.
- **`messages`**: fire-and-forget, no return value. Use for: notifications, state updates, events.
- **`maxRequestTime`**: always set explicitly. Default may be too low for:
  - File save dialogs: set ≥ 30000
  - Database operations: set ≥ 10000
  - Quick in-memory ops: 5000 is fine
- **Type safety**: the schema type is shared — import it on both sides. Never duplicate the type definition.
- **Encryption**: RPC uses AES-GCM. Keys are auto-generated per session. Don't bypass this.

## Common Gotchas

1. **RPC not available yet**: Wait for `dom-ready` event before calling `view.rpc.request.*`.
2. **maxRequestTime timeout**: Error message is generic. If you see "RPC timeout", increase `maxRequestTime`.
3. **sandbox: true**: Disables RPC entirely. Don't set sandbox on windows that need RPC.
4. **Schema mismatch**: If bun and renderer import different schema types, calls silently fail. Use a single shared type file.
5. **Renderer not initialized**: `Electroview.defineRPC` must be called before any `rpc.request.*` calls.
```

- [ ] **Step 2: Verify the file was written**

```bash
head -5 ~/.claude/plugins/electrobun-dev/skills/electrobun-rpc/SKILL.md
```
Expected: Shows `name: Electrobun RPC` in frontmatter.

---

### Task 4: Write the WebGPU skill

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/skills/electrobun-webgpu/SKILL.md`

- [ ] **Step 1: Write `skills/electrobun-webgpu/SKILL.md`**

```markdown
---
name: Electrobun WebGPU
description: Use when working with WebGPU in Electrobun — GpuWindow, WGPUView, WGSL shaders, KEEPALIVE pattern, render loops, FFI pointer management, and GPU buffer serialization.
version: 1.0.0
---

# Electrobun WebGPU Patterns

Electrobun wraps WGPU (a Rust WebGPU abstraction) via Bun FFI. GPU windows bypass the webview entirely — they render directly to a native surface.

## Config Requirement

**Always required before WebGPU code will work:**
```typescript
// electrobun.config.ts
mac: { bundleWGPU: true },
win: { bundleWGPU: true },
linux: { bundleWGPU: true },
```

## GpuWindow Setup

```typescript
import { GpuWindow } from "electrobun/bun";

const gpuWin = new GpuWindow({
  title: "GPU App",
  frame: { width: 800, height: 600 },
  centered: true,
});

const view = gpuWin.createView(); // WGPUView
```

## KEEPALIVE — Critical Pattern

Bun's GC will collect FFI pointers unless you hold a reference to them. Without KEEPALIVE, your app will crash mid-render with a segfault.

```typescript
// ALWAYS create this array and push every GPU object into it
const KEEPALIVE: unknown[] = [];

const adapter = await navigator.gpu.requestAdapter();
KEEPALIVE.push(adapter);

const device = await adapter.requestDevice();
KEEPALIVE.push(device);

const pipeline = device.createRenderPipeline({ /* ... */ });
KEEPALIVE.push(pipeline);

const buffer = device.createBuffer({ /* ... */ });
KEEPALIVE.push(buffer);
```

## Render Loop Pattern

```typescript
const FRAME_MS = 16; // ~60fps

function renderFrame() {
  // 1. Update uniform buffer with current state
  const data = new ArrayBuffer(32);
  const view = new DataView(data);
  view.setFloat32(0, performance.now() / 1000, true); // time
  view.setFloat32(4, canvas.width, true);             // resolution.x
  view.setFloat32(8, canvas.height, true);            // resolution.y
  view.setFloat32(12, mouseX, true);                  // mouse.x
  view.setFloat32(16, mouseY, true);                  // mouse.y
  device.queue.writeBuffer(uniformBuffer, 0, data);

  // 2. Create command encoder
  const encoder = device.createCommandEncoder();
  KEEPALIVE.push(encoder);

  // 3. Begin render pass
  const pass = encoder.beginRenderPass({
    colorAttachments: [{
      view: context.getCurrentTexture().createView(),
      clearValue: { r: 0, g: 0, b: 0, a: 1 },
      loadOp: "clear",
      storeOp: "store",
    }],
  });

  // 4. Draw
  pass.setPipeline(pipeline);
  pass.setVertexBuffer(0, vertexBuffer);
  pass.draw(3);
  pass.end();

  // 5. Submit
  device.queue.submit([encoder.finish()]);
}

setInterval(renderFrame, FRAME_MS);
```

## WGSL Shader Structure

```wgsl
// Vertex shader
struct VertexInput {
  @location(0) position: vec2f,
  @location(1) time: f32,
  @location(2) resolution: vec2f,
  @location(3) mouse: vec2f,
}

struct VertexOutput {
  @builtin(position) position: vec4f,
  @location(0) uv: vec2f,
  @location(1) time: f32,
  @location(2) resolution: vec2f,
  @location(3) mouse: vec2f,
}

@vertex
fn vs_main(in: VertexInput) -> VertexOutput {
  var out: VertexOutput;
  out.position = vec4f(in.position, 0.0, 1.0);
  out.uv = (in.position + vec2f(1.0)) * 0.5;
  out.time = in.time;
  out.resolution = in.resolution;
  out.mouse = in.mouse;
  return out;
}

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4f {
  // Your shader logic here
  return vec4f(in.uv, sin(in.time) * 0.5 + 0.5, 1.0);
}
```

## Vertex Buffer with DataView

GPU structs must be manually serialized. Vertex layout: `[x, y, time, res.x, res.y, mouse.x, mouse.y]` (all f32).

```typescript
// 3 vertices × 7 floats × 4 bytes = 84 bytes
const vertexData = new ArrayBuffer(3 * 7 * 4);
const dv = new DataView(vertexData);

// Full-screen triangle vertices
const verts = [
  [-1, -1], [3, -1], [-1, 3],
];

verts.forEach(([x, y], i) => {
  const offset = i * 7 * 4;
  dv.setFloat32(offset + 0,  x, true);    // position.x
  dv.setFloat32(offset + 4,  y, true);    // position.y
  dv.setFloat32(offset + 8,  time, true); // time
  dv.setFloat32(offset + 12, width, true);
  dv.setFloat32(offset + 16, height, true);
  dv.setFloat32(offset + 20, mouseX, true);
  dv.setFloat32(offset + 24, mouseY, true);
});
```

## Common Mistakes

1. **No KEEPALIVE** → segfault or silent corruption mid-render.
2. **`bundleWGPU: false`** → runtime error; Electrobun won't include the WGPU native library.
3. **Not recreating swap chain on resize** → distorted rendering after window resize.
4. **Forgetting `little-endian: true`** in `DataView.setFloat32` → garbage GPU data on most platforms.
5. **Blocking the render loop** → use async I/O outside the render interval, never inside.
```

- [ ] **Step 2: Verify**

```bash
grep -c "KEEPALIVE" ~/.claude/plugins/electrobun-dev/skills/electrobun-webgpu/SKILL.md
```
Expected: number > 1.

---

### Task 5: Write the build skill

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/skills/electrobun-build/SKILL.md`

- [ ] **Step 1: Write `skills/electrobun-build/SKILL.md`**

```markdown
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
      // Any Bun.build() options can go here
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
        entitlements: "entitlements.plist", // optional
      },
      notarize: {
        teamId: "XXXXXXXXXX",
        // Uses APPLE_ID + APPLE_PASSWORD env vars
      },
      icon: "assets/icon.icns", // 1024×1024 recommended
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
    exitOnLastWindowClosed: true, // Quit app when last window closes
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
    generatePatch: true, // Generate bsdiff patch for small incremental updates
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

// Check for updates on startup
const updater = new Updater();
const status = await updater.checkForUpdates();

if (status.hasUpdate) {
  // Download and prepare update
  await updater.downloadUpdate();
  // Apply on next restart
  await updater.applyUpdate();
}

// Or: check and apply automatically
await updater.checkAndApply({ silent: false });
```

Update files are served from `release.baseUrl`. With `generatePatch: true`, Electrobun creates a bsdiff patch — updates can be as small as 14KB.

## Code Signing (macOS)

```bash
# Set up environment
export APPLE_ID="developer@example.com"
export APPLE_PASSWORD="app-specific-password"  # From appleid.apple.com

# Build + sign + notarize
electrobun build

# Verify signing
codesign -dv --verbose=4 build/MyApp.app
spctl --assess --verbose build/MyApp.app
```

Identity format: `"Developer ID Application: Full Name (TEAMID)"`

## Common Build Issues

1. **`bundleWGPU`/`bundleCEF` not set** → runtime "module not found" error. Set for ALL platforms you target.
2. **Icon missing** → build will warn and use a placeholder. Provide `.icns` (mac), `.ico` (win), `.png` (linux).
3. **`views` entry missing** → renderer HTML loads but JS fails silently. Every renderer with a `.ts` entry needs a `views` entry.
4. **Notarization fails** → most common cause is missing entitlements. Hardened runtime requires `com.apple.security.cs.allow-jit` for Bun.
5. **Patch update too large** → `generatePatch: true` only helps if assets haven't changed. Put large assets in `copy` to avoid re-bundling.
```

- [ ] **Step 2: Verify**

```bash
grep "bundleWGPU" ~/.claude/plugins/electrobun-dev/skills/electrobun-build/SKILL.md | wc -l
```
Expected: several matches.

---

## Chunk 3: Commands

### Task 6: Write the init command

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/commands/electrobun-init.md`

- [ ] **Step 1: Write `commands/electrobun-init.md`**

```markdown
---
name: electrobun-init
description: Scaffold a new Electrobun desktop app project from scratch. Creates full directory structure, package.json, electrobun.config.ts, tsconfig.json, and entry point files.
---

Scaffold a new Electrobun project in the current directory (or a subdirectory if the user provides a name).

## Steps

1. **Ask the user** for the following if not already provided:
   - App name (e.g. `my-app`)
   - App identifier in reverse DNS format (e.g. `com.example.myapp`)
   - Project type: `webview` (standard UI app), `webgpu` (GPU rendering only), or `both`

2. **Create the directory structure:**
   ```
   <name>/
   ├── src/
   │   ├── bun/
   │   │   └── index.ts
   │   └── mainview/          (if webview or both)
   │       ├── index.html
   │       ├── index.css
   │       └── index.ts
   ├── assets/
   ├── package.json
   ├── electrobun.config.ts
   └── tsconfig.json
   ```

3. **Write `package.json`:**
   ```json
   {
     "name": "<name>",
     "version": "0.0.1",
     "scripts": {
       "start": "electrobun dev",
       "dev": "electrobun dev --watch",
       "build": "electrobun build",
       "build:canary": "electrobun build --env=canary"
     },
     "dependencies": {
       "electrobun": "latest"
     },
     "devDependencies": {
       "typescript": "^5.7.0",
       "@types/bun": "latest"
     }
   }
   ```

4. **Write `tsconfig.json`:**
   ```json
   {
     "compilerOptions": {
       "target": "ESNext",
       "module": "ESNext",
       "moduleResolution": "bundler",
       "strict": true,
       "noUnusedLocals": true,
       "noUnusedParameters": true
     }
   }
   ```

5. **Write `electrobun.config.ts`** with the correct content based on type:
   - For `webview`: `bundleWGPU: false`, include `views.mainview`
   - For `webgpu`: `bundleWGPU: true`, no `views`
   - For `both`: `bundleWGPU: true`, include `views.mainview`

6. **Write `src/bun/index.ts`** with a working skeleton based on type:

   For **webview**:
   ```typescript
   import { BrowserWindow, Electrobun } from "electrobun/bun";

   const win = new BrowserWindow({
     title: "<App Name>",
     frame: { width: 1200, height: 800 },
     url: "mainview://index.html",
   });

   win.on("close", () => {
     if (Electrobun.getAllWindows().length === 0) {
       process.exit(0);
     }
   });
   ```

   For **webgpu**:
   ```typescript
   import { GpuWindow } from "electrobun/bun";

   const KEEPALIVE: unknown[] = [];

   const win = new GpuWindow({
     title: "<App Name>",
     frame: { width: 800, height: 600 },
     centered: true,
   });

   const adapter = await navigator.gpu.requestAdapter();
   KEEPALIVE.push(adapter);
   if (!adapter) throw new Error("No GPU adapter found");

   const device = await adapter.requestDevice();
   KEEPALIVE.push(device);

   // TODO: create pipeline, buffers, and start render loop
   setInterval(() => {
     // render frame here
   }, 16);
   ```

7. **Write renderer files** (if webview or both):

   `src/mainview/index.html`:
   ```html
   <!DOCTYPE html>
   <html>
   <head>
     <meta charset="UTF-8" />
     <title><App Name></title>
     <link rel="stylesheet" href="index.css" />
   </head>
   <body>
     <div id="app"></div>
     <script type="module" src="index.js"></script>
   </body>
   </html>
   ```

   `src/mainview/index.ts`:
   ```typescript
   import { Electroview } from "electrobun/browser";

   const rpc = Electroview.defineRPC<any>({
     maxRequestTime: 10000,
     handlers: {
       requests: {},
       messages: {},
     },
   });

   document.getElementById("app")!.innerHTML = "<h1>Hello from <App Name></h1>";
   ```

   `src/mainview/index.css`:
   ```css
   * { box-sizing: border-box; margin: 0; padding: 0; }
   body { font-family: system-ui, sans-serif; }
   ```

8. **Run `bun install`** in the new project directory.

9. **Tell the user** the project is ready. Show the commands: `bun start` to run, `bun run dev` for watch mode.
```

- [ ] **Step 2: Verify frontmatter**

```bash
head -6 ~/.claude/plugins/electrobun-dev/commands/electrobun-init.md
```
Expected: `name: electrobun-init` visible.

---

### Task 7: Write the window command

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/commands/electrobun-window.md`

- [ ] **Step 1: Write `commands/electrobun-window.md`**

```markdown
---
name: electrobun-window
description: Generate a new BrowserWindow with a matching renderer entry point and type-safe RPC schema connecting them. Usage: /electrobun-window [name]
---

Generate a new BrowserWindow + BrowserView pair with a working RPC connection.

## Steps

1. **Determine the window name** from the argument or ask the user (e.g. `settings`, `about`, `editor`).

2. **Read the existing codebase** to understand:
   - Where `src/bun/index.ts` is
   - Whether a shared RPC schema file exists (e.g. `src/shared/rpc.ts`)
   - What renderer pattern is used (native vs CEF)

3. **Create `src/<name>/index.html`**:
   ```html
   <!DOCTYPE html>
   <html>
   <head>
     <meta charset="UTF-8" />
     <title><Name> Window</title>
     <link rel="stylesheet" href="index.css" />
   </head>
   <body>
     <div id="app"></div>
     <script type="module" src="index.js"></script>
   </body>
   </html>
   ```

4. **Create `src/<name>/index.css`** with a minimal reset.

5. **Create `src/<name>/index.ts`**:
   ```typescript
   import { Electroview } from "electrobun/browser";
   import type { <Name>RPC } from "../shared/<name>-rpc";

   const rpc = Electroview.defineRPC<<Name>RPC>({
     maxRequestTime: 10000,
     handlers: {
       requests: {},  // TODO: add renderer-side request handlers
       messages: {},  // TODO: add renderer-side message handlers
     },
   });

   // TODO: implement renderer UI
   document.getElementById("app")!.textContent = "<Name> Window";
   ```

6. **Create `src/shared/<name>-rpc.ts`**:
   ```typescript
   import type { ElectrobunRPCSchema } from "electrobun/bun";

   export type <Name>RPC = ElectrobunRPCSchema<{
     requests: {
       // TODO: add requests (renderer asks bun, or bun asks renderer)
       // example: getData: { args: {}; response: string[] };
     };
     messages: {
       // TODO: add messages (fire-and-forget)
       // example: notify: { message: string };
     };
   }>;
   ```

7. **Add to `electrobun.config.ts`** — add `<name>: { entrypoint: "src/<name>/index.ts" }` to the `views` section.

8. **Add a BrowserWindow creation block to `src/bun/index.ts`**:
   ```typescript
   import { BrowserView } from "electrobun/bun";
   import type { <Name>RPC } from "../shared/<name>-rpc";

   // <Name> Window
   const <name>RPC = BrowserView.defineRPC<<Name>RPC>({
     maxRequestTime: 10000,
     handlers: {
       requests: {
         // TODO: add bun-side request handlers
       },
       messages: {
         // TODO: add bun-side message handlers
       },
     },
   });

   const <name>Win = new BrowserWindow({
     title: "<Name>",
     frame: { width: 800, height: 600 },
     url: "<name>://index.html",
     rpc: <name>RPC,
   });
   ```

9. **Tell the user** what was created and prompt them to fill in the `TODO` sections in both the schema and the handlers.
```

---

### Task 8: Write the RPC generation command

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/commands/electrobun-rpc.md`

- [ ] **Step 1: Write `commands/electrobun-rpc.md`**

```markdown
---
name: electrobun-rpc
description: Introspect existing Electrobun bun-process and renderer files to generate a complete type-safe RPC schema. Reads your code, infers all cross-process calls, and writes the defineElectrobunRPC + Electroview.defineRPC boilerplate.
---

Analyze the current codebase and generate or update the full Electrobun RPC schema.

## Steps

1. **Read all TypeScript files** in `src/bun/` and all renderer directories (any `src/*/index.ts` that imports from `electrobun/browser`).

2. **Identify cross-process calls** by scanning for these patterns:

   In bun-side files (outgoing calls to renderer):
   - `rpc?.send.<methodName>(` → message from bun to renderer
   - `rpc?.request.<methodName>(` → request from bun to renderer
   - `rpc.send.<methodName>(` → message from bun to renderer
   - `rpc.request.<methodName>(` → request from bun to renderer

   In renderer-side files (outgoing calls to bun):
   - `rpc.send.<methodName>(` → message from renderer to bun
   - `rpc.request.<methodName>(` → request from renderer to bun
   - `rpc?.send.<methodName>(` / `rpc?.request.<methodName>(`

   In handler definitions (already implemented):
   - `handlers.requests.<methodName>:` → already has a handler
   - `handlers.messages.<methodName>:` → already has a handler

3. **Infer argument and return types** from call sites where possible:
   - If a call site passes `{ id: string, title: string }`, record that as the args type
   - If `await rpc.request.foo({})` is assigned to a typed variable, use that type as response
   - If types cannot be inferred, use `unknown` as a placeholder with a `// TODO: type this` comment

4. **Identify which pair** (bun ↔ renderer) each call belongs to, based on which view/window the rpc is attached to.

5. **Generate or update the schema file** at `src/shared/<view>-rpc.ts` (or `src/shared/rpc.ts` if single-view):

   ```typescript
   import type { ElectrobunRPCSchema } from "electrobun/bun";

   export type <View>RPC = ElectrobunRPCSchema<{
     requests: {
       // ── Renderer → Bun ────────────────────────────────────────────────
       <methodName>: { args: <ArgsType>; response: <ResponseType> };
       // TODO: type this
       <untypedMethod>: { args: unknown; response: unknown };

       // ── Bun → Renderer ────────────────────────────────────────────────
       <rendererMethod>: { args: <ArgsType>; response: <ResponseType> };
     };
     messages: {
       // ── Bun → Renderer (fire-and-forget) ──────────────────────────────
       <messageName>: <PayloadType>;

       // ── Renderer → Bun (fire-and-forget) ──────────────────────────────
       <rendererMessage>: <PayloadType>;
     };
   }>;
   ```

6. **Generate updated handler stubs** for any calls that don't yet have handlers:

   In the bun-side file, add stubs to `BrowserView.defineRPC`:
   ```typescript
   // NEW — generated stub, implement me
   <methodName>: async (args) => {
     throw new Error("Not implemented: <methodName>");
   },
   ```

   In the renderer-side file, add stubs to `Electroview.defineRPC`:
   ```typescript
   // NEW — generated stub, implement me
   <methodName>: (args) => {
     throw new Error("Not implemented: <methodName>");
   },
   ```

7. **Show the user** a summary:
   - How many methods were found (requests + messages)
   - How many were already typed vs need `// TODO: type this`
   - How many new stubs were added
   - Where the schema file was written

8. **Ask if they want to proceed** with implementing any of the stub handlers now.
```

---

### Task 9: Write the WebGPU setup command

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/commands/electrobun-wgpu.md`

- [ ] **Step 1: Write `commands/electrobun-wgpu.md`**

```markdown
---
name: electrobun-wgpu
description: Set up a WebGPU rendering window in an Electrobun project. Creates a GpuWindow with WGPUView, a minimal WGSL pass-through shader, render loop, and KEEPALIVE array. Sets bundleWGPU: true in config. Usage: /electrobun-wgpu [name]
---

Add a WebGPU rendering window to the current Electrobun project.

## Steps

1. **Determine the window name** from the argument or ask the user (defaults to `gpu`).

2. **Read `electrobun.config.ts`** and check if `bundleWGPU` is set for each platform.
   - If not set or set to `false`, update the config to set `bundleWGPU: true` for mac, win, and linux.
   - Tell the user: "Set bundleWGPU: true in electrobun.config.ts — this is required for WebGPU to work."

3. **Create `src/bun/<name>.ts`** (or add to `src/bun/index.ts` if the project is small):

   ```typescript
   import { GpuWindow } from "electrobun/bun";

   // KEEPALIVE prevents Bun's GC from collecting FFI pointers mid-render
   const KEEPALIVE: unknown[] = [];

   // ── Window ────────────────────────────────────────────────────────────────
   const win = new GpuWindow({
     title: "<Name>",
     frame: { width: 800, height: 600 },
     centered: true,
   });

   // ── GPU Setup ─────────────────────────────────────────────────────────────
   const adapter = await navigator.gpu.requestAdapter();
   KEEPALIVE.push(adapter);
   if (!adapter) throw new Error("WebGPU adapter not found. Is bundleWGPU: true set in config?");

   const device = await adapter.requestDevice();
   KEEPALIVE.push(device);

   const context = win.getContext("webgpu")!;
   const format = navigator.gpu.getPreferredCanvasFormat();
   context.configure({ device, format });

   // ── Shader ────────────────────────────────────────────────────────────────
   const shaderCode = /* wgsl */`
   struct VertexOutput {
     @builtin(position) position: vec4f,
     @location(0) uv: vec2f,
   }

   @vertex
   fn vs_main(@builtin(vertex_index) idx: u32) -> VertexOutput {
     // Full-screen triangle
     var positions = array<vec2f, 3>(
       vec2f(-1.0, -1.0),
       vec2f( 3.0, -1.0),
       vec2f(-1.0,  3.0),
     );
     var out: VertexOutput;
     out.position = vec4f(positions[idx], 0.0, 1.0);
     out.uv = (positions[idx] + vec2f(1.0)) * 0.5;
     return out;
   }

   @fragment
   fn fs_main(in: VertexOutput) -> @location(0) vec4f {
     // TODO: replace with your shader logic
     return vec4f(in.uv, 0.5, 1.0);
   }
   `;

   const shaderModule = device.createShaderModule({ code: shaderCode });
   KEEPALIVE.push(shaderModule);

   // ── Pipeline ──────────────────────────────────────────────────────────────
   const pipeline = device.createRenderPipeline({
     layout: "auto",
     vertex: {
       module: shaderModule,
       entryPoint: "vs_main",
     },
     fragment: {
       module: shaderModule,
       entryPoint: "fs_main",
       targets: [{ format }],
     },
     primitive: { topology: "triangle-list" },
   });
   KEEPALIVE.push(pipeline);

   // ── Render Loop ───────────────────────────────────────────────────────────
   setInterval(() => {
     const commandEncoder = device.createCommandEncoder();
     KEEPALIVE.push(commandEncoder);

     const passEncoder = commandEncoder.beginRenderPass({
       colorAttachments: [{
         view: context.getCurrentTexture().createView(),
         clearValue: { r: 0.0, g: 0.0, b: 0.0, a: 1.0 },
         loadOp: "clear",
         storeOp: "store",
       }],
     });

     passEncoder.setPipeline(pipeline);
     passEncoder.draw(3);
     passEncoder.end();

     device.queue.submit([commandEncoder.finish()]);
   }, 16); // ~60fps
   ```

4. **Tell the user** what was created and point to the `// TODO` section in the fragment shader as the customization point.

5. **Remind the user**: run `bun run dev` and the black window with a UV gradient should appear. If it crashes immediately, confirm `bundleWGPU: true` is in the config.
```

---

### Task 10: Write the menu command

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/commands/electrobun-menu.md`

- [ ] **Step 1: Write `commands/electrobun-menu.md`**

```markdown
---
name: electrobun-menu
description: Generate an ApplicationMenu with File/Edit/View/Help items wired to application-menu-clicked events, plus optional Tray setup. Asks whether to forward events to a webview via RPC.
---

Add an ApplicationMenu (and optionally a Tray) to the current Electrobun project.

## Steps

1. **Ask the user:**
   - Do you want a Tray icon as well? (yes/no)
   - Should menu clicks be forwarded to the webview via RPC? (yes/no)
   - If yes to RPC: which BrowserView variable to target? (e.g. `win.webview`)

2. **Read `src/bun/index.ts`** to understand the existing structure and imports.

3. **Add to `src/bun/index.ts`** (or create `src/bun/menu.ts` if the file is large):

   ```typescript
   import { ApplicationMenu } from "electrobun/bun";
   // (add Tray import if requested)
   import { Tray } from "electrobun/bun";

   // ── Application Menu ──────────────────────────────────────────────────────
   ApplicationMenu.setMenu([
     {
       label: "File",
       submenu: [
         { label: "New",   action: "file-new",   accelerator: "CmdOrCtrl+N" },
         { label: "Open",  action: "file-open",  accelerator: "CmdOrCtrl+O" },
         { label: "Save",  action: "file-save",  accelerator: "CmdOrCtrl+S" },
         { type: "separator" },
         { label: "Quit",  role: "quit" },
       ],
     },
     {
       label: "Edit",
       submenu: [
         { label: "Undo",      role: "undo",      accelerator: "CmdOrCtrl+Z" },
         { label: "Redo",      role: "redo",      accelerator: "CmdOrCtrl+Shift+Z" },
         { type: "separator" },
         { label: "Cut",       role: "cut",       accelerator: "CmdOrCtrl+X" },
         { label: "Copy",      role: "copy",      accelerator: "CmdOrCtrl+C" },
         { label: "Paste",     role: "paste",     accelerator: "CmdOrCtrl+V" },
         { label: "Select All",role: "selectall", accelerator: "CmdOrCtrl+A" },
       ],
     },
     {
       label: "View",
       submenu: [
         { label: "Reload",          role: "reload",          accelerator: "CmdOrCtrl+R" },
         { label: "Toggle DevTools", role: "toggledevtools",  accelerator: "CmdOrCtrl+Option+I" },
         { type: "separator" },
         { label: "Actual Size",     role: "resetzoom" },
         { label: "Zoom In",         role: "zoomin",          accelerator: "CmdOrCtrl+=" },
         { label: "Zoom Out",        role: "zoomout",         accelerator: "CmdOrCtrl+-" },
       ],
     },
     {
       label: "Help",
       submenu: [
         { label: "About", action: "help-about" },
       ],
     },
   ]);

   // ── Menu Event Handler ────────────────────────────────────────────────────
   Electrobun.events.on("application-menu-clicked", (e) => {
     const { action, role } = e.data;
     console.log("Menu clicked:", action ?? role);

     // Handle built-in actions
     switch (action) {
       case "file-new":  /* TODO */; break;
       case "file-open": /* TODO */; break;
       case "file-save": /* TODO */; break;
       case "help-about": /* TODO */; break;
     }

     // Forward to renderer via RPC (if requested)
     <targetView>.rpc?.send.menuAction({ action: action ?? "", role: role ?? "" });
   });
   ```

   If Tray was requested, also add:
   ```typescript
   // ── Tray ──────────────────────────────────────────────────────────────────
   const tray = new Tray({
     icon: "assets/tray-icon.png",
     tooltip: "<App Name>",
   });

   tray.setMenu([
     { label: "Show",   action: "tray-show" },
     { label: "Hide",   action: "tray-hide" },
     { type: "separator" },
     { label: "Quit",   role: "quit" },
   ]);

   // Note: tray click events do NOT fire on Linux with AppIndicator
   ```

4. **If RPC forwarding was requested**, add `menuAction` to the RPC schema:
   - In the schema file: `menuAction: { action: string; role: string };` under `messages`
   - In the renderer: add a `menuAction` handler that dispatches the action to the UI

5. **Remind the user** about the Linux tray limitation if Tray was added.
```

---

### Task 11: Write the release command

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/commands/electrobun-release.md`

- [ ] **Step 1: Write `commands/electrobun-release.md`**

```markdown
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
   # List available signing identities
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
   - macOS: check that `build/*.app` exists and is code-signed
     ```bash
     codesign -dv --verbose=4 build/<AppName>.app
     spctl --assess --verbose build/<AppName>.app
     ```
   - Windows: check that `build/*.exe` exists

8. **Configure the Updater** — verify or add update check to `src/bun/index.ts`:
   ```typescript
   import { Updater } from "electrobun/bun";

   const updater = new Updater();
   // Check for updates on startup (non-blocking)
   updater.checkForUpdates().then(async (status) => {
     if (status.hasUpdate) {
       console.log("Update available:", status.version);
       await updater.downloadUpdate();
       await updater.applyUpdate();
     }
   }).catch(console.error);
   ```

9. **Summarize** what was done and what the user needs to do next (upload artifacts to `release.baseUrl`, bump version for next release, etc.).
```

---

## Chunk 4: Agents

### Task 12: Write the architect agent

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/agents/electrobun-architect.md`

- [ ] **Step 1: Write `agents/electrobun-architect.md`**

```markdown
---
description: Electrobun application architect. Designs multi-window desktop app architecture before code is written — produces window/view layouts, complete RPC flow diagrams, file structure recommendations, and electrobun.config.ts skeletons. Call this agent when planning a new Electrobun app or adding a significant new feature that requires multiple windows or views.
capabilities:
  - Design BrowserWindow and BrowserView layouts with sizing and positioning
  - Map RPC flows between all bun-process and renderer processes
  - Distinguish requests (need a response) from messages (fire-and-forget) for each call
  - Recommend file structure for multi-view projects
  - Generate electrobun.config.ts skeleton with correct views, platform flags, and renderer choice
  - Advise on CEF vs native renderer tradeoffs
  - Design data flow patterns across the process boundary
---

# Electrobun Architect

You are an Electrobun application architect. Your job is to design the structure of a desktop app **before any code is written**.

## Constraints You Must Design Around

1. **Each BrowserView is a separate OS process** — data does not share memory between views. All cross-view communication goes through the bun process (bun acts as broker).
2. **RPC is the only safe cross-process channel** — no shared globals, no window.opener, no postMessage between views.
3. **Native renderer limitations**: no browser extensions, limited devtools, no SharedArrayBuffer. CEF removes these limits but adds ~120MB.
4. **GPU windows are exclusive** — a GpuWindow cannot host a BrowserView. If you need both GPU rendering and a UI overlay, you need two separate windows.

## What to Produce

When given a description of an app, produce:

### 1. Window & View Layout

A table listing each window/view:
| Window/View | Type | Size | URL/Content | Purpose |
|---|---|---|---|---|
| mainWin | BrowserWindow | 1200×800 | mainview | Primary UI |
| sidebar | BrowserView | 300×800 | sidebar | Navigation |

### 2. RPC Flow Diagram

For each pair that communicates, list every call:
```
mainview → bun (requests):
  - getNotes(): Note[]
  - saveNote(id, title, content): { success: boolean }

bun → mainview (messages):
  - menuAction(action, role): void
  - noteUpdatedExternally(id): void

sidebar → bun (messages):
  - sidebarNavigation(path): void
```

Classify every call: is it a **request** (needs a response) or a **message** (fire-and-forget)?

### 3. File Structure

```
src/
├── bun/
│   ├── index.ts        # App entry, window creation
│   ├── db.ts           # Data layer
│   └── menu.ts         # ApplicationMenu + Tray
├── mainview/
│   ├── index.html
│   ├── index.ts
│   └── index.css
├── sidebar/
│   ├── index.html
│   ├── index.ts
│   └── index.css
└── shared/
    ├── mainview-rpc.ts
    └── sidebar-rpc.ts
```

### 4. electrobun.config.ts Skeleton

Write the full config skeleton with all views, platform flags, and recommended settings.

### 5. Platform Notes

Note any platform-specific behavior the app needs to handle (tray limitations, title bar styles, etc.).

## Process

1. Ask clarifying questions if the description is ambiguous (one at a time)
2. Produce the design in the sections above
3. Ask if they want to proceed — if yes, suggest running `/electrobun-init` followed by `/electrobun-window` for each additional view
```

---

### Task 13: Write the debugger agent

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/agents/electrobun-debugger.md`

- [ ] **Step 1: Write `agents/electrobun-debugger.md`**

```markdown
---
description: Electrobun debugger. Diagnoses build failures, RPC timeouts, webview rendering issues, input passthrough problems, and cross-platform bugs. Use this agent when something is broken and the cause isn't obvious.
capabilities:
  - Diagnose build errors (missing bundleWGPU, missing views entry, icon format issues)
  - Diagnose RPC failures (timeouts, schema mismatches, sandbox mode, initialization order)
  - Diagnose webview rendering issues (blank windows, toggle passthrough, z-order problems)
  - Diagnose WebGPU crashes (KEEPALIVE, swap chain, format mismatches)
  - Diagnose platform-specific bugs (Linux tray, Windows title bar, macOS notarization)
  - Read error messages and stack traces to identify root cause
---

# Electrobun Debugger

You are an Electrobun debugging specialist. Follow this diagnostic tree systematically.

## Step 1: Classify the Failure

Ask the user (or infer from the error): which category?

| # | Category | Symptoms |
|---|---|---|
| A | **Build error** | `electrobun build` fails, missing module, icon error |
| B | **Runtime crash** | App exits immediately, segfault, "module not found" |
| C | **RPC failure** | "RPC timeout", calls silently fail, type errors |
| D | **Webview blank** | Window opens but shows nothing |
| E | **Input/passthrough** | Clicks not registering, UI unresponsive |
| F | **WebGPU crash** | Black window, segfault after a few frames, invalid pointer |
| G | **Platform bug** | Works on macOS, broken on Linux/Windows |

## Diagnostic Trees

### A: Build Errors

1. Is the error "module not found" for a view?
   → Check `electrobun.config.ts` `views` section. Every renderer with a `.ts` entrypoint needs an entry there.

2. Is the error related to WGPU or native binaries?
   → Check `bundleWGPU: true` is set for the platform being built.

3. Is the error about CEF?
   → Check `bundleCEF: true` is set AND `renderer: "cef"` is used in the BrowserView.

4. Is the error about the app icon?
   → macOS requires `.icns`, Windows requires `.ico`. They can't be interchanged.

### B: Runtime Crashes

1. Crash immediately on launch?
   → Check if `bundleWGPU`/`bundleCEF` is set correctly for the current platform.
   → Check the entrypoint compiles: `bun src/bun/index.ts` directly.

2. Crash after a few seconds?
   → If WebGPU is involved: almost certainly a missing KEEPALIVE entry. Ask the user to audit every GPU object.

3. "Cannot find name X" at runtime?
   → Check import paths — `electrobun/bun` vs `electrobun/browser`. Bun-side code uses `electrobun/bun`; renderer code uses `electrobun/browser`.

### C: RPC Failures

1. "RPC timeout"?
   → `maxRequestTime` is too low. Increase to 30000 for file dialogs, 10000 for DB ops.

2. Calls silently return undefined?
   → Check that `sandbox: false` on the BrowserView. `sandbox: true` disables RPC.
   → Check that `rpc` is passed to the BrowserView constructor.
   → Check that `Electroview.defineRPC` is called BEFORE any `rpc.request.*` calls in the renderer.

3. TypeScript type errors on RPC calls?
   → Schema mismatch between bun side and renderer side. Both must import from the same shared type file. Run `/electrobun-rpc` to regenerate.

4. `rpc.send.*` works but `rpc.request.*` hangs?
   → The handler on the other side is missing or throws. Check both handler dictionaries.

### D: Webview Blank

1. Window opens but shows white/black/nothing?
   → Check that the URL is correct (local file: `view://index.html`, remote: full URL).
   → Check that the renderer entrypoint compiled without errors: look at build output.
   → Open devtools: `win.webview.openDevTools()` — add this temporarily and check console.

2. HTML loads but JavaScript doesn't run?
   → Check that `views` entry exists in `electrobun.config.ts` for this renderer.
   → Check that the `<script type="module">` tag points to the compiled output (`index.js`, not `index.ts`).

### E: Input/Passthrough

1. Clicks going through a webview to the one behind it?
   → Check `togglePassthrough` — if set to true, the view is click-through. Only set this on overlay views that should not capture input.

2. Webview not responding to clicks at all?
   → Check `toggleHidden` — if the view is hidden it won't receive input.
   → Check the `frame` position and size: a view sized to 0×0 is invisible and unclickable.

### F: WebGPU Crashes

1. Segfault or invalid pointer after a few frames?
   → **KEEPALIVE**. Every GPU object (adapter, device, pipeline, buffer, encoder, texture, sampler) must be in the KEEPALIVE array. Bun's GC collects unreferenced FFI objects.

2. Black window, no error?
   → Check `bundleWGPU: true` in config.
   → Verify `context.configure({ device, format })` is called after creating the surface.
   → Check the render loop is actually running: add `console.log` inside `setInterval`.

3. Distorted rendering after resize?
   → The swap chain must be reconfigured on resize. Listen for resize events and call `context.configure()` again.

### G: Platform-Specific Bugs

- **Linux tray click doesn't fire**: Known limitation with AppIndicator. Use tray menu items instead of click handlers.
- **Windows hiddenInset title bar**: Not supported. Use `titleBarStyle: "hidden"` instead.
- **macOS notarization fails**: Most common cause is hardened runtime without JIT entitlement. Add `com.apple.security.cs.allow-jit` to entitlements.plist.
- **Linux ApplicationMenu**: Renders in the app window, not the system menu bar. Layout may differ from macOS/Windows.

## Response Format

1. State the failure category you've identified
2. Walk through the relevant diagnostic tree
3. Give a ranked list of likely causes (most likely first)
4. Provide the exact fix for each cause
5. Tell the user how to verify the fix worked
```

---

## Chunk 5: Hook

### Task 14: Write the hook configuration

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/hooks/hooks.json`

- [ ] **Step 1: Write `hooks/hooks.json`**

```json
{
  "PreToolUse": [
    {
      "matcher": "Write|Edit",
      "hooks": [
        {
          "type": "command",
          "command": "bash $CLAUDE_PLUGIN_ROOT/hooks/scripts/validate-electrobun.sh",
          "timeout": 10
        }
      ]
    }
  ]
}
```

- [ ] **Step 2: Validate the JSON**

```bash
jq . ~/.claude/plugins/electrobun-dev/hooks/hooks.json
```
Expected: prints cleanly.

---

### Task 15: Write the validation script

**Files:**
- Create: `~/.claude/plugins/electrobun-dev/hooks/scripts/validate-electrobun.sh`

- [ ] **Step 1: Write `hooks/scripts/validate-electrobun.sh`**

```bash
#!/usr/bin/env bash
# electrobun-dev: Pre-write/edit validator
# Non-blocking — emits warnings to stderr, always exits 0

# Read tool input JSON from stdin
INPUT=$(cat)

FILE_PATH=$(echo "$INPUT" | jq -r '.tool_input.file_path // empty' 2>/dev/null)
CONTENT=$(echo "$INPUT" | jq -r '.tool_input.content // (.tool_input.new_string // empty)' 2>/dev/null)

# Only check TypeScript/JavaScript source files
if [[ -z "$CONTENT" ]] || [[ -z "$FILE_PATH" ]]; then
  exit 0
fi

if [[ ! "$FILE_PATH" =~ \.(ts|tsx|js|jsx)$ ]]; then
  exit 0
fi

WARNINGS=()

# ── 1. WebGPU code without bundleWGPU ──────────────────────────────────────
if echo "$CONTENT" | grep -qiE "(GpuWindow|WGPUView|wgpu)"; then
  if [[ "$FILE_PATH" != *"electrobun.config"* ]]; then
    # Walk up from file to find electrobun.config.ts
    DIR=$(dirname "$FILE_PATH")
    CONFIG_FILE=""
    while [[ "$DIR" != "/" && "$DIR" != "." ]]; do
      if [[ -f "$DIR/electrobun.config.ts" ]]; then
        CONFIG_FILE="$DIR/electrobun.config.ts"
        break
      fi
      DIR=$(dirname "$DIR")
    done

    if [[ -n "$CONFIG_FILE" ]]; then
      if ! grep -q "bundleWGPU.*true" "$CONFIG_FILE" 2>/dev/null; then
        WARNINGS+=("⚠️  WebGPU code detected but bundleWGPU: true is missing in electrobun.config.ts — the app will fail to load the WGPU native library at runtime")
      fi
    else
      WARNINGS+=("⚠️  WebGPU code detected — ensure bundleWGPU: true is set in electrobun.config.ts for each target platform")
    fi
  fi
fi

# ── 2. CEF renderer without bundleCEF in config ────────────────────────────
if echo "$CONTENT" | grep -qE "renderer[[:space:]]*:[[:space:]]*['\"]cef['\"]"; then
  DIR=$(dirname "$FILE_PATH")
  CEF_CONFIG=""
  while [[ "$DIR" != "/" && "$DIR" != "." ]]; do
    if [[ -f "$DIR/electrobun.config.ts" ]]; then
      CEF_CONFIG="$DIR/electrobun.config.ts"
      break
    fi
    DIR=$(dirname "$DIR")
  done
  if [[ -n "$CEF_CONFIG" ]]; then
    if ! grep -q "bundleCEF.*true" "$CEF_CONFIG" 2>/dev/null; then
      WARNINGS+=("⚠️  CEF renderer detected but bundleCEF: true is missing in electrobun.config.ts — app will fail to load Chromium at runtime (adds ~120MB to bundle size)")
    fi
  else
    WARNINGS+=("⚠️  CEF renderer detected — ensure bundleCEF: true is set in electrobun.config.ts for each target platform (adds ~120MB to bundle size)")
  fi
fi

# ── 3. defineRPC without maxRequestTime ────────────────────────────────────
if echo "$CONTENT" | grep -qE "(defineRPC|defineElectrobunRPC)[[:space:]]*\("; then
  if ! echo "$CONTENT" | grep -q "maxRequestTime"; then
    WARNINGS+=("⚠️  RPC definition missing maxRequestTime — long operations (file dialogs, DB writes) will hit the default timeout. Set maxRequestTime: 10000 or higher for non-trivial handlers")
  fi
fi

# ── 4. FFI pointers in GPU code without KEEPALIVE ──────────────────────────
if echo "$CONTENT" | grep -qE "(GpuWindow|WGPUView|requestAdapter|requestDevice|createRenderPipeline|createBuffer)"; then
  if ! echo "$CONTENT" | grep -q "KEEPALIVE"; then
    WARNINGS+=("⚠️  WebGPU objects without KEEPALIVE array — Bun's GC will collect FFI pointers mid-render causing segfaults. Add: const KEEPALIVE: unknown[] = [] and push every GPU object into it")
  fi
fi

# ── Emit warnings ───────────────────────────────────────────────────────────
if [[ ${#WARNINGS[@]} -gt 0 ]]; then
  echo "" >&2
  echo "┌─ electrobun-dev ─────────────────────────────────────────────" >&2
  for W in "${WARNINGS[@]}"; do
    echo "│ $W" >&2
  done
  echo "└───────────────────────────────────────────────────────────────" >&2
  echo "" >&2
fi

exit 0
```

- [ ] **Step 2: Make the script executable**

```bash
chmod +x ~/.claude/plugins/electrobun-dev/hooks/scripts/validate-electrobun.sh
```

- [ ] **Step 3: Validate bash syntax**

```bash
bash -n ~/.claude/plugins/electrobun-dev/hooks/scripts/validate-electrobun.sh
```
Expected: no output (clean syntax).

- [ ] **Step 4: Smoke test the script with a WebGPU file**

```bash
echo '{"tool_name":"Write","tool_input":{"file_path":"/tmp/test.ts","content":"const win = new GpuWindow({ width: 800 });"}}' \
  | bash ~/.claude/plugins/electrobun-dev/hooks/scripts/validate-electrobun.sh
```
Expected: Warning about WebGPU and KEEPALIVE printed to stderr. Exit 0.

---

### Task 16: Final validation with plugin-validator

- [ ] **Step 1: Run the plugin-validator agent**

Invoke the `plugin-dev:plugin-validator` agent, pointing it at `~/.claude/plugins/electrobun-dev/`. It should validate:
- `plugin.json` structure and required fields
- All command files have valid frontmatter
- All agent files have valid frontmatter
- All skill directories contain `SKILL.md` with valid frontmatter
- `hooks/hooks.json` references a script that exists

- [ ] **Step 2: Fix any issues found by the validator**

Address all errors. Warnings are acceptable if they're known limitations.

- [ ] **Step 3: Commit**

```bash
cd ~/.claude/plugins/electrobun-dev
git init
git add .
git commit -m "feat: add electrobun-dev plugin

Skills: electrobun, electrobun-rpc, electrobun-webgpu, electrobun-build
Commands: init, window, rpc, wgpu, menu, release
Agents: architect, debugger
Hook: PreToolUse Electrobun config/pattern validator"
```
