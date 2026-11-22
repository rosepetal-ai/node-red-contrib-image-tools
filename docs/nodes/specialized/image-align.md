---
title: "Image-Align Node - Align an Image to a Reference"
description: "Align a target image to a reference image in Node-RED with ECC, with ORB or SIFT features, or with features refined by ECC. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, image alignment, image registration, ecc, orb, sift, homography, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Image-Align Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Align a target image to a reference image in Node-RED with ECC, with ORB or SIFT features, or with features refined by ECC."
---

# Image-Align Node

## Purpose & Use Cases

The `image-align` node moves a target image so that it lines up with a reference image, and returns it in the frame and size of the reference. It finds the transform with ORB or SIFT feature matching, with OpenCV's Enhanced Correlation Coefficient (ECC) algorithm, or with features refined by ECC, for a translation, euclidean, affine or homography motion model. It can also return the transform and move polygons with the image.

**Real-World Applications:**
- **Quality Control**: Line up each product image with a golden image before comparing them (for example with [heat-diff](./heat-diff.md))
- **Fixed Inspection Regions**: Move regions of interest drawn on the reference so they follow the part in every image
- **Camera Shake**: Compensate small movements of a fixed camera between captures
- **Document Processing**: Bring scans of the same form into one frame

## Input/Output Specification

### Inputs
Only `msg` properties are read and written: the editor also offers `flow` and `global`, but the node reads the typed text as a `msg` path.

- **Reference Image** (`msg.payload.reference` by default): the image to align to
- **Target Image** (`msg.payload.target` by default): the image to move
- Each is a raw image object or an encoded JPEG/PNG/WebP/BMP `Buffer`. Both may also be arrays of the same length: the images are then paired by index and aligned in parallel, and every output below becomes an array in the same order.
- **Polygon Path** (`msg.payload.polygon` by default, with **Transform polygon coordinates**): one polygon `[[x, y], ...]` or an array of polygons `[[[x, y], ...], ...]`, normalized 0-1, giving positions on the target image

### Outputs
One output:
- **Output to** (`msg.payload` by default): the aligned target, with the width and height of the reference, as a raw image object or an encoded `Buffer` (JPEG/PNG/WebP). The area the target does not cover is black. When the alignment fails, the target is only scaled to the size of the reference.
- `msg.alignment`:

```javascript
{
  success: true,            // false when no transform was found (for arrays: true only if every pair succeeded)
  timing: { convertMs, taskMs, encodeMs },
  preset: "ultra-fast",
  motionModel: "affine",
  pipeline: "features+ecc",
  eccRefine: "always",
  detector: "orb",
  transformMatrix: {        // with "Return transformation matrix", when the alignment succeeded
    dx: 12.4, dy: -3.1,     // translation part, in pixels
    matrix2x3: [a, b, dx, c, d, dy],
    matrix3x3: [a, b, dx, c, d, dy, g, h, 1],   // last row 0, 0, 1 except for homography
    transform: [ ... ],     // same as matrix3x3
    motionModel: "affine", pipeline: "features+ecc", eccRefine: "always"
  },
  originalPolygon: [[0.1, 0.1], ...],      // with "Transform polygon coordinates"
  transformedPolygon: [[0.12, 0.09], ...], // or transformedPolygons for an array of polygons
  isPolygonArray: false
}
```

The matrix maps a pixel of the aligned (reference) frame to its position in the target image. The transformed polygons are the input polygons moved to where they lie on the aligned image, clamped to 0-1; they are only present when the alignment succeeded.

- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds

## Configuration Options

| Property | Default | Meaning |
|----------|---------|---------|
| Name | empty | Label of the node; also the key of its timings in `msg.performance.rpimage` |
| Reference Image | `msg.payload.reference` | Image to align to |
| Target Image | `msg.payload.target` | Image to move |
| Output to | `msg.payload` | Where the aligned image is written |
| Output Format | Raw (fastest) | Raw (fastest), JPEG, PNG or WebP, with Quality (`90`), PNG Optimize and the WebP options as in [resize](../transform/resize.md#output-format-options) |
| Debug / Debug Width | off / `200` | **Show processed image**: a preview of the (first) aligned image under the node in the editor |
| Transform Matrix | off | **Return transformation matrix** in `msg.alignment.transformMatrix` |
| Transform Polygon | off | **Transform polygon coordinates** read from Polygon Path |
| Polygon Path | `msg.payload.polygon` | The polygon(s) to move |
| Pipeline | Features → ECC refine | **ECC only (intensity)**, **Features only (ORB)** (uses the selected Detector) or **Features → ECC refine** |
| Detector | ORB (fast, default) | **ORB** or **SIFT (robust, slower)**, for the feature stage |
| ECC Refine | Always (legacy, accurate) | With Features → ECC refine: **Always**, **Auto (skip when seed is strong)** or **Never (use seed only)** |
| Motion Model | Affine (6 DoF, +scale+shear) | **Translation (2 DoF)**, **Euclidean (3 DoF, +rotation)**, **Affine (6 DoF, +scale+shear)** or **Homography (8 DoF, +perspective)** |
| Speed Preset | Ultra Fast | Ultra Fast, Fast, Balanced, Quality or Custom: see below |
| Scale / Max Iterations / Termination Eps | `0.2` / `10` / `0.1` | *Custom only.* Number, or read from `msg`, `flow` or `global` |

Nodes saved by versions that had no Motion Model or Pipeline setting run as **Translation** with **ECC only**.

### Pipelines
- **ECC only**: intensity-based ECC starting from no movement. Suits small shifts between images of the same scene.
- **Features only**: up to 800 ORB (or SIFT) keypoints per image, matched and fitted robustly (USAC MAGSAC for affine and homography, RANSAC for translation and euclidean). Handles larger movements; no ECC.
- **Features → ECC refine**: the feature result is the starting point of ECC. If ECC does not converge, the feature result is used. With **Auto**, ECC is skipped when the feature fit has at least 50 inliers; **Never** is the same as Features only.

Before ECC, both images are converted to grayscale and their brightness and contrast are normalized, so moderate lighting changes are tolerated. ECC counts as converged when its correlation is above 0.1.

### Speed Presets
Both stages work on copies of the images scaled by **Scale**; the transform is then scaled back to full resolution.

| Preset | Scale | Max Iterations | Termination Eps |
|--------|-------|----------------|-----------------|
| Ultra Fast | 0.2 | 10 | 0.1 |
| Fast | 0.3 | 20 | 0.01 |
| Balanced | 0.5 | 30 | 0.001 |
| Quality | 0.7 | 50 | 0.0001 |
| Custom | 0.1 to 1.0 | 1 to 200 | 0.000001 to 0.1 |

Max Iterations and Termination Eps apply to ECC. Custom values outside the ranges are clamped; empty or invalid values fall back to 0.2, 10 and 0.1.

### Image Sizes
For the ECC pipelines, and whenever polygons are transformed, a target of another size is first scaled to the size of the reference. With Features only, the target keeps its size for matching. The aligned image is resampled with nearest-neighbour interpolation.

## Status and Errors

- **Success**: green dot `OK: <n> img in <t> ms (conv <c> ms | task <k> ms)`, where `<n>` is the number of pairs, plus the preview format with Debug on. A failed alignment is not an error: check `msg.alignment.success`
- **Failure**: `msg.error` is set, the target image is written to **Output to**, a warning is logged, the status shows a red ring `Error: <first 30 characters>` and the message is still sent. Messages include `Reference and target must both be single images or both be arrays`, `Image arrays must be non-empty and of equal length (got <a> and <b>)`, `Reference image is invalid or missing` and `Target image is invalid or missing`
- **Invalid polygons**: a warning such as `Invalid coordinate at index 2: coordinates must be in range [0, 1]`, and the polygon transform is skipped

## Performance Notes

- Alignment and encoding run in the native OpenCV addon on the libuv thread pool; pairs of an array are aligned in parallel
- Repeated inputs are cached in the addon (scaled grayscale images, prepared ECC images and feature results), which helps the common case of one fixed reference and a new target per message
- SIFT is slower than ORB; Homography is the most flexible model and the easiest to fit wrongly on images with few features

## Real-World Examples

### Golden-Image Comparison
```
[Function: msg.payload = { reference: golden, target: msg.capture }] → [Image-Align: Features → ECC, Affine, Balanced] → [Heat-Diff]
```
Align each capture to a golden image before computing the difference.

### Inspection Region Follows the Part
```
[Function: set payload.reference, payload.target, payload.polygon] → [Image-Align: Transform polygon coordinates] → [Function: msg.roi = msg.alignment.transformedPolygon] → [Add-Mask: Polygon Path msg.roi]
```
Draw the region of interest once on the reference; every aligned image gets it in the right place.

### Batch of Stations
```
[Function: payload.reference = [ref1, ref2, ref3]; payload.target = [img1, img2, img3]] → [Image-Align] → [Split]
```
Three camera stations aligned in one message, each to its own reference.

## Common Issues & Troubleshooting

### Alignment Fails (`success: false`)
- **Cause**: too little texture or overlap, a movement too large for ECC only, or a motion model that cannot describe the movement
- **Solution**: use **Features → ECC refine**, try **SIFT** on blurry or low-texture images, raise the preset (Balanced or Quality), or pick a model with more freedom

### Wrong Alignment
- **Cause**: repeated patterns or a model with too much freedom (Homography on few features)
- **Solution**: use the simplest model that fits the setup (Translation or Euclidean for a fixed camera)

### Black Borders
- **Issue**: Edges of the output are black
- **Cause**: Those areas of the reference frame are not covered by the moved target
- **Solution**: Crop the output, or ignore the border in later comparisons

### Images from Context Are Ignored
- **Issue**: Reference or target set to `flow` or `global` is not found
- **Solution**: Copy the images into the message first; this node reads `msg` properties only
