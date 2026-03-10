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
   if (!adapter) throw new Error("No GPU adapter found");
   KEEPALIVE.push(adapter);

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
   import { Electroview } from "electrobun/view";

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
