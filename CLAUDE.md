# cardboard-web

Web port of a Google Cardboard VR app originally built in Unity with the Cardboard XR Plugin. Used for thesis data collection — participants open a URL instead of installing an app.

## What it does

Three-step flow before entering VR:

1. **`ruler.html`** — DPI calibration. User places a physical ruler or credit card against the screen and adjusts an on-screen reference image until they match. Calculates the device's true PPI and saves it to `localStorage`.
2. **`qr-scan.html`** — Cardboard viewer QR scan. Reads the viewer's profile (IPD, lens distortion coefficients, FOV) from its QR code using jsQR. Saves the raw base64url protobuf string to `localStorage`.
3. **`vr-scene.html`** — The VR experience. Decodes the saved QR protobuf, applies the viewer parameters to a manual stereo renderer built on A-Frame + Three.js, and enters stereoscopic mode.

## File structure

```
cardboard-web/
├── ruler.html          Step 1 — DPI calibration
├── qr-scan.html        Step 2 — Cardboard QR scan
├── vr-scene.html       Step 3 — Stereoscopic VR scene
└── assets/
    ├── ruler_image.png     124mm reference ruler (from Unity project)
    └── ruler_mark.png
```

No build step. No npm. Pure HTML/CSS/JS — deploy by uploading files to any static host.

## Hosting requirement

**Must be served over HTTPS.** The gyroscope (`deviceorientation` events) is blocked on plain HTTP in Chrome and requires explicit permission on iOS 13+. GitHub Pages is the intended host.

## How each step works

### Step 1 — DPI calibration (`ruler.html`)

Ported directly from `DPICalculator.cs` and `ScaleRuler.cs` in the Unity project.

- Reference object: 124mm ruler (`assets/ruler_image.png`) or credit card (85.6mm)
- Formula: `DPI = (cssWidth × window.devicePixelRatio) / referenceInches`
- `window.devicePixelRatio` is the web equivalent of Unity's `canvas.scaleFactor`
- `localStorage` is the web equivalent of Unity's `PlayerPrefs`
- Fine step: ±2px per press (matches `widthStep = 2f` in `ScaleRuler.cs`)
- Coarse step: ±10px. Hold-to-repeat on both.
- Saves: `localStorage.savedDPI`, `localStorage.rulerWidth_ruler`, `localStorage.rulerWidth_card`

### Step 2 — QR scan (`qr-scan.html`)

- Uses **jsQR** (CDN: `cdn.jsdelivr.net/npm/jsqr@1.4.0`) loaded synchronously before the main script
- Camera starts on user tap (not auto-start) to avoid iOS permission issues
- Scans every animation frame; shows any detected QR data in the status bar for feedback
- Cardboard QR URL format: `http://google.com/cardboard/cfg?p=<base64url-protobuf>`
- **Old viewers use `goo.gl/...` short URLs** (without `https://` prefix in the QR data). These are resolved automatically via `https://api.allorigins.win/get?url=<encoded>` CORS proxy. The proxy follows the redirect server-side and returns the final URL in `json.status.url`. If the proxy fails (8s timeout), falls back to a manual paste field.
- Saves: `localStorage.cardboardParams` (raw base64url `p` value), `localStorage.cardboardQRUrl`
- Manual URL entry is always available via "Enter URL manually instead"

### Step 3 — VR scene (`vr-scene.html`)

#### QR protobuf decoder

`decodeCardboardParams(base64urlStr)` in a `<script>` block in `<head>` — a minimal hand-written protobuf parser for the `CardboardDeviceParams` proto:

| Field | # | Type | Used for |
|---|---|---|---|
| vendor | 1 | string | display only |
| model | 2 | string | display only |
| screen_to_lens_distance | 3 | float32 | (decoded, not yet applied) |
| inter_lens_distance | 4 | float32 | `THREE.StereoCamera.eyeSep` |
| vertical_alignment | 5 | varint/enum | (decoded, not yet applied) |
| tray_to_lens_distance | 6 | float32 | (decoded, not yet applied) |
| distortion_coefficients | 7 | repeated float32 | shader uniforms k1, k2 |
| left_eye_fov_angles | 8 | nested message | camera vertical FOV |
| has_magnet | 10 | bool | (decoded, not applied) |

Defaults (Cardboard v1 generic viewer): IPD=64mm, k1=0.441, k2=0.156, FOV=100°.

Handles both packed and unpacked repeated floats for field 7. The `left_eye_fov_angles` nested message is parsed by `_parseFov()`.

#### Stereo rendering — `manual-stereo` A-Frame system

Registered as `AFRAME.registerSystem('manual-stereo', ...)`. Activated manually via `stereoSys.activate(ipd, k1, k2)` — **no WebXR dependency**.

**Two-pass render pipeline:**

```
Each frame (when active):
  Pass 1 — render to WebGLRenderTarget (off-screen)
    ├── left eye  → left half  of render target (viewport + scissor)
    └── right eye → right half of render target (viewport + scissor)
    Uses THREE.StereoCamera for correct IPD-offset camera pair

  Pass 2 — distortion pass → screen
    └── Full-screen quad with barrel distortion fragment shader
        reads render target texture, warps each eye half
```

The renderer is patched in `_patchRenderer()` which replaces `renderer.render` with a wrapper that calls `_drawStereo()` when active, otherwise calls the original. `_origRender` holds the unpatched function — used inside `_drawStereo` to avoid infinite recursion.

The `WebGLRenderTarget` is resized on `window.resize` via `_onResize()`.

#### Barrel distortion shader

Applied per eye half. Forward barrel formula moves the UV sample point outward, compressing the image inward (pincushion effect on screen). The Cardboard lens then expands it back to look undistorted.

```glsl
vec2 barrelUV(vec2 uv, vec2 lensCenter) {
    vec2 p = uv - lensCenter;
    p.x *= 2.0;                              // aspect correction per eye half
    float r2 = dot(p, p);
    p *= 1.0 + k1 * r2 + k2 * r2 * r2;     // forward barrel
    p.x /= 2.0;
    return p + lensCenter;
}
// Left eye lens centre:  (0.25, 0.5)
// Right eye lens centre: (0.75, 0.5)
// Pixels outside the valid eye region → black
```

`k1` and `k2` are shader uniforms updated from the decoded QR params at VR entry.

#### Head tracking

`look-controls` on the camera entity with `magicWindowTrackingEnabled: true; touchEnabled: true`. Touch drag works on desktop/testing. Gyroscope drives rotation when served over HTTPS. iOS 13+ permission is requested inside `enterVR()` (must be a direct user gesture).

#### aframe-stereo-component

Included from CDN (`cdn.jsdelivr.net/npm/aframe-stereo-component`). Used for Three.js layer assignment:
- Camera entity: `stereocam="eye: left"`
- All scene objects: `stereo="eye: both"` (layer 0 — visible to both eyes)

#### Data collection

`window.collectDataPoint(eventType, payload)` writes timestamped events to `localStorage.thesisData` (JSON array). Call it from any scene script to log participant interactions. Each entry includes `t` (timestamp ms), `dpi`, `ipd`.

## Key gotchas

- **`enterVR()` must be called synchronously inside a user gesture handler.** iOS blocks DeviceOrientationEvent permission requests made outside a gesture. The current button `onclick="enterVR()"` satisfies this.
- **`_origRender` vs `renderer.render`** — inside `_drawStereo`, always call `this._origRender(...)`, never `renderer.render(...)`, to avoid infinite recursion from the patched wrapper.
- **goo.gl URLs have no `https://` prefix in the QR data** — always prepend `https://` before passing to fetch or the allorigins proxy: `shortUrl.startsWith('http') ? shortUrl : 'https://' + shortUrl`.
- **`screen_to_lens_distance` and `tray_to_lens_distance`** are decoded from the QR but not yet applied. They affect the vertical lens-centre offset (how high the viewport sits). If the image looks vertically off in the viewer, implement a vertical shift based on these values.
- **jsQR must be loaded before the script block that calls it.** The `<script src="jsqr...">` tag is placed immediately before the main `<script>` block in `qr-scan.html`.

## Source Unity project

Original app: `C:\Users\epyyn\Downloads\google_cardboard_calibration_unity_plugin-main\...\cardboard-xr-plugin-master`

Key Unity scripts and their web equivalents:

| Unity | Web |
|---|---|
| `DPICalculator.cs` | `ruler.html` JS |
| `ScaleRuler.cs` | `ruler.html` JS |
| `Api.cs` (`ScanDeviceParams`) | `qr-scan.html` |
| `XRLoader.cs` (`RecalculateRectangles`) | `manual-stereo` system |
| `CardboardReticlePointer.cs` | A-Frame `cursor` with `fuse: true; fuseTimeout: 1500` |
| `PlayerPrefs` | `localStorage` |
| `canvas.scaleFactor` | `window.devicePixelRatio` |
