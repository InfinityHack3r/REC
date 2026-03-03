# REC

Browser-based screen & camera recorder. Everything runs locally — zero data uploaded, no server, no sign-up.

Two versions are included:

| File | Size | Use when… |
|------|------|-----------|
| `REC.html` | Full | You want overlays, audio meters, device selection, recordings list, and Save to Disk |
| `REC-tiny.html` | ~16 KB | You want a minimal, dependency-free drop-in with no extras |

## Pages

Hosted on GitHub Pages, automatically deployed on every push to `main`:

- **Full:** https://infinityhack3r.github.io/REC/REC.html
- **Tiny:** https://infinityhack3r.github.io/REC/REC-tiny.html

---

## REC-tiny

A stripped-down single-file recorder. No fonts, no extra UI — just capture, record, pause, stop.

### Modes

| Mode | Description |
|------|-------------|
| **Screen** | Screen/tab/window capture with optional system audio and mic |
| **Camera** | Webcam only with optional mic |
| **Both** | Screen and camera rendered side-by-side onto a single canvas; the combined stream is recorded as one video |

### Controls

- **Q** — bitrate: 1 Mbps / 2.5 Mbps (default) / 5 Mbps
- **Res** — output resolution; options change based on mode:
  - Single (Screen or Camera): 1280×720 · 1920×1080 · 2560×1440 · 3840×2160
  - Both (side-by-side): 1280×360 · 1920×540 · 2560×720 · 3840×1080 (default)
- **Sys** checkbox — capture system/tab audio
- **Mic** checkbox — capture microphone audio
- **Capture → Record → Pause/Resume → Stop**

### File output

- Prefers `video/mp4;codecs=avc1`, falls back to `video/webm`
- If `showSaveFilePicker` is available (Chrome/Edge), prompts to choose a save location and streams chunks directly to disk
- Otherwise accumulates in memory and triggers a download on stop
- Filenames are timestamped: `rec-YYYY-MM-DD-HH-MM-SS.{mp4,webm}`

### File size estimator

Displays projected file sizes at 1 min / 10 min / 1 h / 2 h based on the selected bitrate **and resolution** (scales proportionally to pixel count relative to 1920×1080).

### What's not in tiny

No overlay layers, no audio VU meter, no device selection chips, no recordings list, no Google Fonts, no audio capture guide panel.

### Browser & edge case notes

**MIME / codec selection**

Tiny tries exactly three types in order: `video/mp4;codecs=avc1` → `video/webm;codecs=vp9,opus` → `video/webm`. The first type reported as supported by `MediaRecorder.isTypeSupported` wins.

| Browser | Result |
|---------|--------|
| Chrome / Edge | `video/mp4;codecs=avc1` (H.264 MP4) |
| Firefox | `video/webm;codecs=vp9,opus` — MP4 is never supported |
| Safari | `video/mp4;codecs=avc1` typically; bare `video/webm` may be used on older versions |

If the `MediaRecorder` constructor throws with the chosen type + bitrate options (rare, but possible with certain GPU encoders), tiny silently retries with `new MediaRecorder(str)` — no mimeType or bitrate — and the browser picks its default codec and quality.

**System audio retry**

If `getDisplayMedia` throws `NotReadableError` ("Could not start audio source") — the known Chrome bug when sharing Entire Screen with system audio — tiny automatically retries without audio and logs the retry in the status line. No other error types trigger a retry.

**"Both" mode specifics**

- The canvas is sized exactly to the selected resolution (e.g., 3840×1080). Screen fills the left half, camera fills the right half at half-width.
- Camera dimensions are requested as `{ width: { ideal: W/2 }, height: { ideal: H } }` — these are hints only. If the camera doesn't support the exact size, the browser picks the closest available and the video element scales to fit the half-canvas slot.
- Only the **screen track** has an `onended` handler wired to `kill()`. If the **camera stream** drops (e.g., another app takes the device), the canvas will freeze the camera half but recording continues. Stop manually if this happens.
- `canvas.captureStream(30)` is used for the combined output. This requires `captureStream` support — available in Chrome, Edge, Firefox, and Safari 11+. The canvas draw loop runs via `requestAnimationFrame`; the stream is capped at 30 fps.

**Audio mixing**

When both Sys and Mic are enabled, tiny creates an `AudioContext` + `createMediaStreamDestination` to mix the two tracks into one combined audio track. The `AudioContext` is created inside the click handler, satisfying the browser's user-gesture requirement (including Safari).

**No device selection**

Tiny always uses the browser's **default** camera and microphone. There is no UI to pick a specific device. If you need a particular mic or camera, set it as the system default before capturing, or use the full version.

**Resolution is a hint**

The Res dropdown passes `width` / `height` as `ideal` constraints to `getUserMedia` (camera) and uses them to size the canvas (Both mode). `getDisplayMedia` ignores the resolution hint entirely — the screen capture resolution is determined by the OS / share picker. For camera-only or Both mode, the actual resolution may differ if the hardware doesn't support the exact dimensions.

**Save to Disk**

- If `showSaveFilePicker` is available (Chrome/Edge), the file picker opens when you click **Record**. Cancelling the picker (`AbortError`) aborts the recording entirely — nothing is saved and the Record button re-enables.
- Any other `showSaveFilePicker` error (e.g., permissions policy) logs "Disk failed, download fallback" and falls back to the in-memory download path.
- On Firefox and Safari, `showSaveFilePicker` is absent; tiny logs "No disk API, download on stop" and accumulates chunks in memory.

**`killed` flag and corrupt-file prevention**

Tiny sets a `killed` flag when `kill()` is called (track ended, or Capture re-clicked). The `onstop` handler checks this flag and skips saving if set, preventing a corrupt or empty file from being written/downloaded after an unclean teardown.

`kill()` also explicitly calls `writable.close()` and nulls it out before `onstop` fires. This is necessary because `onstop` bails immediately when `killed=1` — without the early close in `kill()`, the `FileSystemWritableFileStream` would be left open indefinitely if the screen share ended or the user re-captured mid-recording.

The `AudioContext` created for audio mixing is stored as `ax` and closed (`ax.close()`) in `kill()`, preventing the context from leaking across re-captures.

**`beforeunload`**

`window.onbeforeunload` calls `recorder.stop()` and sets `e.returnValue` to trigger the "Leave site?" dialog mid-recording. Chrome and Edge honour this. Safari on iOS suppresses `beforeunload` completely — closing the tab on iOS will not trigger the guard and in-memory chunks will be lost.

---

## REC (full)

### Features

#### Capture modes

| Mode | Source | Audio |
|------|--------|-------|
| **Screen** | Tab, window, or entire screen via `getDisplayMedia` | System audio + optional mic |
| **Camera / Device** | Any connected camera via `getUserMedia` — select from detected devices | Mic only (system audio N/A, mic auto-enabled) |

#### Recording

- **Formats:** WebM (VP9/Opus) or MP4 (H.264/AAC) — selected before capture, availability depends on browser codec support
- **Quality:** Medium (2.5 Mbps), High (5 Mbps), Ultra (10 Mbps)
- **Pause / Resume** during recording
- **Timer** overlay on the preview while recording
- **REC badge** animates on the preview and header dot pulses red while recording
- Built-in playback + download for each completed recording in a list below the controls
- **Save to Disk** toggle — stream chunks directly to a local file via the File System Access API (Chrome/Edge only). Zero RAM usage during recording, no separate download step needed.
- **File size estimator** — shows projected sizes at 1 min, 10 min, 1 h, and 2 h, scaling with both the selected bitrate and resolution

#### Device selection

- **Microphone** — enable the Mic toggle to see a list of detected audio input devices; click a chip to select one
- **Camera / Device mode** — shows a list of detected video input devices; click a chip to select one
- System Audio toggle is automatically disabled (and mic auto-enabled) when Camera mode is selected

#### Safety

- If you close or refresh the tab while recording, the browser shows a **"Leave site?"** warning
- Stopping is triggered automatically on exit so the recorder attempts a clean shutdown
- In Save to Disk mode this gives the file stream the best chance of closing properly; WebM files are also more recoverable than MP4 if the stream is cut short

#### Overlay layers (burned into video)

Add composited overlays drawn directly onto the recorded output via an offscreen canvas pipeline.

| Layer type | Description |
|------------|-------------|
| **Text** | Customizable watermark with color, size, and opacity. Uses Rajdhani bold font. |
| **Camera** | Picture-in-picture webcam overlay with shape (rect / rounded / circle), mirror toggle, color border, size, and opacity. |
| **Image** | Upload any image as an overlay (e.g., logo). Supports size and opacity. |

- Click the preview to position the active layer
- Layers render in stack order with per-layer visibility toggle
- Select a layer with its dot indicator, then click anywhere on the preview to move it

#### Audio level meter

Dual-channel vertical VU meter displayed to the right of the preview:

- **SYS** — system / tab audio level
- **MIC** — microphone audio level
- dB scale (0 to −60) with color gradient (green → yellow → red) and peak hold indicators
- Always visible — shows idle state (−∞) when no audio is active
- Channel labels highlight when the respective source is active

#### Audio capture guide

Collapsible panel explaining how to enable system audio in Chrome / Edge share dialogs, with a compatibility table by share type:

| Share Type | Audio |
|------------|-------|
| Browser Tab | ✓ Tab audio ("Also share tab audio") |
| Entire Screen | ⚠ Buggy in Chrome — retries without audio automatically |
| Window | ✗ No audio available |

## Quick start

**Tiny version:** Open `REC-tiny.html` in any modern browser. Pick a mode, quality, and resolution, then Capture → Record → Stop.

**Full version:**

1. Open `REC.html` in **Chrome** or **Edge** (latest recommended).
2. Choose **Screen** or **Camera / Device** from the Source dropdown.
3. Select **Format** (WebM / MP4) and **Quality**.
4. Toggle **System Audio** and/or **Microphone** as needed. Select a specific device if multiple appear.
5. *(Optional)* Enable **Save to Disk** to write directly to a local file.
6. Click **Start Capture** — grant the browser permission when prompted.
7. *(Optional)* Add overlay layers (text, camera, image) and click the preview to position them.
8. Click **Record** → **Pause** / **Stop** as needed.
9. Play back or download recordings from the list below the controls.

No build step, no npm install. Just a browser.

## Browser compatibility

### Feature matrix

| Feature | Chrome | Edge | Firefox | Safari |
|---------|--------|------|---------|--------|
| Screen capture (`getDisplayMedia`) | ✓ | ✓ | ✓ | ✓ 13+ |
| Camera capture (`getUserMedia`) | ✓ | ✓ | ✓ | ✓ |
| WebM recording (VP9/Opus) | ✓ | ✓ | ✓ | ✗ |
| MP4 recording (H.264/AAC) | ✓ | ✓ | ✗ | ✓ (partial) |
| System audio — Tab sharing | ✓ | ✓ | ✗ | ✗ |
| System audio — Entire screen (Windows) | ⚠ buggy | ⚠ buggy | ✗ | ✗ |
| System audio — Entire screen (macOS) | ✗ | ✗ | ✗ | ✗ |
| System audio — Entire screen (Linux) | ✗ | ✗ | ✗ | ✗ |
| System audio — Window sharing | ✗ | ✗ | ✗ | ✗ |
| Overlay burn-in (`canvas.captureStream`) | ✓ | ✓ | ✓ | ✓ 11+ |
| Audio level meter (`AudioContext`) | ✓ | ✓ | ✓ | ✓ |
| Device selection (`enumerateDevices`) | ✓ | ✓ | ✓ | ✓ |
| Save to Disk (`showSaveFilePicker`) | ✓ | ✓ | ✗ | ✗ |
| Rounded camera overlay (`roundRect`) | ✓ 99+ | ✓ 99+ | ✓ 112+ | ✓ 15.4+ |

Best experience: latest **Chrome** or **Edge** on **Windows**.

---

### Chrome / Edge (Chromium)

**Recommended browsers.** All features work as designed.

- **System audio — tab sharing:** Tick "Also share tab audio" at the bottom of the share dialog. This is the most reliable path for audio capture on all platforms.
- **System audio — entire screen (Windows/ChromeOS):** Tick "Also share system audio". Known Chromium bug: this can raise `NotReadableError: Could not start audio source`. The recorder catches this and **automatically retries without audio**, then shows a warning banner with workarounds.
- **System audio — entire screen (macOS):** Chrome/Edge do not expose system audio on macOS regardless of the checkbox — macOS requires a virtual audio driver (e.g. BlackHole, Loopback) and a separate system configuration. The recorder will capture without audio and show the no-audio status.
- **System audio — window sharing:** No audio available from any app-window share in any browser. The audio checkbox does not appear.
- **MP4 fallback chain:** The recorder tries `video/mp4;codecs=h264,aac` → `video/mp4;codecs=h264` → `video/mp4` → `video/webm;codecs=h264` in order and uses the first supported type. If none match, it falls back to WebM.
- **`roundRect` on canvas:** Required for the "Rounded" camera overlay shape. Present in Chrome 99+ and Edge 99+. Older versions silently fall back to a plain rectangle.
- **Device labels in `enumerateDevices`:** Labels are empty strings until the user grants camera/mic permission at least once. The recorder requests a temporary stream to trigger the permission prompt, after which the device list populates properly.
- **Save to Disk:** `showSaveFilePicker` is Chromium-only. The toggle self-disables with an error banner on unsupported browsers; recording still works and saves via download on stop.
- **`beforeunload` clean-stop:** The recorder calls `recorder.stop()` inside `beforeunload` to attempt a clean shutdown when the tab is closed mid-recording. Chrome honours this. The disk writable stream is also closed in the same handler.

---

### Firefox

Firefox supports basic screen and camera capture and WebM recording, but several features are unavailable or behave differently.

- **System audio:** `getDisplayMedia` in Firefox does not expose system or tab audio. The recorder will capture video only; the no-audio status banner is shown.
- **MP4 / H.264:** Firefox's `MediaRecorder` does not support `video/mp4` MIME types. Selecting MP4 in the format dropdown will fall through the full codec chain and end up recording in WebM regardless. The filename will still use `.mp4` if MP4 was selected; rename to `.webm` or switch the format before recording.
- **Save to Disk:** `showSaveFilePicker` is not implemented. The toggle will self-disable with an error banner; recordings download via memory blob on stop.
- **`roundRect` on canvas:** Available from Firefox 112. On older versions the "Rounded" camera overlay shape renders as a plain rectangle without error.
- **`selfBrowserSurface` / `preferCurrentTab` / `systemAudio` constraints:** These `getDisplayMedia` hints are Chromium-specific and are silently ignored by Firefox.
- **`canvas.captureStream`:** Supported. Overlay burn-in pipeline works.

---

### Safari

Safari has partial support. Core capture and recording work in recent versions but many features are missing.

- **Screen capture:** Available from Safari 13+. The user sees a share picker similar to Chrome.
- **System and tab audio:** Not supported on any OS. `getDisplayMedia` never returns audio tracks in Safari.
- **MP4 / H.264:** Safari's `MediaRecorder` supports MP4 (H.264) on macOS/iOS, but codec negotiation behaves differently — `video/mp4;codecs=h264,aac` may not be recognised as a valid MIME string. The recorder falls through to the bare `video/mp4` type if needed.
- **WebM:** Not supported by Safari's `MediaRecorder`. If WebM is selected, `isTypeSupported` returns false for all VP9/VP8 types. The recorder picks up bare `video/webm` as a last resort, but Safari will likely fail or produce no output — use MP4 on Safari.
- **Save to Disk:** `showSaveFilePicker` is not implemented. Toggle self-disables; download on stop is used instead.
- **`AudioContext` and level meters:** Supported, but Safari requires a user gesture before an `AudioContext` can be created or resumed. Clicking "Start Capture" is sufficient.
- **`roundRect` on canvas:** Available from Safari 15.4.
- **`beforeunload`:** Safari on iOS suppresses `beforeunload` entirely. On iOS, closing the tab mid-recording will not trigger the "Leave site?" guard and the recording may be lost.
- **Mobile (iOS):** `getDisplayMedia` is not available on iOS Safari (and no other browser on iOS can use it either, due to WebKit engine restrictions). Camera capture via `getUserMedia` works.

---

### OS-level system audio notes

| OS | Tab audio | Full desktop audio |
|----|-----------|-------------------|
| Windows 10/11 | ✓ Chrome/Edge | ⚠ Buggy in Chrome (retry without audio); works in some Edge versions |
| macOS | ✓ Chrome/Edge tab audio | ✗ Requires third-party virtual audio driver |
| Linux (PulseAudio/Pipewire) | ✗ Not exposed | ✗ Not exposed |
| ChromeOS | ✓ Chrome | ✓ Chrome |

For macOS desktop audio, install [BlackHole](https://github.com/ExistentialAudio/BlackHole) or a similar loopback driver (e.g. Loopback) and set it as the system output; the recorder can then capture it via the Microphone input.

## Privacy

- All processing happens in-browser.
- No server component. No analytics. No cookies.
- Media is only saved when you explicitly stop and download / save it.
- Camera and mic access use the browser's standard permission dialogs.

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No system audio captured | Re-capture and tick the audio checkbox in Chrome's share picker. Tab sharing is most reliable. |
| Chrome "Could not start audio source" error | Known Chromium bug with "Entire Screen" + system audio. The recorder retries without audio automatically. Share a tab instead for reliable audio. |
| No system audio on macOS | macOS does not expose system audio to browsers. Use a virtual audio driver (e.g. [BlackHole](https://github.com/ExistentialAudio/BlackHole)) and capture it as the Mic source. |
| Microphone not working | Check mic permission in browser site settings. Re-capture after granting. |
| MP4 unavailable / records WebM anyway | Browser does not support H.264 in MediaRecorder (common on Firefox and some Linux setups). Switch format to WebM. |
| Camera overlay shows "No feed" | Click "Pick camera" to re-request access, or check that no other app is using the camera. |
| Audio meter stuck at −∞ | No audio source connected. Enable System Audio or Microphone and re-capture. |
| Save to Disk toggle does nothing | Browser does not support the File System Access API. Use Chrome or Edge. |
| File corrupt after forced close | Use WebM format — it is more recoverable than MP4 when a stream is cut short. |
| Device list is empty | Grant camera/mic permission once (the recorder requests a temporary stream to trigger enumeration), then re-open the device list. |
| "Rounded" camera overlay shows as rectangle | Browser is older than Chrome 99 / Firefox 112 / Safari 15.4 and does not support `canvas.roundRect`. Update your browser or use the Rect shape. |
| Screen share unavailable on iOS | iOS WebKit does not support `getDisplayMedia`. Camera capture still works. |

## Tech stack

- Static HTML + CSS + vanilla JavaScript
- No dependencies, no build tool, no framework
- Google Fonts: [Share Tech Mono](https://fonts.google.com/specimen/Share+Tech+Mono), [Rajdhani](https://fonts.google.com/specimen/Rajdhani)
- Web APIs: `MediaRecorder`, `getDisplayMedia`, `getUserMedia`, `enumerateDevices`, `AudioContext`, `AnalyserNode`, `Canvas2D`, `captureStream`, `FileSystemWritableFileStream`

## Development

Edit either file directly — no build step needed. Serve with any static server:

```bash
python3 -m http.server 8080
```

- Full: `http://localhost:8080/REC.html`
- Tiny: `http://localhost:8080/REC-tiny.html`

## Licence

MIT © 2026 [InfinityHack3r](https://github.com/InfinityHack3r)

See [LICENSE](LICENSE) for the full text.
