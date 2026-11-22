---
title: "Add-BBs Node - Draw Bounding Boxes with Labels"
description: "Draw the bounding boxes of a detection result on an image in Node-RED, labelled with their tag and confidence, with a color per tag. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, add-bbs, bounding box, object detection, draw boxes, labels, confidence, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Add-BBs Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Draw the bounding boxes of a detection result on an image in Node-RED, labelled with their tag and confidence, with a color per tag."
---

# Add-BBs Node

> **Type**: `rp-add-bbs` · **Category**: Blend · **Since**: v1.1.3

## Overview

Add-BBs draws the boxes of an object detection result on the image: the outline of each box in the colour of its tag, and a label with the tag and the confidence. Boxes may be rotated. For segmentation masks instead of boxes, use [add-masks](./add-masks.md); to cut the boxes out as separate images, use [cropBB](../specialized/cropBB.md).

**Use cases**
- Show what a detection model found, on a dashboard or in saved images
- Keep an annotated copy of every inspected part
- Highlight only some classes (for example defects) with **Only Draw Mapped Tags**

## Inputs

Only `msg` properties are read: the editor also offers `flow` and `global`, but the node reads the typed text as a `msg` path.

- **Image** (`msg.payload.image` by default): a raw image object or an encoded JPEG/PNG/WebP/BMP `Buffer`.
- **Boxes Array** (`msg.payload` by default): an array of detections. Each one needs:
  - `tag`: a non-empty string, the class of the box
  - a box in normalized coordinates (0-1), in one of these forms: `raw_boxes` (four `[x, y]` corner points), `4points` (`{ x0, y0, x1, y1, x2, y2, x3, y3 }`), `xywhr` (`{ x, y, w, h, r }`, top-left corner and rotation in degrees), `cwh` (`{ x, y, w, h, r }`, centre) or `2points` (`{ xmin, ymin, xmax, ymax }`). When the object has no `raw_boxes`, the node adds them to it, computed from the other form.
  - `confidence` (optional): 0-1; without it the label shows 100.0%.

The default paths (`msg.payload.image` and `msg.payload`) cannot both hold data in a normal message, so set at least one of them. For example, with **Image** `msg.image` and **Boxes Array** `msg.boxes`:

```javascript
msg.image = image;   // raw image object or encoded Buffer
msg.boxes = [
  { tag: "scratch", confidence: 0.91, "2points": { xmin: 0.12, ymin: 0.30, xmax: 0.25, ymax: 0.41 } },
  { tag: "dent",    confidence: 0.67, raw_boxes: [[0.6,0.2],[0.8,0.2],[0.8,0.5],[0.6,0.5]] }
];
```

## Outputs

One output.

- **Output** (`msg.payload` by default; a `msg` property): the image with the boxes, with the size and color space of the input, as a raw image object or an encoded `Buffer` (JPEG, PNG, WebP).
- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds.
- On failure: `msg.error` (`message`, `operation`, `hint`, `details`, `timestamp`, `nodeId`, `nodeName`, `nodeType`, and `validationError: true` for invalid input), with the input image written to **Output**; the message is still sent.

## Configuration

| Field | Type | Default | Description |
|---|---|---|---|
| Name | string | empty | Label of the node; also the key of its timings in `msg.performance.rpimage` |
| Image | msg path | `msg.payload.image` | The image to draw on |
| Boxes Array | msg path | `msg.payload` | The detections |
| Output | msg path | `msg.payload` | Where the result is written |
| Output Format | enum | Raw (fastest) | Raw (fastest), JPEG, PNG or WebP |
| Quality | number | `90` | JPEG and WebP quality, 1-100 |
| PNG Optimize | boolean | off | PNG compression level 6 instead of 0 |
| Lossless / Smart Subsample / Effort | boolean / boolean / number | off / off / empty | WebP options; when any is set the WebP is encoded with `sharp` |
| Debug / Debug Width | boolean / number | off / `200` | **Show processed image** under the node in the editor |
| Show Tag Name | boolean | on | Put the tag in the label |
| Show Confidence | boolean | on | Put the confidence in the label, as a percentage with one decimal |
| Only Draw Mapped Tags | boolean | off | Draw only the boxes whose tag is in Tag Color Mappings |
| Box Thickness | number | `2` | 1-10, scaled with the image size |
| Font Size | number | `0.5` | 0.3-2.0, scaled with the image size |
| Tag Color Mappings | list | empty | Tag names with a colour each, added with **Add Tag Mapping** |

### Drawing

- Each box is drawn as a closed outline through its four corners, so rotated boxes are drawn rotated.
- The label (`scratch (91.0%)`, `scratch` or `91.0%`, depending on the two Show options) is written in black on a filled rectangle of the box colour, above the topmost corner of the box. With both Show options off, no label is drawn.
- Line thickness and text size grow with the image: the thickness in pixels is about Box Thickness × √(width × height) / 1000, at least 1.
- Tags without a mapping get an automatic colour: the colour of a fixed palette of 216 that differs most from those already in use, kept for that tag as long as Node-RED runs.

### Status

- Green dot `OK: <n> img in <t> ms (conv <c> ms | task <k> ms)`, where `<n>` is the number of boxes drawn.
- An empty array writes the image to Output unchanged, with `OK: 0 img ...`.
- Red ring `Error: <first 30 characters>` on failure.

## Examples

```
[camera] → [inference] → [Function: msg.image = image; msg.boxes = detections] → [rp-add-bbs: Image msg.image, Boxes Array msg.boxes] → [rp-image-out]
```

Saves every inspected image with its detections drawn.

```
[detector] → [rp-add-bbs: Only Draw Mapped Tags, scratch → #ff0000, dent → #ffaa00] → [dashboard image]
```

Shows only the two defect classes, each in a fixed colour.

## Troubleshooting

- **`Boxes input must be an array`** — Cause: **Boxes Array** points at the object that contains the detections, or at nothing. · Solution: point it at the array itself.
- **`Box coordinates out of range [0,1] at index <i>`** — Cause: the box is in pixels. · Solution: divide by the image width and height first.
- **`Invalid tag at index <i>: must be a non-empty string`** or **`Empty tag at index <i>: tag cannot be empty`** — Cause: a detection has no `tag`, or it is not a string. · Solution: copy the class name into `tag` (for example from `class_name`).
- **`Invalid box object at index <i>: 'raw_boxes' property must be an array`** — Cause: the detection has none of the box forms listed above. · Solution: add `raw_boxes` or one of the other forms.
- **Boxes missing** — Cause: **Only Draw Mapped Tags** is on and the tag is not in the mappings (names must match exactly). · Solution: add the tag or turn the option off.

## See Also

- [add-masks](./add-masks.md): draw segmentation masks coloured by class
- [cropBB](../specialized/cropBB.md): cut each box out as its own image
- [draw](../transform/draw.md): draw points and lines
