# VidReplace

A Windows desktop app for applying video effects to selected regions of a video. Draw a crop box (or animate it with keyframes), choose from 9 built-in effects, and export the result as a WebM video and PNG frame sequence.

All processing happens locally on your machine — no uploads, no cloud services, no API keys required.

## Install

Download the latest **VidReplace MSI installer** from the [Releases page](https://github.com/MushroomFleet/vid-replacer-dev/releases).

Run the installer and launch VidReplace from your Start menu.

## How It Works

1. **Load a video** (MP4 or WebM) by dragging it onto the app or clicking "Choose File"
2. **Draw a crop box** on the video to select the region you want to apply an effect to
3. **Choose an effect** from the dropdown in the Effect panel
4. **Adjust settings** for the selected effect
5. **Process all frames** — the effect is applied only inside the crop box, then patched back into the original video
6. **Export** — download a ZIP of PNG frames and a rendered WebM video

## Crop Modes

### Fixed Mode
The crop box stays in the same position for every frame. Draw once, apply everywhere.

### Tracking Mode
Animate the crop box across the video using keyframes:
- Switch to **TRACKING** mode
- Scrub to a frame and **click** to place a keyframe at the object's center
- Scrub to another frame and click again
- The crop box position is automatically interpolated between keyframes
- Place as many keyframes as you need for smooth motion

Set the crop box dimensions (W/H) using the size controls in the toolbar.

## Effects

### VidVec (Motion Vectors)
Renders optical flow vectors as lines on each frame. Supports sparse (Lucas-Kanade) and dense (Farneback) algorithms. Optional color burn overlay mode composites the vectors onto the original footage. Includes MODNet AI person masking for clean overlay compositing.

### Datamosh 1 — Glide
Block-based motion estimation displaces pixels across frames, creating flowing distortion trails and motion smears.

### Datamosh 2 — Multi-Mosh
Three sub-modes: **Glide Loop** extracts motion from the first two frames and repeats it infinitely. **Movement** applies frame-by-frame motion compensation. **Copy** passes frames through unchanged.

### Datamosh 3 — I-Frame Kill
Simulates missing keyframes by accumulating pixel deltas. Colors bloom and stack on themselves in the classic datamosh style. **Delta Repeat** sub-mode loops a short motion pattern cyclically.

### Datamosh 4 — Vector Transfer
Extracts motion vectors from the video and re-applies them with doubled intensity, amplifying all motion in the scene.

### Datamosh 5 — Size Sort
Reorders frames by their visual complexity (measured by compressed PNG size). Creates a progression from simple to complex frames or vice versa.

### Datamosh 6 — Pixel Sort (Sobel)
Sobel edge detection defines segment boundaries. Pixels between edges are sorted by luminance, creating horizontal glitch streaks.

### Datamosh 7 — Pixel Sort (Advanced)
Multi-mode pixel sorting with four sort criteria (luminance, hue, saturation, edge intensity), adjustable rotation for vertical/diagonal sorting, and optional multi-pass mode that applies all four criteria sequentially.

### Datamosh 8 — Pixel Sort (Masked)
Same as Advanced pixel sort but restricted to a mask region. Choose **Auto Luma** (brightness threshold) or **Auto MODNet** (AI person detection) to control which areas get sorted.

## AI Features

### MODNet Person Masking
The MODNet AI model (~25 MB) is downloaded automatically the first time you enable a masking feature. It is cached locally in your browser storage for instant reuse on future sessions. MODNet detects people in the frame and generates a silhouette mask used for:
- **Mask Overlay** (VidVec) — clean compositing without a visible rectangle
- **Masked Pixel Sort** (Datamosh 8) — sort only foreground/background regions

### OpenCV Optical Flow
OpenCV.js (WASM) powers the VidVec motion vector analysis. It loads automatically when the app starts.

## Export

After processing completes:
- **PNG frames** are packaged into a ZIP file and downloaded automatically (auto-splits at 250 MB per part)
- **WebM video** is encoded from the PNG frames with VP8 codec and downloaded separately
- Optional **source audio** can be included in the WebM output (Opus codec)

Processed videos are saved to the in-app **Gallery** for review and re-download.

## Debug Console

Press **Ctrl+F9** to toggle the built-in debug console, which shows real-time log output for troubleshooting.

## System Requirements

- Windows 10/11 (x64)
- WebView2 runtime (included with Windows 10 1803+ and Windows 11)
- ~200 MB disk space
- Recommended: 8 GB+ RAM for processing long videos

---

## Citation

If you use this codebase in your research or project, please cite:

```bibtex
@software{vidreplace,
  title = {VidReplace: Regional Video Effect Processor with Keyframe Crop Animation},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/vid-replacer-dev},
  version = {3.1.0}
}
```

### Donate:

[![Ko-Fi](https://cdn.ko-fi.com/cdn/kofi3.png?v=3)](https://ko-fi.com/driftjohnson)
