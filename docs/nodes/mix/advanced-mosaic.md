---
title: "Advanced Mosaic Node - Canvas Layout with Resize and Rotation"
description: "Compose the images of an array on a canvas in Node-RED with a position, size, rotation and stacking order per image. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, advanced mosaic, canvas, image layout, rotation, layering, mask propagation, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Advanced Mosaic Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Compose the images of an array on a canvas in Node-RED with a position, size, rotation and stacking order per image."
---

# Advanced Mosaic Node

## Purpose & Use Cases

The `advanced-mosaic` node composes the images of an array on a canvas of a fixed size. For each placement it sets a position, an optional new size, a rotation and a stacking order, laid out in a visual editor. Unlike [mosaic](./mosaic.md), images can be resized and rotated, and rotated images are blended over what is below them. It can also carry a mask per image through the same steps.

**Real-World Applications:**
- **Inspection Reports**: Place a full view and enlarged details of a part on one image
- **Multi-Camera Views**: Combine the images of several cameras, scaled and turned to a common orientation
- **Overlay Compositions**: Stack images in a chosen order
- **Segmentation Output**: Move the masks of each image to the composed canvas together with the image

## Input/Output Specification

### Inputs
- **Image Array**: an array of raw image objects or encoded JPEG/PNG/WebP/BMP `Buffer`s, read from **Input from** (`msg.payload` by default). A single image is treated as an array of one.
- **Placements**: set in the node's **Visual Canvas Editor**, not taken from the message.
- **Canvas Width / Canvas Height**: a number, or read from `msg`, `flow` or `global` for each message.
- **Masks** (optional, with **Propagate masks**): an array aligned with the images, read from **Masks from** (`msg.masks` by default). See [Mask Propagation Input Format](#mask-propagation-input-format).

### Outputs
One output:
- **Composite Image**: one image of Canvas Width × Canvas Height, written to **Output to** (`msg.payload` by default), as a raw image object or an encoded `Buffer` (JPEG/PNG/WebP)
- **Masks** (with **Propagate masks**): written to **Masks to** (`msg.masks` by default)
- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds

## Configuration Options

| Property | Default | Meaning |
|----------|---------|---------|
| Name | empty | Label of the node; also the key of its timings in `msg.performance.rpimage` |
| Input from | `msg.payload` | Where the array is read (`msg`, `flow` or `global`) |
| Output to | `msg.payload` | Where the composite is written (`msg`, `flow` or `global`) |
| Propagate masks | off | Transform a mask per image together with its image |
| Masks from | `msg.masks` | Where the masks are read |
| Masks to | `msg.masks` | Where the transformed masks are written |
| Output Format | Raw (fastest) | Raw (fastest), JPEG, PNG or WebP, with Quality (`90`), PNG Optimize and the WebP options as in [resize](../transform/resize.md#output-format-options) |
| Debug / Debug Width | off / `200` | **Show processed image** under the node in the editor |
| Canvas Width | `800` | Width of the canvas in pixels; **Apply Canvas Size** updates the visual editor |
| Canvas Height | `600` | Height of the canvas in pixels |
| Background Color | `#000000` | Colour of the canvas where no image is placed (opaque) |
| Visual Canvas Editor | one placement | The placements; a new node starts with image 0 at (50, 50), resized to 200 × 150 |
| Inference | off | Under **Advanced options**: **Transform inference coordinates**, with **Inf. Input** and **Inf. Output** (both `msg.inference`) |

### Visual Canvas Editor
Each placement is a rectangle on a scaled view of the canvas. Drag it to move it; **Add Image**, **Remove Selected**, **Duplicate**, **Bring to Front** and **Send to Back** manage the placements, and **Show Grid**, **Snap to Grid**, **Magnetic Snap** and **Lock Aspect Ratio** help the layout. The properties panel of the selected placement has:

| Field | Meaning |
|-------|---------|
| Array Index | Which image of the input array is placed (0 = first). A placement whose index is outside the array is ignored |
| Resize Width / Resize Height | New size in pixels. Empty keeps the original size; with only one of them, the other follows the aspect ratio. Bilinear interpolation |
| Rotation (°) | Rotation after the resize, around the image centre, on a canvas grown to fit the rotated image. See the note below on the direction |
| X Position / Y Position | Where the top-left corner of the (resized and rotated) image goes, in pixels. The part outside the canvas is cut off |
| Z-Index | Stacking order: lower values are drawn first, higher values on top |

The editor assumes an input of 800 × 600 for its preview (**Input Width** and **Input Height** are read-only); at runtime the real size of each image is used.

**Rotation direction**: positive angles turn the image clockwise, as in the editor's preview, except exactly 90° and 270°: 90° turns it counter-clockwise and 270° clockwise. Use for example 89.9° or −90° for a quarter turn clockwise.

### Canvas and Blending
- When any placement has a rotation, the canvas is RGBA (and opaque). The corners uncovered by each rotation are transparent, so the background and the images below show through; images are blended over the canvas with their alpha channel.
- Without rotations, the canvas takes the color space of the inputs (RGBA if any image is RGBA, else BGRA, RGB, BGR and GRAY, in that order) and images replace the pixels below them.

### Output Format Options
- **Raw**: Standard image object (fastest between image nodes)
- **JPEG**: Compressed; the alpha channel is dropped
- **PNG**: Lossless, keeps the alpha channel
- **WebP**: Keeps the alpha channel

## Mask Propagation Input Format

When *Propagate masks* is checked, supply `msg.masks` (or the configured path) as an **array aligned with the input images array**: index `i` is the mask of image `i`. Each entry can be one of:

- **Polygon mask**: `{ tag?: "class", polygons: [ [ [x,y], [x,y], ... ], ... ] }` where points are normalized 0–1 in the image.
- **Mask image**: `{ mask: { data, width, height, channels, colorSpace, dtype }, tag?: "class" }`, an encoded image `Buffer` in `mask`, or the raw image object itself.
- **2D matrix mask**: `{ mask: [[0,1,1,...], ...], tag?: "class" }` or the 2D numeric matrix itself.

Rules:
- If the masks input is not an array, the node warns `Mask propagation enabled but masks input is not an array; skipping mask propagation.` and composes the image only.
- Entries after the last image, and entries of another shape, are ignored.
- The label is read from `tag`, `class_name`, `className`, `label` or `class` and returned as `tag`.
- Masks are made binary (any non-zero value becomes 255; color masks are converted to gray first), resized with nearest-neighbour interpolation, rotated and positioned exactly like their image.

Output: an array of `{ mask, tag? }`, where `mask` is a single-channel (`GRAY`) raw image of the canvas size. Only images that have a mask and are placed produce an entry, so the output array is not aligned with the input images when some have none. An image placed several times has all its placements in the same mask.

### Inference
With **Transform inference coordinates** checked, **Inf. Input** must hold an array with one entry per input image, each the inference array of that image. Detections and segmentations are mapped through the resize, rotation and position of each placement of their image (clipped to the canvas) and the node writes a single flat array to **Inf. Output**.

## Status and Errors

- **Success**: green dot `OK: <n> img in <t> ms (conv <c> ms | task <k> ms)`, where `<n>` is the number of placements, with `/<m> mask` after it when masks were propagated; `OK: empty canvas in <t> ms` when no placement applies. With Debug on: `OK: 1 img in ...` plus the preview format
- **Failure**: `msg.error` is set, the **first** input image is written to **Output to**, a warning is logged, the status shows a red ring `Error: <first 30 characters>` and the message is still sent. An invalid array gives `Invalid image list`; a canvas size of 0 or less gives `Canvas dimensions must be positive numbers`

## Performance Notes

- Resizing, rotation, blending and encoding run in one call to the native OpenCV addon, on the libuv thread pool
- Rotations other than multiples of 90° need interpolation and a larger intermediate image

## Real-World Examples

### Overview and Detail
Placements (canvas 1200 × 600, input array: full view, close-up):

| Array Index | Resize Width | Resize Height | Rotation (°) | X Position | Y Position | Z-Index |
|-------------|--------------|---------------|--------------|------------|------------|---------|
| 0 | 800 | | 0 | 0 | 0 | 0 |
| 1 | 400 | | 0 | 800 | 0 | 1 |

```
[Camera A] → [Array-In: pos=0] ┐
[Camera B] → [Array-In: pos=1] ├→ [Array-Out] → [Advanced Mosaic] → [Image-Out]
```
Array-In and Array-Out are part of [`@rosepetal/node-red-contrib-utils`](https://docs.rosepetal.ai/flows/node-red-contrib-utils/nodes/io/array-in).

### Masks Follow the Layout
```
[Images + segmentation] → [Function: msg.masks = one { polygons, tag } per image] → [Advanced Mosaic: Propagate masks] → [Add-Masks]
```
The composed image and the masks in canvas coordinates leave the node together.

## Common Issues & Troubleshooting

### Image Not Visible
- **Issue**: A placement does not appear
- **Solution**: Check that its **Array Index** exists in the input array, that its position is inside the canvas, and that no placement with a higher **Z-Index** covers it

### Quarter Turn in the Wrong Direction
- **Issue**: A rotation of exactly 90° or 270° turns the opposite way to the preview
- **Solution**: See the note on rotation direction above; use −90° (or 270°) for clockwise

### Four Channels in the Output
- **Issue**: The output is RGBA although the inputs are RGB
- **Cause**: Any rotation makes the canvas RGBA (it stays opaque)
- **Solution**: Convert to RGB with [color-convert](../transform/color-convert.md) if a 3-channel image is needed

### Masks Out of Step with the Images
- **Issue**: Mask `i` of the output does not belong to image `i`
- **Cause**: Images without a mask, or not placed, produce no entry
- **Solution**: Use the `tag` of each mask, or give every image a mask
