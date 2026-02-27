# REC

Browser-based screen & camera recorder. Everything runs locally — zero data uploaded, no server, no sign-up.

Single HTML file. Open it and go.

## Features

### Capture modes

| Mode | Source | Audio |
|------|--------|-------|
| **Screen** | Tab, window, or entire screen via `getDisplayMedia` | System audio + optional mic |
| **Camera** | Any connected camera device via `getUserMedia` | Mic only (system audio N/A) |

### Recording

- **Formats:** WebM (VP9/Opus) or MP4 (H.264/AAC) — availability depends on browser codec support
- **Quality:** Medium (2.5 Mbps), High (5 Mbps), Ultra (10 Mbps)
- **Pause / Resume** during recording
- **Timer** overlay while recording
- Built-in playback + download for each recording

### Overlay layers (burned into video)

Add composited overlays that are drawn directly onto the recorded output via an offscreen canvas pipeline.

| Layer type | Description |
|------------|-------------|
| **Text** | Customizable watermark with color, size, and opacity. Uses Rajdhani bold font. |
| **Camera** | Picture-in-picture webcam overlay with shape (rect / rounded / circle), mirror toggle, color border, size, and opacity. |
| **Image** | Upload any image as an overlay (e.g., logo). Supports size and opacity. |

- Click the preview to position the active layer
- Layers render in stack order with per-layer visibility toggle

### Audio level meter

Dual-channel vertical VU meter displayed to the right of the preview:

- **SYS** — system / tab audio level
- **MIC** — microphone audio level
- dB readout with color coding (green → yellow → red) and peak hold indicators
- Always visible — shows idle state when no audio is active

### Audio capture guide

Collapsible panel explaining how to enable system audio in Chrome / Edge share dialogs, with a compatibility table by share type (tab / screen / window).

## Quick start

1. Open `REC.html` in **Chrome** or **Edge** (latest recommended).
2. Choose **Screen** or **Camera** from the Source dropdown.
3. Toggle **System Audio** and/or **Microphone** as needed.
4. Click **Start Capture** — grant the browser permission when prompted.
5. *(Optional)* Add overlay layers (text, camera, image) and click the preview to position them.
6. Click **Record** → **Pause** / **Stop** as needed.
7. Download recordings from the list below the controls.

No build step, no npm install. Just a browser.

## Browser compatibility

Best experience: latest **Chrome / Edge**.

- `getDisplayMedia` and `MediaRecorder` required.
- MP4 recording depends on browser H.264 encoder support.
- System audio capture varies by OS and share type — tab sharing is most reliable.
- `canvas.captureStream` required for overlay burn-in pipeline.
- `AudioContext` + `AnalyserNode` required for level meters.

Firefox and Safari may work for basic screen capture but overlay, meter, and audio features may differ.

## Privacy

- All processing happens in-browser.
- No server component. No analytics. No cookies.
- Media is only saved when you explicitly download it.
- Camera and mic access use the browser's standard permission dialogs.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No system audio captured | Re-capture and tick the audio checkbox in Chrome's share picker. Tab sharing is most reliable. |
| Microphone not working | Check mic permission in browser site settings. Re-capture after granting. |
| MP4 unavailable | Browser may not support the H.264 MediaRecorder MIME type. Use WebM. |
| Camera overlay shows "No feed" | Click "Pick camera" to re-request access, or check that no other app is using the camera. |
| Audio meter stuck at -∞ | No audio source connected. Enable System Audio or Microphone and re-capture. |

## Tech stack

- Static HTML + CSS + vanilla JavaScript
- No dependencies, no build tool, no framework
- Google Fonts: [Share Tech Mono](https://fonts.google.com/specimen/Share+Tech+Mono), [Rajdhani](https://fonts.google.com/specimen/Rajdhani)
- Web APIs: `MediaRecorder`, `getDisplayMedia`, `getUserMedia`, `AudioContext`, `AnalyserNode`, `Canvas2D`, `captureStream`

## Development

Edit `REC.html` directly. Serve with any static server:

```bash
python3 -m http.server 8080
```

Open `http://localhost:8080/REC.html`.
