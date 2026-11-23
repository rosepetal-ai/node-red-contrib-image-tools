---
title: "Color-Convert Node - Convert Between Color Spaces"
description: "Convert images in Node-RED between the GRAY, RGB, RGBA, BGR and BGRA color spaces, one image or an array. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, color-convert, color space, grayscale, rgb, bgr, rgba, alpha channel, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Color-Convert Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Convert images in Node-RED between the GRAY, RGB, RGBA, BGR and BGRA color spaces, one image or an array."
---

# Color-Convert Node

> **Type**: `rp-color-convert` · **Category**: Transform · **Since**: v1.2.2

## Overview

Color-Convert converts an image, or every image of an array, to another color space: grayscale, RGB, RGBA, BGR or BGRA. The pixels are converted with OpenCV (`cv::cvtColor`) and the `colorSpace` and `channels` of the result change with them. The size of the image does not change.

**Use cases**
- Turn camera images into grayscale before a filter or a model that expects one channel
- Hand BGR images to a library that expects RGB, or the opposite
- Add an alpha channel before compositing, or drop it before saving as JPEG

## Inputs

- **Input** (`msg.payload` by default; `msg`, `flow` or `global`): one image or an array of images. Each image is a raw image object (`colorSpace` `GRAY`, `RGB`, `RGBA`, `BGR` or `BGRA`) or an encoded JPEG/PNG/WebP/BMP `Buffer`, which the addon decodes as RGB, RGBA or GRAY. The images of an array are converted in parallel.

## Outputs

One output.

- **Output** (`msg.payload` by default): the converted image, or an array in the input order. A raw image object with `colorSpace` set to the target and `channels` 1, 3 or 4; or an encoded `Buffer` when **Output Format** is JPEG, PNG or WebP.
- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds.
- On failure: `msg.error` (`message`, `operation`, `hint`, `details`, `timestamp`, `nodeId`, `nodeName`, `nodeType`), with the input written back to **Output**; the message is still sent.

What each conversion does:

| From → To | Result |
|---|---|
| Same color space | The image unchanged |
| Color → GRAY | Luminance (weighted sum of R, G and B); the alpha channel is dropped |
| GRAY → color | The gray value copied to the three color channels; alpha, when added, is 255 |
| RGB ↔ BGR, RGBA ↔ BGRA | The red and blue channels swapped |
| 3 channels → 4 channels | An opaque alpha channel (255) added |
| 4 channels → 3 channels | The alpha channel dropped |

## Configuration

| Field | Type | Default | Description |
|---|---|---|---|
| Name | string | empty | Label of the node; also the key of its timings in `msg.performance.rpimage` |
| Input | TypedInput (msg/flow/global) | `msg.payload` | Where the image or array is read |
| Output | TypedInput (msg/flow/global) | `msg.payload` | Where the result is written |
| Output Format | enum | Raw (fastest) | Raw (fastest), JPEG, PNG or WebP |
| Quality | number | `90` | JPEG and WebP quality, 1-100 |
| PNG Optimize | boolean | off | **Compress PNG (slower, smaller file)**: PNG compression level 6 instead of 0 |
| Lossless | boolean | off | WebP: **Encode WebP without loss** |
| Smart Subsample | boolean | off | WebP: **Better chroma downsampling (lossy only)** |
| Effort | number | empty | WebP: 0-6. When Lossless, Smart Subsample or Effort is set, the WebP is encoded with `sharp` instead of OpenCV |
| Debug | boolean | off | **Show processed image**: a preview under the node in the editor (the first image of an array) |
| Debug Width | TypedInput (num/msg/flow/global) | `200` | Width of the preview in pixels |
| Target Color Space | enum | RGB | Grayscale (GRAY), RGB, RGBA, BGR or BGRA |

The status shows a green dot `OK: <n> img in <t> ms (conv <c> ms | task <k> ms)` after a conversion, and a red ring `Error: <first 30 characters>` after a failure.

## Examples

```
[rp-image-in] → [rp-color-convert: GRAY] → [rp-filter: Otsu Threshold] → [rp-image-out]
```

Converts a photo to one channel and binarizes it.

```
[camera (BGR)] → [rp-color-convert: RGB] → [function: send to a service that expects RGB]
```

Swaps the channel order of BGR camera images.

```
[rp-image-in] → [rp-color-convert: RGBA] → [rp-blend: Alpha compositing]
```

Adds an opaque alpha channel to an image before it is composited.

## Troubleshooting

- **`Invalid image structure` in `msg.error`** — Cause: the `colorSpace` of the input does not match its channel count (for example `RGB` with 4 channels), or the data length does not match `width × height × channels`. · Solution: check the node that produced the image; the warning logged before the error names the problem.
- **`Unsupported color space conversion: <from> -> <to>`** — Cause: the input `colorSpace` is not one of the five names in upper case. · Solution: set `colorSpace` to `GRAY`, `RGB`, `RGBA`, `BGR` or `BGRA`.
- **Transparency lost** — Cause: converting to GRAY, RGB or BGR drops the alpha channel, and so does JPEG output. · Solution: target RGBA or BGRA and save as PNG or WebP.

## See Also

- [filter](./filter.md): Otsu threshold, histogram equalization and other filters
- [padding](./padding.md): add margins, transparent on images with alpha
- [blend](../blend/blend.md): blend two images, with alpha compositing
