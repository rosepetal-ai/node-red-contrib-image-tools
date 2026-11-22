---
title: "Add-Mask Node - Polygon Mask Overlay"
description: "Blend a polygon mask over an image in Node-RED from normalized polygon coordinates, with a fill color and a mask strength. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, add-mask, polygon mask, overlay, segmentation, region of interest, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Add-Mask Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Blend a polygon mask over an image in Node-RED from normalized polygon coordinates, with a fill color and a mask strength."
---

# Add-Mask Node

## Purpose & Use Cases

The `add-mask` node tints the inside of one polygon on an image: within the polygon, the image is mixed with a fill colour by a mask strength. The polygon is given in normalized coordinates, as segmentation models return it. For several polygons coloured by class, use [add-masks](./add-masks.md).

**Real-World Applications:**
- **Segmentation Display**: Show the region a model found on the original image
- **ROI Highlighting**: Mark the region of interest of an inspection
- **Quality Control**: Overlay a detected defect area on the product image

## Input/Output Specification

### Inputs
- **Image**: the base image, read from `msg.payload.image` by default: a raw image object or an encoded JPEG/PNG/WebP/BMP `Buffer`
- **Polygon Path**: one polygon, read from `msg.payload.default.masks[0].mask[0]` by default: an array of `[x, y]` pairs with values from 0 to 1
- Only `msg` properties are read: the editor also offers `flow` and `global`, but the node reads the typed text as a `msg` path

### Outputs
One output:
- **Masked Image**: written to the `msg` property **Output** (`msg.payload` by default), with the size and color space of the input, as a raw image object or an encoded `Buffer` (JPEG/PNG/WebP)
- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds

## Configuration Options

| Property | Default | Meaning |
|----------|---------|---------|
| Image | `msg.payload.image` | Base image |
| Polygon Path | `msg.payload.default.masks[0].mask[0]` | The polygon |
| Output | `msg.payload` | Where the result is written |
| Output Format | Raw (fastest) | Raw (fastest), JPEG, PNG or WebP, with Quality (`90`), PNG Optimize and the WebP options as in [resize](../transform/resize.md#output-format-options) |
| Debug / Debug Width | off / `200` | **Show processed image** under the node in the editor |
| Mask Strength | `50` | 0-100 %: weight of the fill colour inside the polygon |
| Fill Color | `#ffffff` | Colour mixed into the polygon |

The editor has no Name field for this node, so its timings are recorded under `msg.performance.rpimage.rp_add_mask`.

### Polygon Mask Options

#### Coordinate System
- **Normalized Coordinates**: every `x` and `y` must be a number from 0 to 1
- **Format**: `[[x1,y1], [x2,y2], [x3,y3], ...]`
- **Relative Positioning**: 0.0 = left/top edge, 1.0 = right/bottom edge

#### Mask Strength
- **Effect**: inside the polygon each pixel becomes image × (100 − strength) + fill colour × strength; outside it the image is unchanged
- **Range**: 1-100. A value of 0 is read as the default 50

#### Fill Color
- **Format**: picked with the editor's colour picker (`#RRGGBB`)
- **Grayscale images**: the mean of the red, green and blue values is used

The polygon is filled with hard edges (no anti-aliasing).

### Output Format Options
- **Raw**: Standard image object (fastest between image nodes)
- **JPEG**: Compressed; the alpha channel of an RGBA input is dropped
- **PNG**, **WebP**: keep the alpha channel of an RGBA input

## Status and Errors

- **Success**: green dot `OK: 1 img in <t> ms (conv <c> ms | task <k> ms)`, plus the preview format with Debug on
- **Failure**: `msg.error` is set (with `validationError: true` for invalid input), **Output** is left as it was, a warning is logged, the status shows a red ring `Error: <first 30 characters>` and the message is still sent. Messages: `Base image is invalid or missing`, `Polygon coordinates must be an array`, `Polygon coordinates array is empty`, `Invalid polygon coordinate at index <i>: expected [x, y] pair`, `...: coordinates must be numbers`, `...: coordinates must be in range [0, 1]`

## Performance Notes

- The mask is drawn and blended in the native OpenCV addon, on the libuv thread pool

## Real-World Examples

### Segmentation Highlighting
```javascript
// Function node before add-mask
msg.payload = {
  image: msg.image,
  default: { masks: [ { mask: [ [[0.1, 0.2], [0.8, 0.2], [0.8, 0.7], [0.1, 0.7]] ] } ] }
};
return msg;
```
```
[Image + segmentation] → [Function] → [Add-Mask: Mask Strength 30, Fill Color #ff0000] → [Highlighted region]
```

### Inspection Region
```
[Product Image] → [Change: msg.roi = [[0.25,0.25],[0.75,0.25],[0.75,0.75],[0.25,0.75]]] → [Add-Mask: Polygon Path msg.roi, Image msg.payload] → [Inspection Overlay]
```
Mark the centre of the image in yellow (Fill Color `#ffff00`, Mask Strength 50).

## Common Issues & Troubleshooting

### Invalid Polygon Coordinates
- **Issue**: `Invalid polygon coordinate at index ...` in `msg.error`
- **Solution**: Make the polygon an array of `[x, y]` pairs of numbers between 0 and 1; pixel coordinates must be divided by the image width and height first

### Incorrect Path Configuration
- **Issue**: `Base image is invalid or missing` or `Polygon coordinates must be an array`
- **Solution**: Point **Image** and **Polygon Path** at the `msg` properties that hold them
- **Default Paths**: `msg.payload.image` and `msg.payload.default.masks[0].mask[0]`

### Weak or Invisible Mask Effects
- **Issue**: Mask blending barely visible on image
- **Solution**: Increase Mask Strength or choose a Fill Color that contrasts with the image
