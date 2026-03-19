# VidReplace

## Description

VidReplace is a desktop video post-processing application that lets users select a rectangular region of a video, apply a VidVec optical-flow vector effect to that region across all frames, and composite the processed region back into the original video. The result is a seamless "patched" video where only the cropped area shows the effect.

The app combines two proven workflows:
- **Crop-Replace pipeline** (from `djz-cropreplacer-dev`): draw a crop box, process the cropped region, patch it back with edge blending.
- **VidVec effect pipeline** (from `djz-vidvec-dev`): optical flow vector rendering, color burn overlay, MODNet auto-masking, WebM/ZIP export.

**Stack:** Vite + TypeScript + React + Tailwind CSS. Desktop distribution via Tauri (Windows MSI).

---

## Reference Codebases

Both directories live alongside this file and are available at implementation time:

| Reference | Path | What to reuse |
|-----------|------|---------------|
| `djz-cropreplacer-dev` | `./djz-cropreplacer-dev/` | Crop box drawing, edge blending (`applyEdgeBlend`), compositing (`compositePatch`), colour matching (`colourMatch`, `colourMatchLAB`), overlay rendering, IndexedDB wrapper (`db.js`), ZIP export (`zip-export.js`), dark theme CSS variables, Tauri config |
| `djz-vidvec-dev` | `./djz-vidvec-dev/` | Sparse/dense optical flow (`sparseFlow.ts`, `denseFlow.ts`), vector renderer (`renderer.ts`), color burn compositing, MODNet masker (`modnetMasker.ts`), WebM encoder (`webmExporter.ts`), audio decoder (`audioDecoder.ts`), ZIP batch exporter (`zipExporter.ts`), OpenCV loader, frame stepper, FPS detection, output sizing, all TypeScript types |

Implementors should **read and port** from these references rather than reinventing. The plan below specifies how to combine them; code-level details can be lifted directly.

---

## Functionality

### 1. Video Loading

- Accept drag-and-drop or file picker for `.mp4` and `.webm` files.
- On load:
  - Create hidden `<video>` element with `preload="auto"`, `muted`, `playsInline`.
  - Detect FPS using the sampling technique from `djz-vidvec-dev/src/hooks/useVideoLoader.ts` (`detectFPS`): sample at 120 Hz over a 1-second window starting at 0.5 s, snap to common rates (23.976, 24, 25, 29.97, 30, 50, 59.94, 60) within 10% tolerance.
  - Probe for audio track using `probeHasAudio()` from `audioDecoder.ts`.
  - Compute `frameCount = Math.floor(duration * fps)`.
  - Store `VideoInfo`: `{ width, height, fps, frameCount, duration, fileSize, hasAudio, fileName }`.
- Display first frame on the main preview canvas.
- When a **new** video is loaded, flush all stored PNG frame blobs from IndexedDB (gallery webm entries persist until manually deleted).

### 2. Crop Box Drawing (Fixed for All Frames)

Port the crop-box interaction model from `djz-cropreplacer-dev/src/main.js` (lines 180–288), adapted for React:

#### Drawing
1. `mousedown` on the preview canvas → convert client coords to **video pixel space** using DPI-aware scaling: `x = Math.round((clientX - rect.left) * (canvas.width / rect.width))`. Store `boxStart`. Set `boxState = 'drawing'`.
2. `mousemove` → update `boxEnd`, call `renderOverlay()` which:
   - Draws the current video frame.
   - Fills a semi-transparent black overlay (`rgba(0,0,0,0.45)`).
   - Uses `globalCompositeOperation = 'destination-out'` to cut out the box area.
   - Clips and redraws the frame inside the box at full brightness.
   - Strokes the box outline (dashed while drawing, solid when confirmed).
   - Shows a tooltip with `W × H` dimensions.
3. `mouseup` → set `boxState = 'confirmed'`. Enable the "Effect" button only if both dimensions ≥ 64 px.

#### Repositioning
- If mouse down **inside** a confirmed box → enter drag mode. Track offset, constrain to video bounds, update box position on move.

#### Touch Support
- Mirror all mouse events with `touchstart`, `touchmove`, `touchend` using `e.touches[0]`.

#### Clear
- "Clear Box" button resets `boxState = 'idle'`, hides the overlay.

#### State Shape
```typescript
interface CropBox {
  x: number;  // top-left in video pixels
  y: number;
  w: number;
  h: number;
}
type BoxState = 'idle' | 'drawing' | 'confirmed';
```

The crop box is **fixed** — it applies to every frame of the video at the same pixel coordinates.

### 3. Effect Selection Modal

When the user clicks the **"Effect"** button (enabled only when `boxState === 'confirmed'`), open a full-screen modal overlay.

#### Modal Layout
Use the modal pattern from `djz-cropreplacer-dev` (overlay with centered `.modal` card), styled with the dark theme. The modal contains:

1. **Header**: "VidVec Effect Settings" + close (×) button.
2. **Effect selector**: A single option "VidVec" shown as a highlighted card (future-proofed for additional effects but only one exists now). Clicking it reveals the settings panel.
3. **Settings panel**: All VidVec parameters (see §4).
4. **Preview area**: A canvas showing the effect applied to the **current seek frame's crop region** (see §5).
5. **Footer buttons**:
   - **"Preview Frame"** — process only the current frame's crop and show it in the preview canvas.
   - **"Process All Frames"** — begin batch processing (see §6).
   - **"Cancel"** — close modal, discard nothing.

#### Modal Behavior
- Trap focus inside modal (Tab cycling).
- Escape key closes.
- Backdrop: `rgba(0,0,0,0.75)` with `backdrop-filter: blur(4px)`.

### 4. VidVec Settings

Port the `Settings` interface and `SettingsPanel` component from `djz-vidvec-dev`. All settings apply **only to the cropped region**, not the full frame.

```typescript
interface VidVecSettings {
  // Algorithm
  algorithm: 'sparse' | 'flowArrows';

  // Sparse (Lucas-Kanade)
  maxFeatures: number;       // 10–2000, default 300
  reseedInterval: number;    // 5–120 frames, default 30

  // Dense (Farnebäck)
  gridStep: number;          // 4–64, default 16

  // Shared rendering
  minMagnitude: number;      // 0–10, default 0.5
  lengthScale: number;       // 0.5–20, default 3.0
  maxLinePx: number;         // 5–200, default 40
  lineThickness: number;     // 1–5, default 1
  colourByAngle: boolean;    // default false

  // Output
  outputResMegapixels: number; // 0.1–2.0, default 0.5 (of crop region)

  // Compositing
  overlayMode: boolean;      // color burn overlay, default false

  // Masking
  autoMask: boolean;         // MODNet, default false
  invertMask: boolean;       // default false

  // Audio
  includeAudio: boolean;     // default false

  // Edge blending (from cropreplacer)
  edgeBlendPx: number;       // 0–32, default 8
}
```

**Persistence**: Save to `localStorage` under key `'vidreplace_settings'` with 500 ms debounce (same pattern as vidvec-dev).

**UI Controls**: Render each setting as labeled slider, toggle, or button group matching the `SettingsPanel` component layout from vidvec-dev, plus the `edgeBlendPx` slider from cropreplacer.

### 5. Single-Frame Preview

When the user clicks **"Preview Frame"** (or scrubs to a new frame on the main timeline):

1. **Seek** the video to the current frame using `useFrameStepper` pattern: set `video.currentTime = frameIndex / fps`, await `seeked` event.
2. **Extract crop region**: Draw the full frame to an offscreen canvas, then `getImageData(box.x, box.y, box.w, box.h)`.
3. **Compute output size** for the crop using `computeOutputSize(box.w, box.h, settings.outputResMegapixels)` from `outputSizing.ts`.
4. **Scale crop** to output size on a processing canvas.
5. **Run optical flow** on the crop canvas:
   - For the very first preview, there is no "previous frame" — show the raw crop (no vectors). Display a note: "Seek to another frame for flow preview."
   - On second preview (different frame), compute flow between the previously previewed crop and the current crop.
   - Use `processSparseFrame()` or `processDenseFrame()` depending on `settings.algorithm`.
6. **Render vectors** via `renderVectors()` onto a vector canvas at output size.
7. **Apply color burn** if `overlayMode` is on, using `compositeColorBurn()`.
8. **Apply MODNet mask** if `autoMask` is on, using `generateMask()` + `applyMask()`.
9. **Scale result back** to original crop pixel size (`box.w × box.h`) with high-quality downscaling.
10. **Apply edge blend** using the `applyEdgeBlend(canvas, edgeBlendPx)` algorithm from `djz-cropreplacer-dev/src/image-utils.js`: create alpha-gradient masks on all four edges.
11. **Composite** the blended patch onto a copy of the full video frame at `(box.x, box.y)` using `drawImage`.
12. **Display** the composited frame in the modal's preview canvas.

This gives the user an accurate preview of what a single frame will look like after processing and patching.

### 6. Batch Processing ("Process All Frames")

When the user clicks **"Process All Frames"**:

#### Processing Loop

Adapt the batch loop from `djz-vidvec-dev/src/App.tsx` (lines 290–374), scoped to the crop region:

```
for frameIndex = 0 to frameCount - 1:
  1. Seek to frame.
  2. Draw full frame to fullFrameCanvas (native video resolution).
  3. Extract crop region as ImageData from fullFrameCanvas.
  4. Scale crop to output resolution on cropProcessCanvas.
  5. Get ImageData from cropProcessCanvas.
  6. Compute optical flow (sparse or dense) between previous and current crop.
  7. Render vectors onto vectorCanvas (output res).
  8. If overlayMode: compositeColorBurn(vectorCanvas, cropProcessCanvas).
  9. If autoMask: generateMask(cropImageData) → applyMask(vectorCanvas, mask).
  10. Scale vectorCanvas back down to box.w × box.h.
  11. Apply edge blend.
  12. Composite blended patch onto fullFrameCanvas at (box.x, box.y).
  13. Encode composited full frame:
      a. Async PNG blob → push to ZIP batch exporter.
      b. VideoFrame → push to WebM encoder.
  14. Update progress state.
  15. Check cancellation flag.
```

#### Progress UI

Show the `ProgressBar` component (from vidvec-dev) with:
- Current frame / total frames.
- Elapsed time and ETA.
- Cancel button → sets `cancelRef.current = true`, loop breaks, partial output still exported.

#### Output Canvases

| Canvas | Size | Purpose |
|--------|------|---------|
| `fullFrameCanvas` | video native W×H | Holds each composited output frame |
| `cropProcessCanvas` | output res of crop | Optical flow input |
| `vectorCanvas` | output res of crop | Vector rendering target |
| `patchCanvas` | box.w × box.h | Downscaled + edge-blended patch |

All processing canvases are offscreen (`document.createElement('canvas')` or `OffscreenCanvas` where supported). Use `willReadFrequently: true` on contexts that call `getImageData()`.

#### Flow State Management

- **Sparse flow**: Call `initSparseFlow()` before loop, `processSparseFrame()` per frame, `cleanupSparseFlow()` after.
- **Dense flow**: Call `initDenseFlow()` before loop, `processDenseFrame()` per frame, `cleanupDenseFlow()` after.
- OpenCV Mat objects must be `.delete()`-ed to avoid WASM heap leaks (handled inside the flow modules).

#### Async PNG Compression (Rec-2 pattern)

Start `canvas.toBlob()` without awaiting; flush the previous frame's blob in the next iteration. This overlaps PNG compression with the next frame's flow computation.

### 7. Export

After batch processing completes (or is cancelled):

#### ZIP Export
- Use the `createBatchExporter(baseName)` pattern from `djz-vidvec-dev/src/lib/zipExporter.ts`.
- PNG frames named `frame_000001.png` through `frame_NNNNNN.png` (6-digit, 1-indexed).
- Auto-split at 250 MB per part: `vidreplace_{videoName}_{timestamp}.zip`, `_part002.zip`, etc.
- Trigger download via `FileSaver.saveAs()`.

#### WebM Export
- Use `createWebmEncoder(canvas, fps, baseName, audioBuffer?)` from `webmExporter.ts`.
- VP8 codec, 8 Mbps bitrate, keyframe every 60 frames.
- If `includeAudio && hasAudio`: decode source audio to AudioBuffer at 48 kHz via `decodeAudio()`, encode as Opus at 128 kbps.
- Output: `vidreplace_{videoName}_{timestamp}.webm`.

#### Export Button
After processing completes, show an **"Export"** button in the modal footer. Clicking it:
1. Finalizes and downloads the ZIP (if not already auto-downloaded in batches).
2. Finalizes and downloads the WebM.
3. Adds the WebM blob to the **gallery** (see §8).
4. Shows status: "ZIP files saved: N" / "WebM saved" / "WebM not supported in this browser".

### 8. Gallery & Storage

#### IndexedDB Schema

Use the wrapper pattern from `djz-cropreplacer-dev/src/db.js` (with memory fallback), adapted to TypeScript:

```typescript
// Database: 'vidreplace-db', version 1
// Stores:

interface FrameStore {
  id: string;          // UUID
  videoId: string;     // links to source video
  frameIndex: number;
  blob: Blob;          // PNG
  createdAt: number;
}

interface GalleryEntry {
  id: string;          // UUID
  videoId: string;
  fileName: string;    // original video name
  webmBlob: Blob;
  thumbnailBlob: Blob; // 128×72 PNG thumbnail
  settings: VidVecSettings;
  cropBox: CropBox;
  frameCount: number;
  createdAt: number;
}
```

**Stores**: `frames` (keyPath: `id`, index on `videoId`), `gallery` (keyPath: `id`).

#### Gallery UI

Below the main preview canvas, render a horizontal scrolling gallery strip (same pattern as cropreplacer's output gallery):
- 128×72 px thumbnail cards for each `GalleryEntry`.
- Click → open an output preview modal showing:
  - Full-size video player (create object URL from `webmBlob`).
  - Metadata: filename, settings summary, crop coords, date, frame count.
  - **Download** button → native save dialog via Tauri or `FileSaver.saveAs()`.
  - **Delete** button → remove from IndexedDB, refresh gallery.

#### Cleanup on New Video Load

When a new video file is loaded:
1. Clear all records from `frames` store (PNG blobs are transient).
2. Keep `gallery` store intact (WebM results persist).
3. Reset crop box state to `idle`.
4. Reset optical flow state (cleanup sparse/dense).
5. Reset processing state to `idle`.

---

## Technical Implementation

### Project Structure

```
vid-replacer/
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.ts
├── postcss.config.js
├── public/
│   └── opencv.js                    # Copy from djz-vidvec-dev/public/
├── src/
│   ├── main.tsx                     # React root
│   ├── App.tsx                      # Top-level layout + state orchestration
│   ├── types.ts                     # All interfaces (VideoInfo, CropBox, VidVecSettings, etc.)
│   ├── vite-env.d.ts
│   ├── components/
│   │   ├── DropZone.tsx             # Port from vidvec-dev
│   │   ├── VideoCanvas.tsx          # Main preview canvas + crop box drawing
│   │   ├── EffectModal.tsx          # Full-screen effect config modal
│   │   ├── SettingsPanel.tsx        # VidVec settings controls (port from vidvec-dev + edgeBlendPx)
│   │   ├── PreviewCanvas.tsx        # Single-frame effect preview inside modal
│   │   ├── ProgressBar.tsx          # Port from vidvec-dev
│   │   ├── Gallery.tsx              # WebM output gallery strip
│   │   └── OutputPreviewModal.tsx   # Gallery item detail view
│   ├── hooks/
│   │   ├── useOpenCV.ts             # Port from vidvec-dev
│   │   ├── useVideoLoader.ts        # Port from vidvec-dev
│   │   ├── useFrameStepper.ts       # Port from vidvec-dev
│   │   ├── useModNet.ts             # Port from vidvec-dev
│   │   ├── useCropBox.ts            # NEW: crop box drawing/dragging state machine
│   │   └── useDB.ts                 # NEW: IndexedDB hook (adapted from cropreplacer db.js)
│   ├── lib/
│   │   ├── opencvLoader.ts          # Port from vidvec-dev
│   │   ├── sparseFlow.ts            # Port from vidvec-dev
│   │   ├── denseFlow.ts             # Port from vidvec-dev
│   │   ├── renderer.ts             # Port from vidvec-dev
│   │   ├── modnetMasker.ts          # Port from vidvec-dev
│   │   ├── outputSizing.ts          # Port from vidvec-dev
│   │   ├── zipExporter.ts           # Port from vidvec-dev
│   │   ├── webmExporter.ts          # Port from vidvec-dev
│   │   ├── audioDecoder.ts          # Port from vidvec-dev
│   │   ├── edgeBlend.ts             # NEW: port applyEdgeBlend from cropreplacer image-utils.js
│   │   ├── compositor.ts            # NEW: patch compositing (drawImage at box coords)
│   │   └── db.ts                    # NEW: IndexedDB wrapper with memory fallback
│   └── styles/
│       └── index.css                # Tailwind base + dark theme custom properties
├── src-tauri/                       # Added after dev phase
│   ├── tauri.conf.json
│   ├── Cargo.toml
│   ├── src/
│   │   └── main.rs
│   └── capabilities/
│       └── default.json
└── djz-cropreplacer-dev/            # Reference (read-only)
└── djz-vidvec-dev/                  # Reference (read-only)
```

### Dependencies

```jsonc
{
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "jszip": "^3.10.1",
    "file-saver": "^2.0.5",
    "webm-muxer": "^5.1.4",
    "onnxruntime-web": "^1.24.2"
  },
  "devDependencies": {
    "@types/react": "^18.3.3",
    "@types/react-dom": "^18.3.0",
    "@types/file-saver": "^2.0.7",
    "typescript": "^5.5.3",
    "vite": "^5.3.4",
    "@vitejs/plugin-react": "^4.3.1",
    "tailwindcss": "^3.4.4",
    "postcss": "^8.4.39",
    "autoprefixer": "^10.4.19",
    "@tauri-apps/cli": "^2.10.1",
    "@tauri-apps/plugin-dialog": "^2.6.0",
    "@tauri-apps/plugin-fs": "^2.4.5"
  }
}
```

### Vite Configuration

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 1420,
    strictPort: true,
    headers: {
      'Cross-Origin-Opener-Policy': 'same-origin',
      'Cross-Origin-Embedder-Policy': 'require-corp',
    },
  },
  clearScreen: false,
});
```

The COOP/COEP headers are required for `SharedArrayBuffer` used by OpenCV WASM and `OfflineAudioContext`.

### Tailwind Theme

Port the dark color scheme from `djz-cropreplacer-dev/src/styles.css`:

```typescript
// tailwind.config.ts
export default {
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        bg: '#0F172A',
        surface: '#1E293B',
        card: '#263548',
        accent: '#63B3ED',
        error: '#FC8181',
        'text-primary': '#F1F5F9',
        'text-muted': '#94A3B8',
      },
    },
  },
};
```

### Key Implementation Details

#### `useCropBox` Hook

New hook encapsulating the crop box state machine:

```typescript
interface UseCropBoxReturn {
  boxState: BoxState;
  cropBox: CropBox | null;
  handlers: {
    onMouseDown: (e: React.MouseEvent<HTMLCanvasElement>) => void;
    onMouseMove: (e: React.MouseEvent<HTMLCanvasElement>) => void;
    onMouseUp: (e: React.MouseEvent<HTMLCanvasElement>) => void;
    onTouchStart: (e: React.TouchEvent<HTMLCanvasElement>) => void;
    onTouchMove: (e: React.TouchEvent<HTMLCanvasElement>) => void;
    onTouchEnd: (e: React.TouchEvent<HTMLCanvasElement>) => void;
  };
  clearBox: () => void;
}
```

Internally tracks `boxStart`, `boxEnd`, `isDragging`, `dragOffset`. Converts client coordinates to video pixel space using the canvas bounding rect and DPI ratio. Constrains box to video bounds. Minimum size 64×64.

#### `VideoCanvas` Component

```typescript
interface VideoCanvasProps {
  videoRef: React.RefObject<HTMLVideoElement>;
  videoInfo: VideoInfo | null;
  currentFrame: number;
  onFrameChange: (frame: number) => void;
  cropBox: CropBox | null;
  boxState: BoxState;
  cropHandlers: UseCropBoxReturn['handlers'];
}
```

Renders:
1. `<canvas>` element sized to fit container while maintaining video aspect ratio.
2. Scrub bar (horizontal slider) beneath canvas: `min=0`, `max=frameCount-1`, `value=currentFrame`.
3. Frame counter display: `Frame {current} / {total}`.
4. Overlay rendering on every frame change or box change.

#### `EffectModal` Component

Manages the entire effect workflow:
1. Renders `SettingsPanel` on the left (or top on narrow screens).
2. Renders `PreviewCanvas` on the right.
3. Renders `ProgressBar` when processing.
4. Holds refs to all processing canvases.
5. Contains the batch processing loop as an async function.
6. Uses `useRef` for cancellation flag.

#### Batch Processing Function

Located inside `EffectModal`. Pseudocode:

```typescript
async function processAllFrames() {
  const { box } = props;
  const { w: outW, h: outH } = computeOutputSize(box.w, box.h, settings.outputResMegapixels);

  // Init canvases
  const fullFrame = createCanvas(videoInfo.width, videoInfo.height);
  const cropProcess = createCanvas(outW, outH, { willReadFrequently: true });
  const vectorCanvas = createCanvas(outW, outH);
  const patchCanvas = createCanvas(box.w, box.h);

  // Init flow
  if (settings.algorithm === 'sparse') initSparseFlow();
  else initDenseFlow();

  // Init exporters
  const zip = createBatchExporter(`vidreplace_${videoInfo.fileName}`);
  const webm = createWebmEncoder(fullFrame.canvas, videoInfo.fps, `vidreplace_${videoInfo.fileName}`, audioBuffer);

  let prevBlobPromise: Promise<Blob | null> | null = null;

  for (let i = 0; i < videoInfo.frameCount; i++) {
    if (cancelRef.current) break;

    // 1. Seek
    await seekToFrame(i);

    // 2. Draw full frame
    fullFrame.ctx.drawImage(video, 0, 0);

    // 3. Extract crop at output res
    cropProcess.ctx.drawImage(
      video,
      box.x, box.y, box.w, box.h,  // source rect
      0, 0, outW, outH              // dest rect
    );
    const cropData = cropProcess.ctx.getImageData(0, 0, outW, outH);

    // 4. Compute flow
    const vectors = settings.algorithm === 'sparse'
      ? processSparseFrame(cropData, outW, outH, settings)
      : processDenseFrame(cropData, outW, outH, settings);

    // 5. Render vectors
    renderVectors(vectorCanvas.ctx, vectors, settings, outW, outH, outW, outH);

    // 6. Color burn
    if (settings.overlayMode) {
      compositeColorBurn(vectorCanvas.ctx, cropProcess.canvas, outW, outH);
    }

    // 7. Mask
    if (settings.autoMask) {
      const mask = await generateMask(cropData);
      applyMask(vectorCanvas.ctx, mask, outW, outH, settings.invertMask);
    }

    // 8. Downscale to patch size
    patchCanvas.ctx.drawImage(vectorCanvas.canvas, 0, 0, box.w, box.h);

    // 9. Edge blend
    applyEdgeBlend(patchCanvas.canvas, settings.edgeBlendPx);

    // 10. Composite onto full frame
    fullFrame.ctx.drawImage(patchCanvas.canvas, box.x, box.y);

    // 11. Encode (async PNG + WebM)
    if (prevBlobPromise) {
      const prevBlob = await prevBlobPromise;
      if (prevBlob) zip.add({ index: i - 1, blob: prevBlob });
    }
    prevBlobPromise = new Promise(resolve =>
      fullFrame.canvas.toBlob(resolve, 'image/png')
    );

    // WebM frame
    const vf = new VideoFrame(fullFrame.canvas, {
      timestamp: i * (1_000_000 / videoInfo.fps),
      duration: 1_000_000 / videoInfo.fps,
    });
    webm.encode(vf, { keyFrame: i % 60 === 0 });
    vf.close();

    // 12. Progress
    setProcessingState(prev => ({ ...prev, currentFrame: i + 1 }));
  }

  // Flush final PNG
  if (prevBlobPromise) {
    const blob = await prevBlobPromise;
    if (blob) zip.add({ index: videoInfo.frameCount - 1, blob });
  }

  // Finalize
  await zip.flush();
  const webmBlob = await webm.finalize();

  // Cleanup flow state
  if (settings.algorithm === 'sparse') cleanupSparseFlow();
  else cleanupDenseFlow();

  return webmBlob;
}
```

#### `edgeBlend.ts`

Port from `djz-cropreplacer-dev/src/image-utils.js`, the `applyEdgeBlend` function:

```typescript
export function applyEdgeBlend(canvas: HTMLCanvasElement, blendPx: number): void {
  if (blendPx <= 0) return;
  const ctx = canvas.getContext('2d')!;
  const { width: w, height: h } = canvas;

  // Create mask canvas
  const mask = document.createElement('canvas');
  mask.width = w;
  mask.height = h;
  const mctx = mask.getContext('2d')!;
  mctx.fillStyle = '#fff';
  mctx.fillRect(0, 0, w, h);

  // Left edge gradient
  const gl = mctx.createLinearGradient(0, 0, blendPx, 0);
  gl.addColorStop(0, 'rgba(255,255,255,0)');
  gl.addColorStop(1, 'rgba(255,255,255,1)');
  mctx.fillStyle = gl;
  mctx.fillRect(0, 0, blendPx, h);

  // Right edge gradient
  const gr = mctx.createLinearGradient(w - blendPx, 0, w, 0);
  gr.addColorStop(0, 'rgba(255,255,255,1)');
  gr.addColorStop(1, 'rgba(255,255,255,0)');
  mctx.fillStyle = gr;
  mctx.fillRect(w - blendPx, 0, blendPx, h);

  // Top edge gradient
  const gt = mctx.createLinearGradient(0, 0, 0, blendPx);
  gt.addColorStop(0, 'rgba(255,255,255,0)');
  gt.addColorStop(1, 'rgba(255,255,255,1)');
  mctx.fillStyle = gt;
  mctx.fillRect(0, 0, w, blendPx);

  // Bottom edge gradient
  const gb = mctx.createLinearGradient(0, h - blendPx, 0, h);
  gb.addColorStop(0, 'rgba(255,255,255,1)');
  gb.addColorStop(1, 'rgba(255,255,255,0)');
  mctx.fillStyle = gb;
  mctx.fillRect(0, h - blendPx, w, blendPx);

  // Apply mask
  ctx.globalCompositeOperation = 'destination-in';
  ctx.drawImage(mask, 0, 0);
  ctx.globalCompositeOperation = 'source-over';
}
```

#### `compositor.ts`

```typescript
export function compositePatch(
  fullFrameCtx: CanvasRenderingContext2D,
  patchCanvas: HTMLCanvasElement,
  box: CropBox
): void {
  fullFrameCtx.drawImage(patchCanvas, box.x, box.y, box.w, box.h);
}
```

#### `db.ts` — IndexedDB Wrapper

Port the pattern from `djz-cropreplacer-dev/src/db.js`, typed for TypeScript:

```typescript
const DB_NAME = 'vidreplace-db';
const DB_VERSION = 1;

interface DBSchema {
  frames: FrameStore;
  gallery: GalleryEntry;
}

class VidReplaceDB {
  private db: IDBDatabase | null = null;
  private memoryFallback: Map<string, Map<string, any>> = new Map();

  async open(): Promise<void> { /* open or fallback to memory */ }
  async put<K extends keyof DBSchema>(store: K, record: DBSchema[K]): Promise<void> { }
  async get<K extends keyof DBSchema>(store: K, id: string): Promise<DBSchema[K] | undefined> { }
  async getAll<K extends keyof DBSchema>(store: K): Promise<DBSchema[K][]> { }
  async delete<K extends keyof DBSchema>(store: K, id: string): Promise<void> { }
  async clearStore<K extends keyof DBSchema>(store: K): Promise<void> { }
}

export const db = new VidReplaceDB();
```

### Application State Overview

All state lives in `App.tsx` via React hooks:

```typescript
// Video
const [videoInfo, setVideoInfo] = useState<VideoInfo | null>(null);
const [currentFrame, setCurrentFrame] = useState(0);
const [hasAudio, setHasAudio] = useState(false);
const videoRef = useRef<HTMLVideoElement>(null);

// Crop
const { boxState, cropBox, handlers, clearBox } = useCropBox(videoRef, videoInfo);

// Settings
const [settings, setSettings] = useState<VidVecSettings>(loadSettings);

// Modal
const [modalOpen, setModalOpen] = useState(false);

// Processing
const [processingState, setProcessingState] = useState<ProcessingState>({
  status: 'idle', currentFrame: 0, totalFrames: 0, startTime: null, error: null,
});

// Gallery
const [galleryEntries, setGalleryEntries] = useState<GalleryEntry[]>([]);

// Output preview
const [previewEntry, setPreviewEntry] = useState<GalleryEntry | null>(null);
```

### UI Layout

```
┌──────────────────────────────────────────────┐
│  VidReplace                    [Settings ⚙]  │  ← Header
├──────────────────────────────────────────────┤
│                                              │
│         ┌──────────────────────┐             │
│         │                      │             │
│         │    Video Canvas      │             │  ← Main preview
│         │    (with crop box)   │             │
│         │                      │             │
│         └──────────────────────┘             │
│         ├━━━━━━━━●━━━━━━━━━━━━━┤             │  ← Scrub bar
│         Frame 142 / 900                      │
│                                              │
│  [Load Video]  [Effect]  [Clear Box]         │  ← Toolbar
│                                              │
├──────────────────────────────────────────────┤
│  Gallery: [thumb1] [thumb2] [thumb3] ...     │  ← WebM gallery strip
└──────────────────────────────────────────────┘
```

**Effect Modal** (overlays everything):

```
┌──────────────────────────────────────────────┐
│  VidVec Effect Settings              [×]     │
├───────────────────┬──────────────────────────┤
│  Algorithm ○ ○    │                          │
│  Max Features ━━● │   ┌──────────────────┐   │
│  Reseed ━━━━━━●   │   │                  │   │
│  Min Mag ━━●      │   │  Preview Canvas  │   │
│  Length ━━━━●     │   │  (composited)    │   │
│  Max Line ━━━●    │   │                  │   │
│  Thickness ━●     │   └──────────────────┘   │
│  Colour ☐         │                          │
│  Overlay ☐        │                          │
│  Mask ☐           │                          │
│  Edge Blend ━━●   │                          │
│  Output MP ━━●    │                          │
│  Audio ☐          │                          │
├───────────────────┴──────────────────────────┤
│  [Preview Frame]  [Process All Frames]       │
│  ━━━━━━━━━━━━━━━━━━━━ 45%  Frame 405/900    │  ← ProgressBar (during processing)
│  [Cancel]                [Export]             │
└──────────────────────────────────────────────┘
```

---

## Build Steps

### Phase 1: Scaffold & Core Infrastructure

1. Initialize Vite + React + TypeScript project.
2. Install all dependencies.
3. Configure `vite.config.ts` with COOP/COEP headers.
4. Configure Tailwind with dark theme.
5. Copy `opencv.js` to `public/`.
6. Port `types.ts` — combine interfaces from both references.
7. Port `lib/` utilities from `djz-vidvec-dev`: `opencvLoader.ts`, `sparseFlow.ts`, `denseFlow.ts`, `renderer.ts`, `modnetMasker.ts`, `outputSizing.ts`, `zipExporter.ts`, `webmExporter.ts`, `audioDecoder.ts`.
8. Create `lib/edgeBlend.ts` — port `applyEdgeBlend` from cropreplacer.
9. Create `lib/compositor.ts`.
10. Create `lib/db.ts` — IndexedDB wrapper.

### Phase 2: Video Loading & Preview

11. Port hooks: `useOpenCV.ts`, `useVideoLoader.ts`, `useFrameStepper.ts`, `useModNet.ts`.
12. Create `useDB.ts` hook.
13. Build `DropZone.tsx` (port from vidvec-dev).
14. Build `VideoCanvas.tsx` with scrub bar and frame counter.
15. Wire `App.tsx` with video loading → display first frame.

### Phase 3: Crop Box

16. Create `useCropBox.ts` hook with full mouse/touch state machine.
17. Integrate crop box overlay rendering into `VideoCanvas.tsx`.
18. Add toolbar buttons: Load Video, Effect, Clear Box.

### Phase 4: Effect Modal & Settings

19. Build `SettingsPanel.tsx` (port from vidvec-dev, add `edgeBlendPx`).
20. Build `PreviewCanvas.tsx`.
21. Build `EffectModal.tsx` shell with layout.
22. Implement single-frame preview pipeline.

### Phase 5: Batch Processing

23. Implement batch processing loop inside `EffectModal.tsx`.
24. Build `ProgressBar.tsx` (port from vidvec-dev).
25. Wire cancellation.
26. Wire ZIP and WebM export.

### Phase 6: Gallery & Export

27. Build `Gallery.tsx` — horizontal strip of WebM thumbnails.
28. Build `OutputPreviewModal.tsx` — detail view with video player.
29. Wire IndexedDB persistence for gallery entries.
30. Implement cleanup on new video load (clear `frames` store).
31. Wire Export button → finalize downloads + add to gallery.

### Phase 7: Polish

32. Add `index.css` with Tailwind base, dark theme custom properties, modal animations.
33. Keyboard accessibility: Escape closes modals, Tab trapping.
34. Error boundaries and error display for OpenCV/MODNet load failures.
35. Settings modal for localStorage management (clear data, reset defaults).

### Phase 8: Tauri Desktop Build

36. Run `npx tauri init` inside project root.
37. Configure `tauri.conf.json`: app name "VidReplace", window title, MSI bundle config.
38. Add `capabilities/default.json` with filesystem dialog permissions (port from cropreplacer's Tauri config).
39. Add native save dialog integration for ZIP/WebM export when running in Tauri.
40. Build MSI: `npm run tauri build`.

---

## Edge Cases & Error Handling

| Scenario | Handling |
|----------|----------|
| Video has no decodable frames | Show error toast, disable crop/effect buttons |
| Crop box < 64×64 | Prevent confirmation, show minimum size tooltip |
| OpenCV WASM fails to load | Show persistent error banner, disable effect button |
| MODNet model fails to load | Disable autoMask toggle, show inline warning |
| WebCodecs unsupported (no VP8 encoder) | Show "WebM not supported" status, still produce ZIP |
| Processing cancelled mid-way | Export whatever frames were completed |
| IndexedDB unavailable | Fall back to in-memory Map (no persistence across reloads) |
| Video file too large for memory | Process frame-by-frame (already the design), never load all frames |
| Browser tab backgrounded during processing | Processing continues (no requestAnimationFrame dependency) |
| Zero vectors on a frame (static scene) | Render black frame (overlay mode) or skip vectors (normal mode) |
| Audio decode fails | Set `hasAudio = false`, continue without audio track in WebM |

---

## Performance Considerations

- **All flow computation runs at output resolution** (not native video resolution). The `outputResMegapixels` setting controls crop processing size, keeping computation manageable even for 4K source video.
- **Async PNG compression** overlaps with next frame's flow computation.
- **Batch ZIP at 250 MB** prevents memory pressure from accumulating PNG blobs.
- **Canvas contexts** use `willReadFrequently: true` where `getImageData` is called.
- **OpenCV Mat cleanup** via `.delete()` prevents WASM heap growth.
- **MODNet session cached** — loaded once, reused across frames.
- **Pre-computed HSL LUT** for colour-by-angle mode (from renderer.ts).
- **Branch-hoisted mask loop** for JIT optimization.

---

## Accessibility

- All buttons have visible text labels (no icon-only buttons without aria-label).
- Modal focus trapping with Tab/Shift+Tab cycling.
- Escape key closes topmost modal.
- Canvas has `role="img"` with `aria-label` describing current state.
- Scrub bar is a native `<input type="range">` with `aria-label="Video frame"`.
- Progress bar has `role="progressbar"` with `aria-valuenow`, `aria-valuemin`, `aria-valuemax`.
- Status announcements via `aria-live="polite"` region.
- Color contrast meets WCAG AA on the dark theme (text-primary on bg = 13.5:1).
