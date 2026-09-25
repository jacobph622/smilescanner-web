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

## Why you pick the model instead of it being sniffed

On first load the page asks which device it is running on and remembers the
answer. Later visits go straight to the board; **Change model** in the console
revises it.

It is a type-ahead search, not a dropdown. Every model is its own suggestion —
`iPhone 15 Pro Max`, not a row shared with three other phones — and each
suggestion shows its panel resolution and ppi. It understands partial words
(`15 pro m`), no spaces (`iphone15promax`), `+` or `plus`, and the model codes a
phone reports, so `SM-S918U1` finds the Galaxy S23 Ultra and `CPH2649` the
OnePlus 13. When the browser gives enough to go on, the matching models are
listed before you type anything.

Auto-detection cannot be made reliable, and the failure is silent. iOS
**Display Zoom** set to *Larger Text* changes the logical screen size a device
reports, so an iPhone 15 Pro Max reports `375×812@3` — byte for byte the same
signature as an iPhone X. Believing that signature applies a 62.4 mm panel
width instead of the true 71.2 mm:

```
iPhone 15 Pro Max, Display Zoom = Larger Text
  signature match (iPhone X panel) : 18.031 px/mm   458 ppi
  model picked     (real panel)    : 15.794 px/mm   401 ppi
  error                            : +14.2%
  a 5.00 mm square really measures : 5.71 mm
```

No web API separates those two devices, so no amount of sniffing fixes it. One
question at startup does.

Given the model, the formula above stays correct *even under Display Zoom*,
because in that mode the render pixels genuinely are physically larger. The
scale is always the measured render buffer against the picked panel's real
width, never a stored px/mm.

Detection is still used — to suggest models, via the iOS screen signature or
the Android UA-client-hints model string. It is a starting guess, not the
answer. The page never hides which path it took: a badge is always on screen
(`460 PPI · SET`, `458.5 PPI · EDITED`, `401 PPI · MANUAL`, `CONFIRM MODEL`,
`SET MODEL`) and the console shows the ppi, the device, how it was decided, and
the screen diagonal that scale implies — a free sanity check against the spec
sheet.

### Tap the ppi to edit it

The ppi in the console is a button. Tap it, type a density, and the board
redraws; the edit is remembered for that model and **Reset to spec** undoes it.
Use it with the verify overlay to fine-tune a phone against a ruler.

The editable figure is the **panel** ppi — the number on a spec sheet — never
the effective render density. They differ on some phones: an iPhone 8 Plus
renders 1242 px across a 1080 px panel, so it draws at 461 ppi on a 401 ppi
panel. Someone who looks up "401" and types it must get the right board, so an
edit replaces the panel density and the formula does the rest. When the two
differ, the console says so (`renders at 461 ppi`); on an iPhone whose panel
does not normally downsample, that line is the tell that Display Zoom is on.

### If a phone measures wrong

1. Check the badge. If it does not say `SET`, the model was guessed.
2. Turn on **Verify overlay** and hold a bank card or steel rule against the
   glass. The outline is an ISO/IEC 7810 ID-1 card, 85.60 × 53.98 mm, with a
   50 mm rule marked in 1 mm ticks.
3. Tap the ppi and correct it, or pick **Not listed — enter the panel ppi** if
   the model is missing. Better still, add a row to `DEVICES` in `index.html`
   and send a PR. A row needs only panel pixels and ppi; the millimetres are
   derived.

## Using it

Pick your device the first time, set screen brightness to maximum by hand
(there is no web brightness API), then tap the board — or the scale badge in
the corner — to show or hide the console. It auto-hides after four seconds,
except while you are editing the ppi. The ChArUco board is shown by default.

| control | range |
|---|---|
| Board | ChArUco (default) or Checkerboard |
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
