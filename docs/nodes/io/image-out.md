---
title: "Image-Out Node - Save Images to Disk"
description: "Save raw images or encoded buffers to disk as JPEG, PNG, WebP or BMP in Node-RED, with timestamped file names and overwrite protection. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, image-out, save image, write image file, jpeg, png, webp, bmp, timestamp, rosepetal"
  - - meta
    - property: "og:title"
      content: "Image-Out Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Save raw images or encoded buffers to disk as JPEG, PNG, WebP or BMP in Node-RED, with timestamped file names and overwrite protection."
---

# Image-Out Node

## Purpose & Use Cases

The `image-out` node is the primary output point for saving processed images to the filesystem from Node-RED flows. It names files with a timestamp or a name you give, protects existing files from being overwritten, converts between formats, and can cap the number of files in the folder and stop saving when the disk is nearly full.

**Real-World Applications:**
- **Automated Image Processing**: Save processed images with organized timestamp naming
- **Computer Vision Results**: Export detected objects or processed frames
- **Batch Processing Workflows**: Save multiple images without filename conflicts
- **Quality Control Systems**: Archive inspection images with automatic naming
- **Web Application Backends**: Save user-processed images with format optimization

## Input/Output Specification

### Inputs
In **Single image** mode, the image read from **Input from** (`msg.payload` by default) is either:
- **Raw Image Object**: `data`, `width`, `height`, `channels`, `colorSpace` (as produced by the other nodes)
- **Encoded Buffer**: a JPEG, PNG or WebP file in a `Buffer`. A buffer already in the output format is written as it is; otherwise `sharp` converts it. BMP output needs a raw image object.

One image per message: an array is not accepted in this mode. To save several images from one message, use **Multiple images** mode.

In **Multiple images** mode, **Input** (`msg.payload` by default) is an array of objects (a single object is treated as an array of one). Each object carries its image in **Image field** (`bitmap` by default), its folder (`outputDir`; an item without one is skipped unless **Output dir** sets one folder for all items) and optionally its file name:

```javascript
msg.payload = [
  { bitmap: <image>, outputDir: "/data/ok",  filename: "part_0001.png" },
  { bitmap: <image>, outputDir: "/data/nok" }   // no filename: image_<timestamp>.<format>
];
```

### Outputs
None. The node writes files and reports the result in its status.

### Button Control
- **Button on the node**: switches the deployed node between active and inactive at once, like the button of the Debug node. The flow is marked as changed, so the state is saved with the next deploy.
- **Inactive**: grey dot `inactive`; incoming messages are dropped without saving.

## Configuration Options

| Property | Default | Meaning |
|----------|---------|---------|
| Name | empty | Label of the node in the editor |
| Mode | Single image | **Single image**: one image per message. **Multiple images**: an array of items, each with its own image, folder and file name |
| Folder Path | empty | *Single image.* Folder where the file is written: a string, or read from `msg`, `flow` or `global`. An empty string saves to the working directory of the Node-RED process |
| Filename Prefix | empty | *Single image.* Prefix of the generated name (`<prefix>_<timestamp>`). Empty: `image`. String, `msg`, `flow`, `global` or JSONata |
| Full Filename | empty | *Single image.* A complete file name that replaces the generated one. String, `msg`, `flow`, `global` or JSONata |
| Input from | `msg.payload` | *Single image.* Where the image is read: a `msg` property or a `flow` / `global` key |
| Input | `msg.payload` | *Multiple images.* Where the array of items is read |
| Image field | `bitmap` | *Multiple images.* Property of each item that holds the image |
| Output dir | item field `outputDir` | *Multiple images.* Folder of each item: a property of the item, or one folder for all items (string, `msg`, `flow` or `global`) |
| Filename | item field `filename` | *Multiple images.* File name of each item: a property of the item, or one name for all items (string, `msg`, `flow` or `global`) |
| Output Format | JPEG | JPEG, PNG, WebP or BMP |
| Quality | `90` | 1-100, for JPEG and WebP |
| Compression | `8` | PNG compression level, 0-9 (higher: smaller file, slower) |
| WebP Lossless | off | Encode WebP without loss |
| WebP Smart Subsample | off | Better chroma downsampling, lossy WebP only |
| WebP Effort | `0` | 0-6, higher gives a smaller file and is slower. Empty leaves the `sharp` default |
| Protection | on | Prevent overwriting existing files: a name that is taken gets `_2`, `_3` and so on |
| Max Images | empty | Keep at most this many files in the folder, deleting the oldest. Empty or `0` keeps all. Number, `msg`, `flow` or `global` |
| Disk Limit | `95` | Skip saving when the disk of the folder is at least this % full. `0` disables the check |
| Debug | off | *Single image.* **Show saved image preview** under the node in the editor |
| Debug Width | `200` | Width of the preview in pixels (a number; values read from `msg`, `flow` or `global` are not resolved by this node and give 200) |

### File Names
- **Generated name**: `<prefix>_<timestamp>.<ext>`, with the prefix `image` when **Filename Prefix** is empty. The timestamp is the UTC time with milliseconds, `YYYYMMDD_HHMMSS-mmmZ`: for example `image_20240109_143022-517Z.jpg`.
- **Full Filename** / **Filename**: used as given. If it ends in `.jpg`, `.jpeg`, `.png`, `.webp` or `.bmp`, that extension sets the format of this file instead of **Output Format**. Without an extension, the extension of **Output Format** is added. A name with `/` or `\` is rejected. Another extension is an error in Single image mode (`Unsupported extension in filename: .tif`); in Multiple images mode it is replaced by the extension of **Output Format**.
- In Multiple images mode an item without a file name gets `image_<timestamp>.<ext>`; **Filename Prefix** does not apply.
- Folders are created when they do not exist.

### Overwrite Protection
With **Protection** checked, a name that already exists in the folder becomes `<name>_2.<ext>`, then `_3` and so on, up to `_999`. The node knows the files of a folder from one listing made when it first writes there, plus its own writes; files that other programs create later are seen again only when **Max Images** re-lists the folder (at most every 5 seconds).

### Max Images
After each save the node counts **every file in the folder**, not only the images it wrote, and deletes the oldest ones (by modification time) until at most **Max Images** remain. Use a folder of its own for each Image-Out node that has a limit.

### Disk Limit
Before writing, the node reads the usage of the file system of the folder. At or above the limit it logs a warning, shows a yellow `disk <n>% full` status and skips the file (in Multiple images mode, the remaining items of the message; when no item of the message was saved, the node then fails with `All <n> items failed.`).

## Performance & Technical Details

### Format Handling
- **Raw image to JPEG, PNG or WebP**: encoded with `sharp`. BGR and BGRA images are converted to RGB/RGBA by the native addon first.
- **Raw image to BMP**: encoded by the native OpenCV addon (uncompressed; 8-bit grayscale, 24-bit or 32-bit).
- **Encoded buffer in the output format**: written without re-encoding. A WebP buffer is re-encoded when **WebP Lossless** or **WebP Effort** is set.
- **Encoded buffer in another format**: converted with `sharp`.

### Status Indicators
- **Grey dot** `inactive`: the node is switched off with its button
- **Blue dot** `saving...`: writing to disk
- **Green dot** `saved: <file> (<ms>ms)`, or `saved: <file> (<ms>ms, avoided overwrite)` when the name got a suffix. With Debug: `saved: <file> | <format> debug`
- **Green dot** `saved <n> files (<ms>ms)`, or `saved <n>/<total> files (<ms>ms)` when items were skipped (Multiple images)
- **Yellow ring** `disk <n>% full`, or `empty array` (Multiple images with an empty array)
- **Red ring** `Error`

## Error Handling

A failure is reported with `node.error` as `Error saving image: <message>` (a Catch node receives it), followed by a `Hint: ...` warning when the node can suggest one. Common messages:
- **"Folder path is not configured or resolved."**: the `msg`, `flow` or `global` value of Folder Path is missing or empty
- **"No image data found at specified input path"**: nothing at **Input from**
- **"Invalid image structure"**: the raw image object is not valid (see the warning logged before it)
- **"Cannot create directory: ..."**: the folder cannot be created; check the permissions
- **"Too many file variations exist"**: the name and all its variations up to `_999` already exist
- **"BMP output requires raw image data, not an encoded buffer"**
- **"Input must resolve to an array or object."** and **"All `<n>` items failed."** (Multiple images). A single item that fails is skipped with a warning such as `Item 3: no image at "bitmap", skipping.`

## Examples

### Basic Save with Timestamp
```javascript
// Input: Processed image
// Config: Folder="/output", Format="jpg"
// Result: /output/image_20240109_143022-517Z.jpg
```

### Custom Prefix for Source Identification
```javascript
// Config: Prefix="camera1", Folder="/surveillance"
// Result: /surveillance/camera1_20240109_143022-517Z.jpg
```

### Format Conversion
```javascript
// Input: PNG buffer
// Config: Format="webp", Quality=80
// Result: Converts PNG to WebP with 80% quality
```

### Dynamic Folder from Message
```javascript
// msg.saveFolder = "/output/batch1"
// Config: Folder from msg.saveFolder
// Result: Saves to dynamic folder location
```

### Fixed Name from the Message
```javascript
// msg.serial = "SN1234"
// Config: Full Filename (JSONata) = serial & ".png", Folder="/archive"
// Result: /archive/SN1234.png in PNG, whatever Output Format is set to
```

### Overwrite Protection in Action
```javascript
// Config: Full Filename="last.jpg"
// First save:  last.jpg
// Second save: last_2.jpg
// Third save:  last_3.jpg
```

## Tips & Best Practices

1. **Use Prefixes**: Identify image sources with meaningful prefixes
2. **Enable Protection**: Keep overwrite protection on to prevent data loss
3. **Choose Format Wisely**:
   - JPEG for photos (smaller files, no alpha)
   - PNG for graphics with transparency, or lossless output
   - WebP for small files, lossy or lossless
   - BMP for uncompressed, pixel-exact output
4. **Button Control**: Switch the node off during development to prevent unwanted saves
5. **Rolling Archive**: Set **Max Images** on a dedicated folder to keep only the most recent files

## Integration Examples

### Computer Vision Pipeline
```
[Camera] → [Object Detection] → [Add-BBs] → [Image-Out]
```
Save annotated detection results with timestamps.

### Batch Processing
```
[Function: build items] → [Image-Out: Multiple images]
```
Save every image of one message, each to its own folder and file name, from an array of `{ bitmap, outputDir, filename }` items.

### Quality Control
```
[Inspection Camera] → [Defect Detection] → [Switch: defect found] → [Image-Out]
```
Save only the images in which a defect was found.

## Related Nodes
- **[image-in](./image-in.md)**: Load images from filesystem
- **[folder-in](./folder-in.md)**: Read the images of a folder one by one
- **[add-bbs](../blend/add-bbs.md)**: Draw detections before saving
- **[filter](../transform/filter.md)**: Apply enhancements before saving
