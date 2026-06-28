# Duplicate Scene — Feature Architecture

> How the "duplicate scene" feature fits into the codebase. Written as an
> onboarding walkthrough: what changed, in which layer, and why.

## TL;DR

The duplicate feature deliberately mirrors the **add/delete scene** pattern
that's already in place — three layers, one new endpoint, one new button. No
data model or schema changed.

## The three-layer flow

Scene mutation in this codebase is always:

**HUD button → client handler → dev save-server endpoint → filesystem + manifest regen → page reload.**

Duplicate slots into exactly that shape.

```
HUD button (index.html)
   #scene-dup-btn  ⧉
        │ click
        ▼
client handler (src/gui/scene_manager.js)
   handleDuplicate()
        │ POST {source, name}
        ▼
dev save-server (server/save-server.js)  ← localhost:6970
   /duplicate-scene  →  fs.cpSync(sourceDir, destDir)  →  writeSceneManifest()
        │ {scene: "<new>"}
        ▼
client navigates to ?scene=<new>  (full reload, scene boots from copied YAML)
```

## What changed, by layer

### 1. New API endpoint — `POST /duplicate-scene` (`save-server.js`)

This is the one genuinely new piece of backend. It's a sibling to the existing
`/create-scene` and `/delete-scene` handlers and reuses their guards:

- Validates both `source` and `name` with `isValidSceneName()` (rejects unsafe
  path segments — no rewriting, fail loud per the codex).
- `404` if the source isn't a real scene (no `scene_config.yaml`), `409` if the
  destination already exists.
- The actual work is one line:
  `fs.cpSync(sourceDir, destDir, { recursive: true })`, which clones the whole
  scene directory — `scene_config.yaml` (fixtures), `controllers.yaml`,
  `patches.yaml`, `views.yaml`, `cameras.yaml`, and `playlists/`.
- Calls `writeSceneManifest()` so the scene dropdown picks up the new scene.

**Why an endpoint at all?** The browser can't write files. All persistence goes
through the dev save-server, which is also why these buttons are hidden on a
static host (GitHub Pages) — there's no backend to reach.

### 2. New UI control — `#scene-dup-btn` (`index.html`)

One `<button>` added between the `＋` and `🗑` buttons, using the existing
`.scene-btn` CSS class (no new styles needed) and the `⧉` copy glyph. The "UI
component" here is just the HUD scene picker cluster next to `#scene-select`.

### 3. Client wiring — `handleDuplicate()` (`scene_manager.js`)

Mirrors `handleAdd()`: opens the same themed in-app modal for a name (pre-filled
placeholder `<scene>_copy`), but additionally sends the currently selected scene
as `source`. It reads the source from `#scene-select` / `window.__activeScene`
(same way `handleDelete()` does), POSTs to the new endpoint, then reloads to
`?scene=<new>`. The static-host guard was extended so all three buttons hide
together.

## Key architectural points

- **No data model or schema changed.** A scene is "a directory under `scenes/`
  containing `scene_config.yaml`." Duplicate is just a directory copy — the
  copied YAML is already in the exact format the client boots from, so no
  transformation is needed.
- **The `marsin_engine/models/*.js` files are not touched.** Those are the
  engine's generated model exports, a separate concern from the sim's scene
  configs. The request was to duplicate "fixtures and controller configs," which
  all live in the scene directory.
- **Consistency over novelty.** Every choice (validation, manifest regen, modal,
  reload-on-success, static-host gating) reuses an established pattern, so the
  new endpoint behaves identically to its siblings and has the same security
  guards.

## Files touched

| Layer | File | Change |
|---|---|---|
| Backend | `simulation/server/save-server.js` | New `POST /duplicate-scene` handler |
| UI | `simulation/index.html` | New `#scene-dup-btn` button (⧉) |
| Client | `simulation/src/gui/scene_manager.js` | `handleDuplicate()` + wiring + static-host guard |
