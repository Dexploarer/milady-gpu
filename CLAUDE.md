# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bun start              # Start dev server (electrobun dev)
bun run dev            # Dev with file watching (electrobun dev --watch)
bun run build:canary   # Build for canary environment
```

No test or lint commands are configured.

## Architecture

Single-file TypeScript application at `src/bun/index.ts` (~527 lines). Uses Electrobun's native WebGPU bindings to create a GPU window and render a real-time ray-marching fragment shader.

**Data flow:**
1. `electrobun dev` launches the Bun process
2. The app initializes a 640×640 native GPU window via Electrobun's WGPU bindings
3. A WGSL shader (lines ~319–402) is compiled once at startup into a render pipeline
4. A `setInterval(renderFrame, 16)` loop updates a vertex buffer with time/resolution/mouse uniforms and submits GPU commands each frame
5. Mouse position controls the ray-march camera; clicks toggle quality (32 vs 64 iterations)

**Key patterns:**
- All WebGPU objects are managed via FFI pointers. A `KEEPALIVE` array stores references to prevent Bun's GC from collecting them.
- GPU struct serialization uses `DataView` for manual byte-level buffer writes.
- `electrobun.config.ts` controls the build — `bundleWGPU: true` is required; `bundleCEF: false` since there's no web view.

## Configuration

`electrobun.config.ts` — app identity (`wgpu.electrobun.dev`) and platform build flags. The entrypoint is `src/bun/index.ts`.
