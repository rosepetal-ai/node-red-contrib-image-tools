---
title: "Image Processing Toolkit for Node-RED"
description: "Nodes for Node-RED that load, resize, crop, rotate, filter, compose, blend, align and save images with a native OpenCV addon. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, image processing, opencv, resize, crop, rotate, filter, mosaic, blend, mask, bounding box, image alignment, rosepetal"
  - - meta
    - property: "og:title"
      content: "Image Processing Toolkit - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Nodes for Node-RED that load, resize, crop, rotate, filter, compose, blend, align and save images with a native OpenCV addon."
---

# Image Processing Toolkit for Node-RED

`@rosepetal/node-red-contrib-image-tools` provides 20 nodes for Node-RED that load, transform, compose, annotate, compare and save images. The image operations run in a native C++ addon built on OpenCV.

![A Node-RED flow of image nodes, each showing its processing time in its status](../assets/flow_time_example.gif)

## Installation

```bash
cd ~/.node-red
npm install @rosepetal/node-red-contrib-image-tools
```

Or search `@rosepetal/node-red-contrib-image-tools` in the Node-RED Palette Manager.

npm fetches a prebuilt native addon, with OpenCV statically linked, for **linux-x64** and **linux-arm64** (glibc), **linuxmusl-x64** (Alpine) and **darwin-arm64**. On any other platform the addon has to be built from source: see [Building from source](#building-from-source).

## Requirements

- Node.js 18.17 or later (the `sharp` dependency supports `^18.17.0 || ^20.3.0 || >=21.0.0`)
- Node-RED >= 1.0.0
- One of the platforms above, or the tools to build the addon: OpenCV 4.x, a C++17 compiler and node-gyp

## Nodes

All node types are prefixed with `rp-` and appear under the **RP Image** category in the editor. Flows saved with the earlier type names, without the prefix (`resize`, `crop` and so on), keep working: the published package also registers those names, hidden from the palette.

### I/O

| Node | Purpose |
|------|---------|
| **[rp-image-in](./nodes/io/image-in.md)** | Load an image file from disk into the raw image object that the other nodes use |
| **[rp-image-out](./nodes/io/image-out.md)** | Save images to disk as JPEG, PNG, WebP or BMP, with timestamped file names and overwrite protection |

### Transform

| Node | Purpose |
|------|---------|
| **[rp-resize](./nodes/transform/resize.md)** | Scale images to a fixed size or by a factor, keeping the aspect ratio when one dimension is left empty |
| **[rp-rotate](./nodes/transform/rotate.md)** | Rotate images by an angle in degrees and fill the uncovered area with a pad color |
| **[rp-crop](./nodes/transform/crop.md)** | Extract a rectangular region, with normalized (0-1) or pixel coordinates |
| **[rp-padding](./nodes/transform/padding.md)** | Add a margin of a solid color on each side |
| **[rp-filter](./nodes/transform/filter.md)** | Apply blur, sharpening, edge detection, emboss, Gaussian blur, Otsu threshold, histogram equalization or CLAHE |
| **[rp-draw](./nodes/transform/draw.md)** | Overlay points and line segments, with normalized coordinates, color and opacity |

### Mix

| Node | Purpose |
|------|---------|
| **[rp-concat](./nodes/mix/concat.md)** | Join an array of images into one, to the right, left, down or up, padding or resizing the ones that differ in size |
| **[rp-mosaic](./nodes/mix/mosaic.md)** | Place the images of an array on a canvas at the positions set in the node, in pixels or normalized coordinates |
| **[rp-advanced-mosaic](./nodes/mix/advanced-mosaic.md)** | Compose the images of an array on a canvas with a position, size, rotation and stacking order per image, set in a visual editor |

### Blend

| Node | Purpose |
|------|---------|
| **[rp-blend](./nodes/blend/blend.md)** | Blend two images with an adjustable opacity, with optional alpha compositing and background color removal |
| **[rp-add-mask](./nodes/blend/add-mask.md)** | Blend a polygon mask over an image, from normalized polygon coordinates, with a fill color and a strength |
| **[rp-add-masks](./nodes/blend/add-masks.md)** | Blend several polygon masks over an image, with a color per class |

### Specialized

| Node | Purpose |
|------|---------|
| **[rp-cropBB](./nodes/specialized/cropBB.md)** | Extract one crop per bounding box of an inference result, filtered by a minimum confidence |
| **[rp-image-align](./nodes/specialized/image-align.md)** | Align a target image to a reference image with ECC, with ORB or SIFT features, or with features refined by ECC |

### Nodes without a reference page

Four nodes are described only by their help text, in the Help tab of the editor's sidebar:

| Node | Category | Purpose |
|------|----------|---------|
| **rp-folder-in** | I/O | Emit the image files of a folder one by one at a set interval, in a loop or once, started and stopped with the node's button |
| **rp-color-convert** | Transform | Convert images between the GRAY, RGB, RGBA, BGR and BGRA color spaces |
| **rp-add-bbs** | Blend | Draw bounding boxes on an image, labelled with their tag and confidence, with a color per tag |
| **rp-heat-diff** | Specialized | Compute the per-pixel absolute difference of two images as a color-mapped heat image, with optional difference statistics |

## Quick start

```
[inject] → [rp-image-in] → [rp-resize] → [rp-image-out]
```

1. In `rp-image-in`, set **File Path** to an image file on the Node-RED host. Any incoming message makes the node load the file into `msg.payload`.
2. In `rp-resize`, set **Width** and leave **Height** empty to keep the aspect ratio.
3. In `rp-image-out`, set **Folder Path**. With the default settings the image is saved there as `image_<timestamp>.jpg`.

## Image format

The nodes pass raw images between them as objects:

```javascript
{
  data: Buffer,        // Raw pixel data
  width: number,       // Image width in pixels
  height: number,      // Image height in pixels
  channels: number,    // Channel count (1, 3, 4)
  colorSpace: string,  // "GRAY", "RGB", "RGBA", "BGR", "BGRA"
  dtype: string        // "uint8"
}
```

`rp-image-in` and `rp-folder-in` produce this object. The addon also decodes an encoded image `Buffer` (JPEG, PNG or WebP) given in its place.

## Common settings

The transform, mix, blend and specialized nodes share these settings:

- **Input from / Output to**: where the image is read and where the result is written, as a `msg` property or a `flow` / `global` context key. The result goes to `msg.payload` by default.
- **Output Format**: `raw` (the image object, the default) or an encoded `Buffer` in `jpg`, `png` or `webp`. **Quality** (1-100, default 90) applies to JPEG and WebP.
- **Debug**: shows a preview of the result in the editor, with a configurable width (200 pixels by default).
- **Inference** (`rp-resize`, `rp-rotate`, `rp-crop`, `rp-padding`, `rp-concat`, `rp-mosaic` and `rp-advanced-mosaic`): when checked, the detections and segmentations read from `msg.inference` (by default) get the same geometric change as the image and are written back, so that their coordinates match the output image.

`rp-resize`, `rp-rotate`, `rp-crop`, `rp-padding`, `rp-filter` and `rp-color-convert` take one image or an array of images, and process the images of an array in parallel.

## Timing and errors

The transform, mix, blend and specialized nodes record their processing time in `msg.performance.rpimage.<key>` as `{ conversion, task, total }`, in milliseconds. `<key>` is the name of the node, or its type when it has no name, in lower case and with every run of other characters replaced by `_` (`rp_resize`).

When the processing fails, they do not stop the flow: they set `msg.error` (`message`, `operation`, `hint`, `details`, `timestamp`, `nodeId`, `nodeName`, `nodeType`), log a warning and still send the message.

## Performance

- Decoding, processing and encoding run on the libuv thread pool, not on the Node-RED event loop; the JavaScript layer only validates and routes messages.
- The pool has 4 threads by default and is shared with `sharp` and file I/O. Under heavy concurrency, raise it before starting Node-RED: `UV_THREADPOOL_SIZE=8 node-red`.
- Keep the `raw` output format between the nodes of a chain and encode only at the end.

## Building from source

For contributors, and for platforms without a prebuilt addon:

```bash
git clone https://github.com/rosepetal-ai/node-red-contrib-image-tools.git
cd node-red-contrib-image-tools
./install.sh
```

The script installs the build tools and OpenCV with the system's package manager (apt, dnf, yum, pacman or Homebrew) when `pkg-config` does not find OpenCV 4, builds the C++ addon, and offers to install the package into `~/.node-red`.

## Related packages

Several examples in these pages collect images into an array with Array-In and Array-Out nodes. They are `rp-array-in` and `rp-array-out`, which moved from this package to [`@rosepetal/node-red-contrib-utils`](https://docs.rosepetal.ai/flows/node-red-contrib-utils).

## License

Apache-2.0. The package is published on npm as [`@rosepetal/node-red-contrib-image-tools`](https://www.npmjs.com/package/@rosepetal/node-red-contrib-image-tools).
