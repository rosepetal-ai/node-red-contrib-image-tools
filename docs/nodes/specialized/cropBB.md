---
title: "CropBB Node - Crop Detection Bounding Boxes"
description: "Extract one image crop per bounding box of an inference result in Node-RED, filtered by a minimum confidence. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, cropBB, bounding box, object detection, crop, inference, confidence, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "CropBB Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Extract one image crop per bounding box of an inference result in Node-RED, filtered by a minimum confidence."
---

# CropBB Node (Crop Bounding Boxes)

## Purpose & Use Cases

The `cropBB` node cuts one crop out of an image for every box of an object detection result whose confidence reaches a minimum. Each crop keeps the tag, the confidence and the position of its box, so the crops can go straight to a classifier or be saved one by one.

**Real-World Applications:**
- **Two-Stage Inspection**: Detect parts or defects, then classify each crop with a second model
- **Quality Control Systems**: Save the defective areas found by an inspection model
- **Dataset Building**: Collect crops of detected objects for training

## Input/Output Specification

### Inputs
Only `msg` properties are read and written: the editor also offers `flow` and `global`, but the node reads the typed text as a `msg` path.

- **Input from** (`msg.image` by default): a raw image object. An array is accepted, but only its first image is cropped. Encoded `Buffer`s are rejected (`cropBB requires raw images with width/height metadata`)
- **Bounding Boxes** (`msg.default` by default): the array of detections
- **Min Confidence**: detections below it are skipped

### Detection Data Format
```javascript
[
  {
    "raw_boxes": [[x1,y1], [x2,y1], [x2,y2], [x1,y2]], // normalized 0-1, axis-aligned
    "confidence": 0.97,
    "tag": "person"
  },
  // ... more detections
]
```

- `raw_boxes` must be four corners in the order top-left, top-right, bottom-right, bottom-left of an axis-aligned box; other shapes are skipped with the warning `Inconsistent corner points in detection raw_boxes: ...` or `Invalid raw_boxes format in detection: ...`
- `confidence` is optional; a detection without it (or with 0) counts as 1.0
- `tag` is optional; without it the crop is tagged `unknown`

### Outputs
One output. **Output to** (`msg.payload` by default) receives an array with one crop per kept detection, in the order of the detections:

```javascript
[
  {
    data: Buffer, width: 200, height: 300, channels: 3, colorSpace: "RGB", dtype: "uint8",
    tag: "person",          // tag of the detection
    confidence: 0.97,       // confidence of the detection
    bbox: {                 // the crop in pixels of the source image
      x: 150, y: 100, width: 200, height: 300,
      normalized: { x1: 0.1, y1: 0.2, x2: 0.3, y2: 0.5 }
    }
  },
  // ... more crops
]
```

`msg.performance.rpimage.<key>` receives `{ conversion, task, total }` in milliseconds.

## Configuration Options

| Property | Default | Meaning |
|----------|---------|---------|
| Name | empty | Label of the node; also the key of its timings in `msg.performance.rpimage` |
| Input from | `msg.image` | The source image |
| Bounding Boxes | `msg.default` | The detections |
| Output to | `msg.payload` | Where the array of crops is written |
| Output Format | Raw (fastest) | Keep **Raw**: see the note below |
| Quality, PNG Optimize, WebP options | `90`, off, off | Encoding options of the output formats |
| Debug / Debug Width | off / `200` | **Show processed image**: a preview of the first crop under the node in the editor |
| Min Confidence | `0.5` | 0.0 to 1.0: a number, or read from `msg`, `flow` or `global`. A value of 0 is read as 0.5 |

**Output Format**: with JPEG, PNG or WebP the current version does not return usable crops (the encoded data is not kept as a `Buffer` next to the tag, and with the WebP options the tag, confidence and bbox are lost). Use Raw and encode the crops later, for example with [image-out](../io/image-out.md).

### Coordinate Handling
- Normalized coordinates are multiplied by the width and height of the image and rounded to whole pixels
- Crops are clipped to the image; a box with no width or height is skipped with a warning

## Status and Errors

- **Success**: green dot `OK: <n> img in <t> ms (conv <c> ms | task <k> ms)`, where `<n>` is the number of crops; with Debug on, the preview format and size follow
- **No box kept**: the warning `No valid bounding boxes found with confidence >= <min>`, an empty array at Output to, and the message is sent
- **Failure**: `msg.error` is set, an empty array is written to Output to, a warning is logged, the status shows a red ring `Error: <first 30 characters>` and the message is still sent. Messages include `No image data found at image`, `No bounding box data found at default` and `Invalid bounding box data structure. Expected array at default`

## Performance Notes

- The crops are cut in parallel by the native OpenCV addon, on the libuv thread pool

## Real-World Examples

### Two-Stage Inspection
```
[Camera] → [Detection model] → [CropBB: Min Confidence 0.7] → [Split] → [Classifier] → [Join]
```
Classify every detected part with a second model; `tag` and `bbox` stay on each crop.

### Save the Defects
```
[Product Image] → [Defect detector] → [CropBB] → [Function: build { bitmap, outputDir, filename } items] → [Image-Out: Multiple images]
```
Save each defect crop with its tag in the file name.

## Common Issues & Troubleshooting

### No Crops Generated
- **Issue**: Empty array and the warning `No valid bounding boxes found ...`
- **Causes**: Min Confidence too high, boxes not in the `raw_boxes` form, or boxes that are rotated or have their corners in another order
- **Solution**: Lower Min Confidence, convert the detections to axis-aligned `raw_boxes` (top-left, top-right, bottom-right, bottom-left)

### Wrong Paths
- **Issue**: `No image data found at ...` or `No bounding box data found at ...`
- **Solution**: Set **Input from** and **Bounding Boxes** to the `msg` properties that hold the image and the detections (defaults `msg.image` and `msg.default`)

### Encoded Image Rejected
- **Issue**: `cropBB requires raw images with width/height metadata`
- **Solution**: Decode the image first, for example with [image-in](../io/image-in.md), or keep Raw output in the node before

### Coordinate Issues
- **Issue**: Crops appear in wrong locations or are empty
- **Cause**: The detection uses pixel coordinates
- **Solution**: Divide the coordinates by the image width and height
