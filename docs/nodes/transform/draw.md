---
title: "Draw Node - Overlay Points and Lines"
description: "Overlay points and line segments on an image in Node-RED, with normalized coordinates, color and opacity. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, draw, overlay, points, lines, keypoints, annotation, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Draw Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Overlay points and line segments on an image in Node-RED, with normalized coordinates, color and opacity."
---

# Draw Node

## Purpose & Use Cases

The `draw` node overlays filled circles (points) and straight line segments on one image, with normalized coordinates, a color and an opacity for each. Drawing happens in the native OpenCV addon, with anti-aliased edges.

**Typical scenarios**
- Visualizing detections (keypoints, landmarks, segmentation anchors)
- Highlighting measurement locations or points-of-interest
- Connecting bounding boxes with relational lines
- Creating quick debugging overlays inside automation flows

## Input / Output

### Input
- **Image**: one raw image object (`data`, `width`, `height`, `channels`, `colorSpace`, `dtype`) or an encoded JPEG/PNG/WebP/BMP `Buffer`. Arrays of images are not accepted.
- Source path defaults to `msg.payload` and can be remapped to any `msg`, `flow`, or `global` property.
- Point and line values set to `msg`, `flow` or `global` are read for each message.

### Output
- One output. The image with the overlays, in raw format (default) or encoded as `jpg`, `png`, or `webp`, written to **Output to** (`msg.payload` by default).
- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds.

## Configuration

### I/O
- **Input from / Output to**: Typed inputs (`msg`, `flow`, `global`), both `msg.payload` by default.
- **Output format**: Raw (fastest), JPEG, PNG or WebP, with Quality (`90`), PNG optimize and the WebP options as in [resize](./resize.md#output-format-options).

### Points
Each row of the **Points** table (columns Mode, X, Y, List, Radius, Color, Opacity) configures one filled circle:
- **X / Y**: Normalized coordinates (`0.0 – 1.0`), required, from `num`, `msg`, `flow` or `global`.
- **Radius**: Normalized size relative to the shortest image dimension (minimum 1 pixel; `0` draws nothing). New rows start at `0.02`; an empty value falls back to `0.01`.
- **Color**: `#RRGGBB` or `#RRGGBBAA`, typed or read from `msg`, `flow` or `global`. Default `#ff0000`.
- **Opacity**: Overlay strength (`0.0 – 1.0`), default `1`. Combined with any alpha channel supplied in the color.
- **Mode**: **Single values**, or **List** to provide `[[x,y], ...]` in one go (JSON, or from `msg`, `flow` or `global`; a JSON string is parsed). The row’s radius/color/opacity apply to every list entry.

Use **Add point** and **Remove** to manage the rows. Fields keep their typed-input settings, allowing mixed static and dynamic values.

### Lines
Each row of the **Lines** table (columns Mode, X1, Y1, X2, Y2, List, Thickness, Color, Opacity) defines one straight segment:
- **Start / End (X1, Y1, X2, Y2)**: Normalized coordinates, required.
- **Thickness**: Normalized width relative to the shortest dimension (minimum 1 pixel). New rows start at `0.01`; an empty value falls back to `0.005`.
- **Color**: `#RRGGBB` or `#RRGGBBAA`, typed or dynamic. Default `#00ff00`.
- **Opacity**: Blend factor (`0.0 – 1.0`), default `1`.
- **Mode**: **List** feeds multiple lines at once as `[[x0,y0,x1,y1], ...]`. Thickness/color/opacity are reused for each segment.

Use **Add line** and **Remove** to manage the rows.

### Debugging
- **Debug preview**: **Show processed image** under the node in the editor.
- **Debug width**: Width of the preview in pixels (default 200): a number, or read from `msg`, `flow` or `global`.

## Runtime Behaviour

1. The node validates the input image using shared `node-utils` helpers.
2. Points and lines are resolved via typed inputs—dynamic properties are evaluated per message.
3. The C++ engine converts normalized coordinates to pixels and composites overlays with anti-aliasing and alpha-aware blending.
4. The status shows a green dot `OK: 1 img in <t> ms (conv <c> ms | task <k> ms)`, and the timings go to `msg.performance.rpimage.<key>`.
5. Optional debug previews reuse the existing toolkit infrastructure (`debug-image` WebSocket).

If the image is invalid or the addon fails, the node sets `msg.error`, writes the input back to **Output to**, logs a warning, shows a red ring `Error: <first 30 characters>` and still sends the message.

## Tips & Edge Cases

- A point or line with a coordinate outside `[0, 1]`, a missing value or an invalid color is skipped with a warning such as `points[0].x must be a number between 0 and 1`; the others are still drawn.
- With no valid point or line the node warns `No valid points or lines configured. Outputting original image.` and sends the image unchanged in raw format. With JPEG, PNG or WebP output the result is an empty `Buffer`, because the addon returns before encoding (only WebP with the `sharp` options is still encoded).
- Opacity multiplies the color alpha, so `#FF000080` at opacity `0.5` results in 25% final strength.
- Grayscale inputs are supported; color overlays are converted using luminance weighting and added to the pixel values rather than blended, so a black overlay leaves a grayscale image unchanged.
- For chained overlays, prefer `raw` output to avoid repeated re-encoding.

## Example Flow

```
[image-in]
   └─▶ [draw]
          • point: x=msg.face.x, y=msg.face.y, radius=0.02, color=#00FFEE
          • line:  x1=0.1, y1=0.9 → x2=0.9, y2=0.9, thickness=0.004, color=#FFAA00
   └─▶ [image-out]
```

This creates a face marker, placed from the message, plus a fixed baseline overlay.
