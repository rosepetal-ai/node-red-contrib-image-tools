---
title: "Blend Node - Combine Two Images"
description: "Blend two images in Node-RED with an adjustable opacity, with optional alpha compositing and background color removal. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, blend, opacity, alpha compositing, overlay, watermark, background removal, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Blend Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Blend two images in Node-RED with an adjustable opacity, with optional alpha compositing and background color removal."
---

# Blend Node

## Purpose & Use Cases

The `blend` node combines two images into one. In the default mode it mixes them with a weight (a cross-fade); with **Alpha Compositing** it lays Image 2 over Image 1 using the alpha channel of Image 2, and can first make one background colour of Image 2 transparent.

**Real-World Applications:**
- **Watermark Application**: Lay a semi-transparent logo (with alpha) over photos
- **Background Replacement**: Remove a uniform background colour from a subject and lay it over a new background
- **Before/After Overlays**: Cross-fade an image with its processed version
- **Visual Inspection**: Overlay a heat map or mask image on the original

## Input/Output Specification

### Inputs
- **Image 1**: the base image, read from `msg.payload.image1` by default
- **Image 2**: the overlay image, read from `msg.payload.image2` by default
- Each is a raw image object or an encoded JPEG/PNG/WebP/BMP `Buffer`. Only `msg` properties are read: the editor also offers `flow` and `global`, but the node reads the typed text as a `msg` path.

### Outputs
One output:
- **Blended Image**: written to the `msg` property **Output** (`msg.payload` by default) as a raw image object or an encoded `Buffer` (JPEG/PNG/WebP)
- **Dimensions**: when the images differ in size, both are scaled (without keeping the aspect ratio) to the largest width and the largest height of the two
- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds

## Configuration Options

| Property | Default | Meaning |
|----------|---------|---------|
| Image 1 | `msg.payload.image1` | Base image |
| Image 2 | `msg.payload.image2` | Overlay image |
| Output | `msg.payload` | Where the result is written |
| Output Format | Raw (fastest) | Raw (fastest), JPEG, PNG or WebP, with Quality (`90`), PNG Optimize and the WebP options as in [resize](../transform/resize.md#output-format-options) |
| Debug / Debug Width | off / `200` | **Show processed image** under the node in the editor |
| Alpha Compositing | off | **Use Image 1 as base layer**: lay Image 2 over Image 1 with its alpha channel |
| Background Removal | off | *Alpha Compositing only.* **Remove background from overlay image** |
| Background Color | `#ffffff` | *Background Removal only.* The colour made transparent in Image 2 |
| Color Tolerance | `10` | *Background Removal only.* 0-100 %: how far a pixel's colour may be from Background Color to be removed |
| Opacity | `50` | 0-100 %: see below |

The editor has no Name field for this node, so its timings are recorded under `msg.performance.rpimage.rp_blend`.

An **Opacity** or **Color Tolerance** of 0 is read as the default (50 and 10): use 1 for the lowest value.

### Blending Modes

#### Weighted Blend (Alpha Compositing off)
- **Result**: Image 1 × Opacity + Image 2 × (100 − Opacity). At 100 only Image 1 is visible; at 50 both count the same
- **Color space**: both images are converted to the one with the most channels (RGBA, then BGRA, RGB, BGR and GRAY)

#### Alpha Compositing
- **Result**: Image 2 over Image 1 ("over" operator), with the alpha of Image 2 multiplied by Opacity. At 100 the overlay is fully applied; low values make it fainter
- **Output**: always 4 channels (`BGRA`). Images without alpha count as opaque, so an RGB overlay at Opacity 100 covers the base completely

#### Background Removal
- **Operation**: pixels of Image 2 whose hue, saturation and value are all within Color Tolerance of Background Color become transparent, with a slight blur of the mask edge (above 1 % tolerance). The existing alpha channel of Image 2 is replaced
- **Note**: the current version swaps the red and blue components of Background Color before matching. White, black, grays, pure green and magenta are matched as expected; for other colours, enter the colour with red and blue exchanged

### Output Format Options
- **Raw**: Standard image object (fastest between image nodes)
- **JPEG**: Compressed; the alpha channel is dropped
- **PNG**: Lossless, keeps the alpha channel
- **WebP**: Keeps the alpha channel

## Status and Errors

- **Success**: green dot `OK: 1 img in <t> ms (conv <c> ms | task <k> ms)`, plus the preview format with Debug on
- **Failure**: `msg.error` is set, Image 1 is written to **Output**, a warning is logged, the status shows a red ring `Error: <first 30 characters>` and the message is still sent. Missing or invalid images give `First image is invalid or missing` or `Second image is invalid or missing`

## Performance Notes

- Conversion, scaling, background removal and blending run in one call to the native OpenCV addon, on the libuv thread pool

## Real-World Examples

### Watermark Application
```
[Function: msg.payload = { image1: photo, image2: logo }] → [Blend: Alpha Compositing, Opacity 30] → [Watermarked Photo]
```
The logo is scaled to the size of the photo, so it should already be a full-size transparent layer with the logo where it must appear.

### Background Replacement
```
[New Background] → image1 ┐
[Subject on white] → image2 ├→ [Blend: Alpha Compositing, Background Removal #ffffff, Tolerance 15, Opacity 100] → [Composite]
```
Make the white background of the subject transparent and lay it over the new background.

### Cross-Fade
```
[Original] → image1 ┐
[Filtered] → image2 ├→ [Blend: Opacity 70] → [70 % original, 30 % filtered]
```

## Common Issues & Troubleshooting

### The Overlay Covers the Base Completely
- **Issue**: With Alpha Compositing only Image 2 is visible
- **Cause**: Image 2 has no alpha channel (or it is fully opaque) and Opacity is 100
- **Solution**: Lower Opacity, use an overlay with alpha, or turn on Background Removal

### Transparency Lost
- **Issue**: Transparent areas become black or white
- **Solution**: Use PNG or WebP output; JPEG drops the alpha channel

### Background Removal Artifacts
- **Issue**: Incomplete or over-aggressive background removal
- **Solution**: Adjust Color Tolerance; check the note on red and blue above for coloured backgrounds

### Stretched Images
- **Issue**: One image is distorted
- **Cause**: Images of different sizes are both scaled to the largest width and height
- **Solution**: Resize or pad the images to the same size first

### Images from Context Are Ignored
- **Issue**: Image 1 or Image 2 set to `flow` or `global` is not found
- **Solution**: Copy the image into the message first (Change node); this node reads `msg` properties only
