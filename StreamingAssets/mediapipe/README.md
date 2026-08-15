# Browser-side ASL assets (WebGL builds)

These files are served by the game's own web host and loaded by
`Assets/Plugins/WebGL/AslWebBridge.jslib`. Everything here exists so a WebGL build can
run hand tracking **without contacting any server**.

## What is here

| File | Source | Status |
|---|---|---|
| `hand_landmarker.task` | copied from `ASLLocalServer/models/` | ✅ present |
| `vision_bundle.mjs` | MediaPipe `@mediapipe/tasks-vision` | ⬜ **you must add** |
| `wasm/` | MediaPipe `@mediapipe/tasks-vision/wasm` | ⬜ **you must add** |

`hand_landmarker.task` is byte-identical to the one the Python server uses, so the
browser and the desktop build track hands with the same model.

## Adding the two missing pieces

They are not committed here because they are third-party redistributables that have to be
fetched deliberately rather than pulled in by a build script.

```bash
npm pack @mediapipe/tasks-vision
# unpack, then copy:
#   package/vision_bundle.mjs        -> Assets/StreamingAssets/mediapipe/vision_bundle.mjs
#   package/wasm/*                   -> Assets/StreamingAssets/mediapipe/wasm/
```

The paths are configurable on the `AslPredictionClient` component
(*Browser Bridge (WebGL only)* header), so a different layout only needs the fields
updated. Pointing `Vision Module Url` at a CDN also works and skips this step entirely —
but that reintroduces a network dependency at load time, which is the thing this setup
exists to remove.

## Serving requirements

- **HTTPS or `localhost`.** `navigator.mediaDevices.getUserMedia` is unavailable on plain
  `http://` origins, so the camera silently fails to open on an insecure host.
- **`.wasm` served as `application/wasm`.** Some static hosts return `text/plain`, which
  makes the streaming compile fall back or fail outright.
- **`.mjs` served as `text/javascript`.** A wrong MIME type makes the dynamic `import()`
  reject and the bridge reports `init failed`.

When the bridge cannot start for any of these reasons it reports `Failed`, and
`AslPredictionClient` falls back to the cloud endpoint rather than leaving ASL mode dead.

## What still needs the cloud

Movement (W/A/S/D) and look (H/L) are pure geometry over the 21 landmarks and run fully
offline here. **ASL letter classification does not** — the 96×96 CNN in
`ASLLocalServer/models/best_asl_model.h5` has not been ported to the browser, so `letter`
is always empty from the bridge and spelling falls through to the cloud endpoint.

Porting it needs either an ONNX conversion of that CNN, or the landmark-MLP retrain, which
is blocked on the training dataset not being in this repository.
