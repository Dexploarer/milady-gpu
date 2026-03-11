# plugin-electrobun-dev

A [Claude Code](https://claude.ai/code) plugin for building Electrobun desktop apps — from first `electrobun init` through production release.

Electrobun is a cross-platform desktop framework (macOS/Windows/Linux) that uses Bun as runtime and a native system webview (or CEF) as renderer.

---

## What's Inside

### 15 Skills

Skills load domain knowledge into Claude's context. Invoke them with the `Skill` tool or reference them in agent dispatch.

| Skill | Purpose |
|-------|---------|
| `electrobun` | Core patterns — BrowserWindow, BrowserView, events, app lifecycle, menus, tray |
| `electrobun-config` | `electrobun.config.ts` field reference and validation rules |
| `electrobun-rpc` | RPC wiring between bun process and renderer |
| `electrobun-platform` | Cross-platform builds, artifact naming, OS-specific behavior |
| `electrobun-init` | 19 starter templates and scaffolding guide |
| `electrobun-dev` | Dev server, hot reload, devtools |
| `electrobun-workflow` | Pipeline status map (INIT → DEV → BUILD → RELEASE) |
| `electrobun-build` | Build system, code signing, CI/CD matrix for all 5 platforms |
| `electrobun-release` | Auto-update configuration, Updater API, artifact publishing |
| `electrobun-webgpu` | Native GPU rendering, WGSL shaders, Dawn/WGPU buffer management |
| `electrobun-testing` | `defineTest()` framework, Kitchen Sink test patterns |
| `electrobun-kitchen-sink` | Kitchen Sink reference app, UI map, test automation |
| `electrobun-sdlc` | Full 8-stage SDLC pipeline overview and stage handoffs |
| `electrobun-teams` | Agent Teams orchestration (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) |
| `electrobun-milady` | milady-ai/milady integration — trust scoring, agent review, milady CI secrets |
| `electrobun-guide` | Master index — decision tree, full inventory, when-to-use reference |

### 13 Commands

Slash commands invoked as `/electrobun-<name>`.

| Command | What it does |
|---------|-------------|
| `/electrobun-init` | Interactive template picker (19 templates) → scaffolds new project |
| `/electrobun-setup` | Run after init — creates CI/CD, tests, docs, CLAUDE.md, optional AI review + trust scoring |
| `/electrobun-workflow` | Shows pipeline stage, what's done, what's next |
| `/electrobun-feature <desc>` | Quick 2-agent build: UI agent → RPC Contract Handoff → backend agent |
| `/electrobun-sdlc <desc>` | Full 8-stage pipeline: researcher → architect → planner → dev squad → QA → tests → alignment → docs |
| `/electrobun-rpc` | RPC setup wizard: schema authoring, handler wiring, shared type file |
| `/electrobun-window` | Window creation wizard: BrowserWindow options, sizing, positioning, title bar |
| `/electrobun-menu` | Application menu / context menu / tray menu builder |
| `/electrobun-wgpu` | WebGPU feature builder: shader setup, render loop, buffer management |
| `/electrobun-test` | 5-option menu: run by name, run all, write new test, manifest ops, coverage |
| `/electrobun-align` | Alignment wizard: scans repo, backs up files, prompts per change, repairs drift |
| `/electrobun-build` | Guided build: env picker, signing prereqs, version confirm, artifact summary |
| `/electrobun-release` | Full release: version bump, signing check, channel select, build, upload, verify |

### 11 Agents

Agents are dispatched by commands or orchestrators.

**SDLC Pipeline (Stages 1–8)**

| Stage | Agent | Role |
|-------|-------|------|
| 1 | `electrobun-researcher` | Codebase scan, API surface mapping, risk identification |
| 2 | `electrobun-architect` | Architecture spec, blast radius analysis, RPC flow, config skeleton |
| 3 | `electrobun-planner` | Atomic TDD task plans, agent assignments, sanity checks |
| 4a | `electrobun-ui-agent` | Renderer files, HTML/CSS, Electroview RPC, produces RPC Contract Handoff |
| 4b | `electrobun-backend-agent` | Bun-side wiring, BrowserView.defineRPC, config updates |
| 5 | `electrobun-qa-engineer` | Spec compliance audit, blast radius check, BLOCKER/IMPORTANT/MINOR report |
| 6 | `electrobun-test-writer` | Golden-outcome tests (Kitchen Sink defineTest or vitest) |
| 7 | `electrobun-alignment-agent` | Fix QA findings in priority order, cleanup, blast radius correction |
| 8 | `electrobun-docs-agent` | Mintlify docs, regression tests, mark plan COMPLETE |

**Specialist Agents**

| Agent | Use When |
|-------|---------|
| `electrobun-debugger` | Something is broken — build failure, RPC timeout, blank window, GPU crash |
| `electrobun-kitchen-agent` | Automating Kitchen Sink tests, reading UI map, running test suites |

---

## Quick Start

```
What do I want to do?
│
├─ Start a brand-new Electrobun app
│    └─ /electrobun-init → then /electrobun-setup
│
├─ Build a complete new feature (with tests + docs)
│    └─ /electrobun-sdlc <feature description>
│
├─ Add a quick feature to an existing view
│    └─ /electrobun-feature <description>
│
├─ Wire up RPC between bun and renderer
│    └─ /electrobun-rpc
│
├─ Build and package the app
│    └─ /electrobun-build
│
├─ Publish a release
│    └─ /electrobun-release
│
├─ Something is broken
│    └─ electrobun-debugger agent
│
└─ Align an existing project with plugin standards
     └─ /electrobun-align
```

---

## Installation

Clone directly to your plugins folder:

```bash
git clone https://github.com/Dexploarer/plugin-electrobun-dev ~/.claude/plugins/electrobun-dev
```

Then restart Claude Code. The plugin's skills, commands, and agents will be available immediately.

---

## Workflow Map

```
NEW PROJECT
    /electrobun-init       → Pick template, scaffold
    /electrobun-setup      → CI/CD, docs, tests, CLAUDE.md

DEVELOPMENT
    /electrobun-feature    → Quick feature (2-agent)
    /electrobun-sdlc       → Full feature (8-stage pipeline)
    /electrobun-rpc        → Wire RPC
    /electrobun-window     → Add window
    /electrobun-menu       → Add menus
    /electrobun-wgpu       → Add GPU rendering

QUALITY
    /electrobun-test       → Run/write tests
    /electrobun-align      → Repair drift

BUILD & RELEASE
    /electrobun-build      → Package for distribution
    /electrobun-release    → Publish + update server

STATUS CHECK
    /electrobun-workflow   → Where am I in the pipeline?
```

---

## milady Integration

This plugin is built to work with the [milady-ai/milady](https://github.com/milady-ai/milady) agents-only contribution model. The `electrobun-milady` skill covers:

- Trust scoring tiers and how to build trust efficiently
- milady-specific CI secret names (different from standard Electrobun names)
- Agent review pipeline — machine-parsed verdict format, follow-up issue auto-creation
- Per-stage SDLC adjustments for milady-targeted development (Biome, no `any`, 500 LOC limit, coverage thresholds)

---

## Plugin Structure

```
plugin-electrobun-dev/
├── skills/
│   ├── electrobun/             ← Core patterns
│   ├── electrobun-build/       ← Build system + CI/CD
│   ├── electrobun-config/      ← Config reference
│   ├── electrobun-dev/         ← Dev server
│   ├── electrobun-guide/       ← Master index
│   ├── electrobun-init/        ← Templates
│   ├── electrobun-kitchen-sink/ ← Kitchen Sink reference
│   ├── electrobun-milady/      ← milady integration
│   ├── electrobun-platform/    ← Cross-platform
│   ├── electrobun-release/     ← Release + Updater
│   ├── electrobun-rpc/         ← RPC system
│   ├── electrobun-sdlc/        ← 8-stage pipeline
│   ├── electrobun-teams/       ← Agent teams
│   ├── electrobun-testing/     ← defineTest framework
│   ├── electrobun-webgpu/      ← GPU rendering
│   └── electrobun-workflow/    ← Pipeline map
├── commands/
│   ├── electrobun-align.md
│   ├── electrobun-build.md
│   ├── electrobun-feature.md
│   ├── electrobun-init.md
│   ├── electrobun-menu.md
│   ├── electrobun-release.md
│   ├── electrobun-rpc.md
│   ├── electrobun-sdlc.md
│   ├── electrobun-setup.md
│   ├── electrobun-test.md
│   ├── electrobun-wgpu.md
│   ├── electrobun-window.md
│   └── electrobun-workflow.md
└── agents/
    ├── electrobun-alignment-agent.md
    ├── electrobun-architect.md
    ├── electrobun-backend-agent.md
    ├── electrobun-debugger.md
    ├── electrobun-docs-agent.md
    ├── electrobun-kitchen-agent.md
    ├── electrobun-planner.md
    ├── electrobun-qa-engineer.md
    ├── electrobun-researcher.md
    ├── electrobun-test-writer.md
    └── electrobun-ui-agent.md
```
