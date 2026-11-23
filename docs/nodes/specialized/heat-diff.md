---
title: "Heat-Diff Node - Difference Heat Map of Two Images"
description: "Compute the per-pixel difference of two images in Node-RED as a color-mapped heat image, with optional difference statistics, SSIM and changed regions. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, heat-diff, image difference, heat map, golden image, change detection, ssim, psnr, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Heat-Diff Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Compute the per-pixel difference of two images in Node-RED as a color-mapped heat image, with optional difference statistics, SSIM and changed regions."
---

# Heat-Diff Node

> **Type**: `rp-heat-diff` · **Category**: Specialized · **Since**: v1.5.1

## Overview

Heat-Diff compares two images pixel by pixel: it takes the absolute difference of their grayscale versions, optionally blurs it and suppresses small values, and returns it as a heat map in a chosen colormap. It can also measure the difference (mean, maximum, percentiles, RMSE, PSNR, SSIM, share of changed pixels and the changed regions), with or without the heat image.

**Use cases**
- Compare each product image with a golden image and see where they differ
- Decide pass or fail from `changedRatio`, `maxDiff` or the largest changed region, without rendering an image
- Check the result of [image-align](./image-align.md) before comparing

## Inputs

Only `msg` properties are read for the images: the editor also offers `flow` and `global`, but the node reads the typed text as a `msg` path.

- **Image 1** (`msg.payload` by default) and **Image 2** (`msg.goldenImage` by default): raw image objects or encoded JPEG/PNG/WebP/BMP `Buffer`s, in any color space. Both may also be arrays of the same length: the images are then paired by index, compared in parallel, and the outputs become arrays in the same order.
- Images of different sizes are both scaled (without keeping the aspect ratio) to the largest width and height of the two.

## Outputs

One output.

- **Output** (`msg.payload` by default; a `msg` property): the heat image, a 3-channel image (`colorSpace` `BGR` in raw form) of the compared size, or an encoded `Buffer`. With **Heat Image** off, Output is left untouched.
- **Stats to** (`msg.diffStats` by default; `msg`, `flow` or `global`), with **Statistics** on:

```javascript
{
  validRatio: 0.98,       // share of pixels compared (see below)
  changedRatio: 0.012,    // share of compared pixels whose difference is above Threshold
  meanDiff: 3.4,          // mean difference, 0-255
  maxDiff: 187,           // largest difference
  p50: 2, p95: 11, p99: 40,  // difference percentiles
  rmse: 6.1,
  psnr: 32.4,             // dB; 100 when the images are identical
  ssim: 0.97,             // null when SSIM is off
  threshold: 25,
  blobCount: 3,           // connected regions of changed pixels
  largestBlobArea: 412,   // pixels
  blobs: [ { x, y, w, h, area, cx, cy }, ... ]   // up to 50 regions, largest first, in pixels
}
```

Pixels that are pure black (0) in either grayscale image are left out of every statistic: they are usually the borders of an alignment or the padding of a composite. `validRatio` is the share of pixels that remain. The statistics are computed on the difference after Blur and Threshold, except SSIM, which compares the two grayscale images.

- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds.
- On failure: `msg.error` (`message`, `operation`, `hint`, `details`, `timestamp`, `nodeId`, `nodeName`, `nodeType`), with Image 1 written to **Output**; the message is still sent.

## Configuration

| Field | Type | Default | Description |
|---|---|---|---|
| Name | string | empty | Label of the node; also the key of its timings in `msg.performance.rpimage` |
| Image 1 | msg path | `msg.payload` | First image (for example the capture) |
| Image 2 | msg path | `msg.goldenImage` | Second image (for example the golden image) |
| Heat Image | boolean | on | **Render the heat map (off = statistics only)**. Off also turns Statistics on |
| Output | msg path | `msg.payload` | Where the heat image is written |
| Output Format | enum | Raw (fastest) | Raw (fastest), JPEG, PNG or WebP, with Quality (`90`), PNG Optimize and the WebP options as in [resize](../transform/resize.md#output-format-options) |
| Statistics | boolean | on | **Compute diff statistics** |
| Stats to | TypedInput (msg/flow/global) | `msg.diffStats` | Where the statistics are written |
| SSIM | boolean | on | **Compute SSIM (dominates the stats cost)**: structural similarity with an 11 × 11 Gaussian window |
| Debug / Debug Width | boolean / number | off / `200` | **Show processed image** under the node in the editor |
| Colormap | enum | Jet | Jet, Hot, Inferno, Turbo, Bone, Cool, Rainbow, Ocean, Pink or HSV (the OpenCV colormaps of those names) |
| Blur | number | `0` | Gaussian blur of the difference: kernel size, made odd; 0 = off. Number, or read from `msg`, `flow` or `global` |
| Threshold | number | `0` | 0-255: differences at or below it are set to 0, and only larger ones count as changed. Number, or read from `msg`, `flow` or `global` |

The status shows a green dot `OK: <n> img in <t> ms (conv <c> ms | task <k> ms)`, where `<n>` is the number of pairs, and a red ring `Error: <first 30 characters>` after a failure. Failures include `image1 and image2 must both be single images or both be arrays`, `Image arrays must be non-empty and of equal length (got <a> and <b>)`, `First image input is invalid or missing` and `Second image input is invalid or missing`.

## Examples

```
[camera] → [rp-image-align: reference golden] → [Change: msg.goldenImage = golden] → [rp-heat-diff: Threshold 25, Blur 5] → [rp-image-out]
```

Aligns each capture with the golden image, then saves a heat map of what changed; small differences from noise are suppressed.

```
[camera] → [rp-heat-diff: Heat Image off, SSIM off, Threshold 30] → [Switch: msg.diffStats.largestBlobArea > 200] → [reject]
```

Statistics only: rejects a part when a changed region larger than 200 pixels appears.

## Troubleshooting

- **The whole heat map is bright** — Cause: the images are not aligned, or differ in exposure. · Solution: align them first with [image-align](./image-align.md); raise Threshold.
- **`validRatio` is low** — Cause: large black areas (alignment borders, padding, dark backgrounds) are left out of the statistics. · Solution: expected for black borders; for dark scenes, keep in mind that pixels of value 0 are never compared.
- **`changedRatio` counts noise** — Cause: Threshold is 0, so any difference counts. · Solution: set Threshold, or Blur, to ignore small differences.
- **Statistics are slow** — Cause: SSIM costs more than all other statistics. · Solution: turn SSIM off when it is not used.
- **`Second image input is invalid or missing`** — Cause: Image 2 is not at `msg.goldenImage`. · Solution: point Image 2 at the `msg` property that holds the golden image.

## See Also

- [image-align](./image-align.md): align the two images before comparing them
- [blend](../blend/blend.md): overlay the heat map on the original image
- [cropBB](./cropBB.md): cut out regions for closer inspection
