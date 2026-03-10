# Design: electrobun-dev Claude Code Plugin

**Date:** 2026-03-10
**Status:** Approved

## Overview

A full-spectrum Claude Code plugin for Electrobun development. Covers scaffolding, ongoing development patterns, architecture design, and debugging — activated both automatically (skills, hook) and on demand (commands, agents).

## Plugin Identity

- **Name:** `electrobun-dev`
- **Location:** `~/.claude/plugins/electrobun-dev/`

## Architecture

```
electrobun-dev/
├── .claude-plugin/
│   └── plugin.json
├── commands/
│   ├── electrobun-init.md
│   ├── electrobun-window.md
│   ├── electrobun-rpc.md
│   ├── electrobun-wgpu.md
│   ├── electrobun-menu.md
│   └── electrobun-release.md
├── agents/
│   ├── electrobun-architect.md
│   └── electrobun-debugger.md
├── skills/
│   ├── electrobun/SKILL.md
│   ├── electrobun-rpc/SKILL.md
│   ├── electrobun-webgpu/SKILL.md
│   └── electrobun-build/SKILL.md
└── hooks/
    ├── hooks.json
    └── scripts/
        └── validate-electrobun.sh
```

## Components

### Skills (4)

| Skill | Trigger Context | Knowledge Domain |
|-------|----------------|-----------------|
| `electrobun` | Any Electrobun project | BrowserWindow/BrowserView options, lifecycle, events API, `exitOnLastWindowClosed`, `urlSchemes`, `navigationRules`, platform quirks |
| `electrobun-rpc` | RPC code (defineElectrobunRPC, Electroview, schema types) | `ElectrobunRPCSchema` anatomy, bun-side vs renderer-side patterns, `maxRequestTime` guidance, WebSocket transport, AES-GCM encryption |
| `electrobun-webgpu` | WebGPU, WGSL, GpuWindow, WGPUView, GPU buffers | `GpuWindow`/`WGPUView` setup, `bundleWGPU: true` requirement, KEEPALIVE pattern, DataView struct serialization, render loop, FFI pointer lifetime |
| `electrobun-build` | `electrobun.config.ts`, build/release tasks | Full `ElectrobunConfig` schema, platform flags, code signing/notarization, `release.baseUrl` + `generatePatch`, `scripts` hooks, CEF vs native |

### Commands (6)

**`/electrobun-init`**
Scaffolds a new project. Prompts for app name, identifier, and type (webview / WebGPU / both). Writes `package.json`, `electrobun.config.ts`, `tsconfig.json`, `src/bun/index.ts`, and `src/mainview/`. Runs `bun install`.

**`/electrobun-window [name]`**
Generates a matched BrowserWindow creation block + corresponding renderer entry point, with a starter RPC schema typed on both sides.

**`/electrobun-rpc`**
Introspects existing `src/bun/` and renderer files. Identifies all cross-process calls. Generates a complete `ElectrobunRPCSchema` type + both `defineElectrobunRPC` and `Electroview.defineRPC` call sites with stubs for unimplemented handlers.

**`/electrobun-wgpu [name]`**
Generates `GpuWindow` + `WGPUView` setup with a minimal WGSL pass-through shader, render loop, KEEPALIVE array. Sets `bundleWGPU: true` in config if missing.

**`/electrobun-menu`**
Generates `ApplicationMenu` (File/Edit/View/Help) wired to `application-menu-clicked` events, plus optional Tray setup. Asks whether to forward events to a webview via RPC.

**`/electrobun-release`**
Guided workflow: verify `release` config section → check signing identity → build → notarize → configure `Updater` with `baseUrl` + `generatePatch`.

### Agents (2)

**`electrobun-architect`**
Pre-code design agent. Given a description of the app, produces:
- Window/view layout (BrowserWindows, BrowserViews, sizing)
- RPC flow diagram (requests vs messages, direction)
- File structure recommendation
- `electrobun.config.ts` skeleton with correct `views` entries
- Platform flags (CEF vs native, WebGPU if needed)

Constraint-aware: knows each BrowserView is a separate renderer process.

**`electrobun-debugger`**
Structured diagnostic agent for broken things. Failure categories: build error / RPC timeout / webview not rendering / input passthrough / platform bug. Knows common culprits: missing bundle flags, `maxRequestTime` too low, KEEPALIVE GC, `togglePassthrough` misconfiguration, Linux tray limitations.

### Hook (1)

**PreToolUse on Write|Edit** — runs `validate-electrobun.sh` (10s timeout, non-blocking warnings):

1. `GpuWindow`/`WGPUView`/`wgpu` in file → check `bundleWGPU: true` in config
2. `renderer: 'cef'` → check `bundleCEF: true` in config
3. `defineRPC`/`defineElectrobunRPC` without `maxRequestTime` → warn about timeout defaults
4. FFI pointer variables in GPU file without `KEEPALIVE` reference → warn about GC invalidation

## Key Design Decisions

- **Non-blocking hook warnings** — hook emits stderr warnings, never blocks writes
- **RPC command is introspection-first** — reads existing code rather than generating from scratch to avoid schema drift
- **Skills are narrow** — four focused skills beat one monolithic one; they activate only when relevant context is present
- **Agents are phase-gated** — architect is pre-code, debugger is post-breakage; they don't overlap
