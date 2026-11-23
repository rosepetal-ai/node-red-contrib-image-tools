---
title: "Folder-In Node - Emit the Images of a Folder"
description: "Emit the image files of a folder one by one at a fixed interval in Node-RED, in a loop or once, started and stopped with the node's button. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, folder-in, image folder, image sequence, replay images, interval, loop, rosepetal"
  - - meta
    - property: "og:title"
      content: "Folder-In Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Emit the image files of a folder one by one at a fixed interval in Node-RED, in a loop or once, started and stopped with the node's button."
---

# Folder-In Node

> **Type**: `rp-folder-in` · **Category**: I/O · **Since**: v1.1.3

## Overview

Folder-In reads the image files of a folder and sends them one per message at a fixed interval, as the same raw image object that [Image-In](./image-in.md) produces. It has no input: the button on the node starts and stops it. Unlike Image-In, which loads one file per incoming message, it replays a whole folder on its own.

**Use cases**
- Replay a set of saved camera images through an inspection flow, as if a camera were sending them
- Test a flow with a fixed set of images at a controlled rate
- Run a folder of images once through a model and save the results

## Inputs

None. Press the button on the left of the node to start; press it again to stop. The button is disabled while the node has changes that are not deployed.

On each start the node lists the folder again, keeps the files whose extension is `.jpg`, `.jpeg`, `.png`, `.webp`, `.bmp`, `.gif` or `.tiff` (in any letter case), and sorts them by name. Subfolders are not read. Files added while it runs are picked up at the next start.

## Outputs

One output. Every interval the node sends a **new** message with:

- **Output to** (`msg.payload` by default): the image, `{ data, width, height, channels, colorSpace, dtype }`, with `colorSpace` `GRAY`, `RGB` or `RGBA` from the channel count and `dtype` `uint8`. BMP files are decoded by the native OpenCV addon, the other formats by `sharp`. When **Output to** is a `flow` or `global` key, the image is stored there and the message carries only the properties below.
- `msg._folderIn`: `{ fileName, filePath, index, total }`, where `index` is the position of the file in the sorted list (from 0) and `total` the number of files. With Debug on, it also has `debug`, the format of the preview.
- `msg.performance.rpimage.<key>`: `{ conversion, task, total }` in milliseconds, where `conversion` is the decoding time. `<key>` is the name of the node, or `rp_folder_in` when it has no name.

## Configuration

| Field | Type | Default | Description |
|---|---|---|---|
| Name | string | empty | Label of the node in the editor |
| Folder Path | string | empty (required) | Folder to read. Only a fixed string works: with `msg`, `flow` or `global` the node warns `Dynamic folder paths require message input to resolve` and does not start |
| Interval (ms) | number | `1000` | Time between two images. The first image is sent at once. If loading an image takes longer than the interval, the next one follows as soon as it is done. An empty or negative value gives 1000. Only a number works: a `msg`, `flow` or `global` value is not resolved and gives 1000 |
| Mode | enum | Loop (repeat images) | **Loop (repeat images)**: start again with the first file after the last one. **One-shot (stop at end)**: stop after the last file |
| Output to | TypedInput (msg/flow/global) | `msg.payload` | Where the image is written |
| Debug | boolean | off | **Show loaded images**: a preview of each image under the node in the editor |
| Debug Width | number | `200` | Width of the preview in pixels |

## Status

| Status | Meaning |
|---|---|
| Grey ring `Stopped` | Not running (after deploy, or after the button stopped it) |
| Yellow dot `Scanning folder...` | Listing the folder |
| Red ring `No images found` | The folder has no file with a supported extension, cannot be read, or Folder Path is empty or dynamic |
| Green dot `Started (<n> images, <ms>ms)` | Running |
| Blue dot `Loading <file>...` | Reading and decoding a file |
| Green dot `<i>/<n>: <file>` | Sent image number `<i>` of `<n>`; with Debug on, the preview format follows |
| Grey ring `Completed all images` | One-shot mode reached the end |
| Red ring `Error: <message>` | A file could not be loaded |

## Examples

```
[folder-in: /data/samples, 500 ms, Loop] → [rp-resize] → [inference] → [debug]
```

Feeds the saved samples to a model at two images per second, over and over, until the button stops the node.

```
[folder-in: /data/batch, 100 ms, One-shot] → [rp-color-convert: GRAY] → [rp-image-out]
```

Converts every image of the folder once, ten per second, and saves the results.

## Troubleshooting

- **The node does not start and shows `No images found`** — Cause: Folder Path is empty, is set from `msg` / `flow` / `global`, does not exist (`Error scanning folder: ...` in the debug sidebar), or holds no file with a supported extension (`.tif` is not one). · Solution: set Folder Path as a string to an existing folder that the Node-RED process can read.
- **The interval does not change when set from `msg`, `flow` or `global`** — Cause: those values are not resolved. · Solution: type the interval as a number.
- **`Error: ...` repeats for the same file** — Cause: a file with a supported extension cannot be decoded (a corrupt file, or a grayscale image with alpha: `Unsupported number of channels: 2`). The node does not move past the failing file. · Solution: stop the node, remove or convert the file, and start it again.
- **Pressing the button shows `Node not deployed`** — Cause: the runtime has no running node with this id (the toggle request got a 404). A new or changed node cannot get there: its button stays disabled until it is deployed. · Solution: deploy the flow, with the node and its flow enabled.

## See Also

- [image-in](./image-in.md): load one file per message, with the path taken from the message if needed
- [image-out](./image-out.md): save images to disk
