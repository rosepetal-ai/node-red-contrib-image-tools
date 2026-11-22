---
title: "Add-Masks Node - Polygon Masks Colored by Class"
description: "Blend several polygon masks over an image in Node-RED, with a color per class name and a global mask strength. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, add-masks, polygon masks, segmentation, class colors, object detection, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Add-Masks Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Blend several polygon masks over an image in Node-RED, with a color per class name and a global mask strength."
---

# add-masks

Applies several polygon or bitmap masks to a base image, each tinted with the colour of its class, with one mask strength for all.

## Description

The `add-masks` node extends [add-mask](./add-mask.md) to an array of masks. Each mask carries a class name; classes listed in **Class Color Mappings** get the colour set there, and the others get a colour picked automatically. Where masks overlap, only the mask of the class with the highest priority is painted.

This node is meant for showing segmentation results on the original image.

## Input Structure

The node reads an array of mask objects. It accepts the segmentation output of the Rosepetal inferencer node directly:

- **Class name** (required): `tag`, `class_name`, `className`, `label` or `class` (the first non-empty string)
- **Geometry** (one of):
  - `polygons`: array of polygons; each polygon is an array of `[x, y]` points in **normalized** coordinates (0‑1)
  - `mask`: a mask image `{data, width, height, channels, colorSpace, dtype}`, a 2D matrix of numbers, or an array of 2D matrices
- Any extra fields are ignored. If an object has valid `polygons`, they are used and `mask` is ignored.

Mask images and matrices are made binary (any non-zero value is inside the mask; for a 4-channel image the alpha channel is used, for a 3-channel image its gray value) and scaled to the size of the base image with nearest-neighbour interpolation.

**Example (direct inferencer output)**
```javascript
msg.array = [
  {
    tag: "bottle",
    polygons: [[[0.1, 0.1], [0.9, 0.1], [0.9, 0.8], [0.1, 0.8]]]
  },
  {
    class_name: "person",
    mask: {
      width: 512,
      height: 512,
      channels: 1,
      colorSpace: "GRAY",
      dtype: "uint8",
      data: <Buffer ...>
    }
  }
];
```

## Configuration

### Paths
Only `msg` properties are read and written: the editor also offers `flow` and `global`, but the node reads the typed text as a `msg` path.
- **Image**: the base image, a raw image object or an encoded JPEG/PNG/WebP/BMP `Buffer` (default: `msg.image`)
- **Masks Array**: the array of masks (default: `msg.array`)
- **Output**: where the result is written (default: `msg.payload`), with the size and color space of the base image

### Processing Options
- **Global Mask Strength**: 0-100 % (default 50): inside a mask each pixel becomes image × (100 − strength) + class colour × strength
- **Output Format**: Raw (fastest), JPEG, PNG or WebP, with Quality (`90`), PNG Optimize and the WebP options as in [resize](../transform/resize.md#output-format-options)
- **Debug / Debug Width**: **Show processed image** under the node in the editor (default width 200)

The editor has no Name field for this node, so its timings are recorded under `msg.performance.rpimage.rp_add_masks`.

### Dynamic Class Color Mapping

**Class Color Mappings** is a list of class names, each with a colour:

1. Click **Add Class Mapping** to create an entry (red by default)
2. Enter the class name (e.g., "dog") exactly as it appears in the masks
3. Pick the colour with the colour picker
4. Remove an entry with its trash button; drag the ☰ handle to reorder

The editor warns about duplicate class names and about colours that are very close to each other.

## Color Assignment Priority

1. **User-defined mapping**: the colour set in Class Color Mappings
2. **Automatic**: for other classes, the colour of a fixed palette of 216 colours that differs most from the colours already in use. The colour chosen for a class name is kept for as long as Node-RED runs, so a class keeps its colour from message to message

## Mask Priority System

When several masks cover the same pixel, only **one mask is painted**:

1. Classes listed in Class Color Mappings come first, in the order of the list (top = highest priority).
2. Unmapped classes come after all mapped classes, in the order of the input array.

### Example

Configuration:
```
Class Color Mappings (drag to reorder):
  1. "person"  → #FF0000 (red)    ← Highest priority
  2. "car"     → #00FF00 (green)
  3. "dog"     → #0000FF (blue)   ← Lowest mapped priority
```

If a pixel is covered by both "person" and "car" masks, only the **red (person)** colour is painted. If a pixel is covered by "cat" and "bird" (both unmapped), the one appearing first in the input array is painted.

## Outputs, Status and Errors

- **Output**: the image with the masks; `msg.performance.rpimage.rp_add_masks`: `{ conversion, task, total }` in milliseconds
- **Status**: green dot `OK: <n> img in <t> ms (conv <c> ms | task <k> ms)`, where `<n>` is the number of polygons and masks drawn
- **Empty array**: the base image is written to Output unchanged and the status shows `OK: 0 img ...`
- **Validation**: the whole message fails if any entry is invalid. The node sets `msg.error` (with `validationError: true`), writes the base image to Output, logs a warning, shows a red ring `Error: <first 30 characters>` and still sends the message. Messages include `Masks input must be an array`, `Base image is invalid or missing`, `Mask at index <i> is missing a valid class/tag field`, `Mask at index <i> must include either 'polygons' or 'mask'`, `Invalid coordinate at index <i>.polygons[<j>][<k>]: coordinates must be in range [0, 1]` and `Invalid mask data at index <i>: expected raw mask image or 2D matrix`

## Examples

### Segmentation Visualization

**Input:**
```javascript
msg.image = originalImage;
msg.array = [
  { tag: "dog", polygons: [[[0.1,0.1], [0.3,0.1], [0.3,0.3], [0.1,0.3]]] },
  { tag: "cat", polygons: [[[0.5,0.5], [0.7,0.5], [0.7,0.7], [0.5,0.7]]] }
];
```

**Configuration:**
- dog → Red (#FF0000)
- cat → Blue (#0000FF)
- Global Mask Strength: 70

**Result:** the image with a red square for the dog and a blue square for the cat.

### Many Classes with Automatic Colours

Define colours only for the classes that matter; the others get distinct automatic colours and are painted below the mapped classes where they overlap.

## Troubleshooting

**"Mask at index 0 is missing a valid class/tag field"**
- Every mask object needs a non-empty `tag`, `class_name`, `className`, `label` or `class`

**"Invalid coordinate ... coordinates must be in range [0, 1]"**
- Polygons must be normalized; divide pixel coordinates by the image width and height first

**"Masks input must be an array"**
- Point **Masks Array** at the array itself (default `msg.array`), not at the object that contains it

**Colours change after a restart**
- Automatic colours depend on the order in which classes first appear; map the important classes to fixed colours

### Performance Tips
- Use raw format for processing chains (fastest)
- Masks are drawn and blended in the native OpenCV addon, on the libuv thread pool; a mask image of the same size as the base image avoids a resize

## Node Interactions

- **Input sources**: segmentation models (for example the Rosepetal inferencer), [advanced-mosaic](../mix/advanced-mosaic.md) with **Propagate masks** (its masks carry a `tag` only when the input masks had a class name)
- **Output destinations**: [image-out](../io/image-out.md), [concat](../mix/concat.md)
- For boxes instead of masks, use [add-bbs](./add-bbs.md)
