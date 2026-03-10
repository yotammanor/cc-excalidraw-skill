# Embedding Excalidraw in Standalone HTML (No Build System)

This guide covers how to embed interactive Excalidraw diagrams in plain HTML files served statically — no Webpack, Vite, or any bundler required. All dependencies are loaded from CDN at runtime.

## Why This Is Tricky

Excalidraw is a React component library. It expects a React 18 environment, JSX runtime, and its own CSS/assets. In a bundled app these are handled by the build tool. In a standalone HTML file you must wire them up manually via `<script type="importmap">` and ESM CDN imports.

Key pitfalls discovered through trial and error:

1. **React must be shared, not duplicated.** Excalidraw imports `react` internally. If you load React from a different CDN URL than Excalidraw expects, you get two React instances and hooks break. The solution is `?external=react,react-dom` on the Excalidraw import, combined with an `importmap` that maps the bare `react` specifier to a single CDN URL.
2. **The asset path must be set globally.** Excalidraw loads fonts and other assets at runtime. Without `window.EXCALIDRAW_ASSET_PATH`, it tries relative paths that 404 on a static server.
3. **The CSS must be loaded separately.** The JS bundle does not inject its own styles. You need a `<link rel="stylesheet">` pointing to the Excalidraw CSS.
4. **`convertToExcalidrawElements` is required.** Raw element JSON (the format documented in SKILL.md) is "skeleton" data. The library's `convertToExcalidrawElements()` function hydrates missing fields (dimensions, version nonces, etc.) so the renderer doesn't crash or produce invisible elements.
5. **Initial zoom/scroll needs explicit handling.** `scrollToContent: true` in `initialData.appState` is not always sufficient. A reliable pattern is to also call `api.scrollToContent(api.getSceneElements(), { fitToContent: true })` via `useEffect` after the Excalidraw API becomes available.
6. **The mount container needs explicit height.** Excalidraw fills its parent via `width:100%; height:100%`. If the parent has no explicit height, the canvas collapses to 0px.

## Tested Working Versions (as of March 2026)

| Package | Version | CDN URL |
|---------|---------|---------|
| `@excalidraw/excalidraw` | `0.18.0` | `https://esm.sh/@excalidraw/excalidraw@0.18.0` |
| `react` | `18.3.1` | `https://esm.sh/react@18.3.1` |
| `react-dom` | `18.3.1` | `https://esm.sh/react-dom@18.3.1` |

## Minimal Complete Example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Excalidraw Standalone</title>

  <!-- 1. Excalidraw CSS (must be loaded separately) -->
  <link rel="stylesheet" href="https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/dev/index.css" />

  <!-- 2. Asset path for fonts/icons loaded at runtime -->
  <script>window.EXCALIDRAW_ASSET_PATH = "https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/prod/";</script>

  <!-- 3. Import map: single source of truth for React so Excalidraw shares the same instance -->
  <script type="importmap">
  {
    "imports": {
      "react": "https://esm.sh/react@18.3.1",
      "react/jsx-runtime": "https://esm.sh/react@18.3.1/jsx-runtime",
      "react-dom": "https://esm.sh/react-dom@18.3.1"
    }
  }
  </script>
</head>
<body>

  <!-- 4. Mount point with EXPLICIT HEIGHT (critical — Excalidraw fills its parent) -->
  <div id="excalidraw-mount" style="height: 700px; border: 1px solid #ccc; overflow: hidden; background: #fff;">
    <div id="excalidraw-fallback" style="display:flex;align-items:center;justify-content:center;height:100%;color:#888;">
      Loading diagram…
    </div>
  </div>

  <!-- 5. Initialization script — must be type="module" -->
  <script type="module">
    // ?external=react,react-dom tells esm.sh NOT to bundle its own React copy;
    // instead it uses the bare "react" import which our importmap resolves.
    import * as ExcalidrawLib from "https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/dev/index.js?external=react,react-dom";
    import React from "react";
    import ReactDOM from "react-dom";

    const h = React.createElement;

    // Define elements using the Excalidraw JSON format (see SKILL.md).
    // Only the fields you care about are needed — convertToExcalidrawElements fills the rest.
    const elements = [
      {
        id: "rect-1", type: "rectangle",
        x: 100, y: 100, width: 200, height: 100,
        backgroundColor: "#a5d8ff", strokeColor: "#1971c2",
        fillStyle: "solid", roughness: 0, strokeWidth: 1.5,
        roundness: { type: 3 },
      },
      {
        id: "text-1", type: "text",
        x: 130, y: 135, text: "Hello Excalidraw",
        fontSize: 16, fontFamily: 2,
        strokeColor: "#1e1e1e",
      },
    ];

    const App = () => {
      const [api, setApi] = React.useState(null);

      // Fit diagram to viewport once the API is ready
      React.useEffect(() => {
        if (!api) return;
        setTimeout(() => {
          api.scrollToContent(api.getSceneElements(), { fitToContent: true });
        }, 200);
      }, [api]);

      return h("div", { style: { width: "100%", height: "100%" } },
        h(ExcalidrawLib.Excalidraw, {
          excalidrawAPI: (a) => setApi(a),
          initialData: {
            // convertToExcalidrawElements hydrates skeleton elements into full Excalidraw objects
            elements: ExcalidrawLib.convertToExcalidrawElements(elements),
            appState: {
              viewBackgroundColor: "#ffffff",
              gridSize: null,
            },
            scrollToContent: true,
          },
          // Read-only viewing mode — set to false if you want editing
          viewModeEnabled: true,
          zenModeEnabled: true,
          UIOptions: {
            canvasActions: {
              changeViewBackgroundColor: false,
              clearCanvas: false,
              export: false,
              loadScene: false,
              saveToActiveFile: false,
              toggleTheme: false,
            },
          },
        })
      );
    };

    // Mount
    const mount = document.getElementById("excalidraw-mount");
    if (mount) {
      const fallback = document.getElementById("excalidraw-fallback");
      if (fallback) fallback.remove();
      const root = ReactDOM.createRoot(mount);
      root.render(h(App));
    }
  </script>

</body>
</html>
```

## Step-by-Step Breakdown

### Step 1: `<head>` — CSS, Asset Path, Import Map

Three things go in `<head>`, in this order:

```html
<!-- CSS — the JS bundle does NOT inject styles -->
<link rel="stylesheet" href="https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/dev/index.css" />

<!-- Global asset path for fonts/icons loaded at runtime -->
<script>window.EXCALIDRAW_ASSET_PATH = "https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/prod/";</script>

<!-- Import map — the ONLY place React URLs should appear -->
<script type="importmap">
{
  "imports": {
    "react": "https://esm.sh/react@18.3.1",
    "react/jsx-runtime": "https://esm.sh/react@18.3.1/jsx-runtime",
    "react-dom": "https://esm.sh/react-dom@18.3.1"
  }
}
</script>
```

The `react/jsx-runtime` entry is needed because Excalidraw's internals import it. Without it you get a module resolution error.

### Step 2: Mount Container with Explicit Height

```html
<div id="excalidraw-mount" style="height: 700px; overflow: hidden; background: #fff;">
  <div id="excalidraw-fallback">Loading diagram…</div>
</div>
```

The fallback div is removed by the script once Excalidraw renders. Without an explicit `height` on the mount container, the Excalidraw canvas renders at 0px tall and is invisible.

### Step 3: `<script type="module">` — Import with `?external=react,react-dom`

```js
import * as ExcalidrawLib from "https://esm.sh/@excalidraw/excalidraw@0.18.0/dist/dev/index.js?external=react,react-dom";
import React from "react";        // resolved via importmap
import ReactDOM from "react-dom";  // resolved via importmap
```

The `?external=react,react-dom` query parameter is **critical**. It tells esm.sh to leave `import "react"` as a bare specifier rather than rewriting it to esm.sh's own bundled React URL. The browser's import map then resolves the bare specifier to our single shared React instance.

Without this, Excalidraw gets its own React copy, hooks fail with "Invalid hook call," and the component never renders.

### Step 4: Define Elements and Render

Use the element JSON format from SKILL.md. You can use shorthand — only include the fields you need. Then call `ExcalidrawLib.convertToExcalidrawElements(elements)` to hydrate them:

```js
const elements = [
  { id: "r1", type: "rectangle", x: 0, y: 0, width: 200, height: 100,
    backgroundColor: "#a5d8ff", strokeColor: "#1971c2",
    fillStyle: "solid", roughness: 0, roundness: { type: 3 } },
];

// In the Excalidraw component:
initialData: {
  elements: ExcalidrawLib.convertToExcalidrawElements(elements),
  appState: { viewBackgroundColor: "#ffffff" },
  scrollToContent: true,
}
```

### Step 5: Fit to Viewport

`scrollToContent: true` in `initialData` helps but is not always reliable on first render. The bulletproof pattern is:

```js
const [api, setApi] = React.useState(null);

React.useEffect(() => {
  if (!api) return;
  setTimeout(() => {
    api.scrollToContent(api.getSceneElements(), { fitToContent: true });
  }, 200);
}, [api]);

// Pass excalidrawAPI callback:
h(ExcalidrawLib.Excalidraw, {
  excalidrawAPI: (a) => setApi(a),
  // ...
})
```

The 200ms timeout gives the canvas time to measure its container before computing the fit.

## Read-Only vs Editable Mode

| Prop | Effect |
|------|--------|
| `viewModeEnabled: true` | Disables all editing tools; user can only pan/zoom |
| `zenModeEnabled: true` | Hides the side toolbar and menus |
| `UIOptions.canvasActions.*: false` | Hides specific menu items (export, clear, theme, etc.) |

For documentation/review pages, set all three. For collaborative editing, omit them.

## Using `React.createElement` Instead of JSX

Since there is no JSX transpiler in a standalone HTML file, use `React.createElement` directly. Alias it for readability:

```js
const h = React.createElement;

// JSX equivalent: <div style={{width:"100%"}}><Excalidraw ... /></div>
h("div", { style: { width: "100%", height: "100%" } },
  h(ExcalidrawLib.Excalidraw, { /* props */ })
);
```

## Loading an Existing `.excalidraw` File

If you have an `.excalidraw` JSON file, fetch it and pass its `elements` through `convertToExcalidrawElements`:

```js
const response = await fetch("./my-diagram.excalidraw");
const data = await response.json();

// In the Excalidraw component:
initialData: {
  elements: ExcalidrawLib.convertToExcalidrawElements(data.elements),
  appState: data.appState || { viewBackgroundColor: "#ffffff" },
  scrollToContent: true,
}
```

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| "Invalid hook call" error | Two React instances | Add `?external=react,react-dom` to the Excalidraw import URL |
| Canvas renders at 0px height | Mount container has no explicit height | Set `height: 700px` (or any fixed value) on the mount `<div>` |
| Fonts render as squares/fallback | Missing asset path | Set `window.EXCALIDRAW_ASSET_PATH` before any module loads |
| No styles / unstyled UI | Missing CSS link | Add the `<link rel="stylesheet">` for Excalidraw's CSS |
| Diagram loads but is off-screen or tiny | `scrollToContent` not working alone | Use `api.scrollToContent(api.getSceneElements(), { fitToContent: true })` in a `useEffect` with a small delay |
| Elements are invisible despite valid JSON | Raw element JSON missing hydrated fields | Wrap elements in `ExcalidrawLib.convertToExcalidrawElements()` |
| Module resolution error for `react/jsx-runtime` | Import map missing the jsx-runtime entry | Add `"react/jsx-runtime": "https://esm.sh/react@18.3.1/jsx-runtime"` to the import map |
| "Loading diagram..." never disappears | Script error prevents mount | Check browser console; most likely one of the above issues |
