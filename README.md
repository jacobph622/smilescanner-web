# On-Screen ChArUco Target

Turns a phone screen into a camera-calibration target: a checkerboard, or a
ChArUco board with a unique `DICT_4X4_100` ArUco marker in every white square,
drawn at **physically exact millimetre sizes**.

**Live page: <https://jacobph622.github.io/smilescanner-web/>**

A web port of [`Orthotech1/SmileScanner_Andtroid`](https://github.com/Orthotech1/SmileScanner_Andtroid),
whose Android original uses `displayMetrics.xdpi` to convert millimetres to
pixels. The marker dictionary here is byte-identical to the Kotlin table, so
both renderers produce the same board — detect with `cv::aruco::DICT_4X4_100`.

## The hard part: millimetres on the web

No browser API reports a screen's true physical density. CSS `mm` and `in` are
pinned to the fiction that `1in = 96px`, and `devicePixelRatio` only relates CSS
pixels to device pixels — neither knows the real panel. So the scale is derived
from the device's known physical screen width instead:

```
pxPerMm = (screen.width_css * devicePixelRatio) / physicalScreenWidthMm
```

`screen.width_css * devicePixelRatio` is the render buffer in real pixels, which
makes the result invariant to:

- **browser page zoom** — both terms scale inversely, so it cancels
- **Android display-size / Samsung Screen-zoom** — those move density, not panel size
- **phones rendering below native resolution** — e.g. a Galaxy S21 Ultra left on
  FHD+ correctly reads 386 ppi rather than its 515 ppi panel spec, because its
  render pixels really are physically larger

It also handles Apple's downsampling models for free, where the render buffer is
not the panel: iPhone 6–8 Plus render 1242×2208 onto a 1080×1920 panel, and
iPhone 12/13 mini render 1125×2436 onto 1080×2340. Both are ~8% off if you
assume the panel spec.

Physical width comes from panel pixels ÷ panel ppi, both fixed published specs.
The device tables therefore store **panel pixels and ppi** and derive
millimetres — never a hardcoded px/mm.

## How the device is identified

| platform | method | confidence |
|---|---|---|
| iOS / iPadOS | `screen` dimensions + `devicePixelRatio` signature | exact for catalogued signatures |
| Android | model string from UA client hints, matched against a device table | exact for catalogued models |
| anything else | `devicePixelRatio × 160` (Android sets `densityDpi` near true ppi) | estimate, may drift several percent |

The page never hides which path it took. A badge is always on screen — `SCALE OK
460 PPI`, `CONFIRM MODEL`, or an amber `ESTIMATED SCALE` — and the console shows
the ppi, the device, the detection method, and the screen diagonal that scale
implies, as a sanity check against the spec sheet.

Where a signature is genuinely ambiguous the page asks rather than guessing.
`375×812@3` is shared by the iPhone X / XS / 11 Pro (458 ppi) and the iPhone
12 mini / 13 mini (476 ppi) — an 8% split no web API can resolve — so it offers
a one-tap choice and remembers it.

### If a phone measures wrong

1. Turn on **Verify overlay** and hold a bank card or steel rule against the
   glass. The outline is an ISO/IEC 7810 ID-1 card, 85.60 × 53.98 mm, with a
   50 mm rule marked in 1 mm ticks.
2. Type the true panel ppi into **Override ppi**. It is saved per device.
3. Better: add a row to `ANDROID_DB` or `IOS_SIGS` in `index.html` and send a PR.

Two known limits: an unrecognised Android model falls back to the estimate, and
iOS **Display Zoom** set to *Larger Text* changes the reported screen size, so
leave it on *Default*.

## Using it

Open the page, set screen brightness to maximum by hand (there is no web
brightness API), and tap the board to show or hide the console. It auto-hides
after four seconds.

| control | range |
|---|---|
| Board | Checkerboard or ChArUco |
| Square | 1.0 – 10.0 mm, 0.5 mm steps |
| Columns / Rows | bounded by what fits on screen, and for ChArUco by the 100-marker dictionary |
| Marker | 0.40 – 0.90 of the square |

Feed the calibrator the **as-rendered** geometry from the console, not the
requested size. Squares are snapped to whole device pixels, so a 5.0 mm request
becomes e.g. `5.042 mm (59 × 59 px)`. Snap error stays under about 30 µm; the
readout always reports what was actually drawn.

The marker slider stops below 1.0 because OpenCV requires
`markerLength < squareLength` — the white margin around each marker is what lets
the detector separate it from the black squares.

## Files

| file | |
|---|---|
| `index.html` | the whole app: one self-contained ASCII file, no build step, no dependencies beyond a webfont |
| `qr.html` | encodes the page URL as a QR code locally in the browser; nothing is sent anywhere |

Drop `index.html` on any static host. It is pure client-side and stores only a
ppi override and a model choice in `localStorage`.
