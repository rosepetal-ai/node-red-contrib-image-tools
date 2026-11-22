# Node-RED Image Processing Toolkit

A comprehensive Node-RED package providing 20 specialized nodes for high-performance image processing using C++ OpenCV backend. Perfect for computer vision workflows, batch processing, and automated image manipulation.

![Demo](assets/flow_time_example.gif)

## Documentation

| Section | Pages |
|---------|-------|
| Overview | [Installation, requirements, nodes, image format and common settings](docs/index.md) |
| I/O nodes | [image-in](docs/nodes/io/image-in.md) · [folder-in](docs/nodes/io/folder-in.md) · [image-out](docs/nodes/io/image-out.md) |
| Transform nodes | [resize](docs/nodes/transform/resize.md) · [rotate](docs/nodes/transform/rotate.md) · [crop](docs/nodes/transform/crop.md) · [padding](docs/nodes/transform/padding.md) · [filter](docs/nodes/transform/filter.md) · [draw](docs/nodes/transform/draw.md) · [color-convert](docs/nodes/transform/color-convert.md) |
| Mix nodes | [concat](docs/nodes/mix/concat.md) · [mosaic](docs/nodes/mix/mosaic.md) · [advanced-mosaic](docs/nodes/mix/advanced-mosaic.md) |
| Blend nodes | [blend](docs/nodes/blend/blend.md) · [add-mask](docs/nodes/blend/add-mask.md) · [add-masks](docs/nodes/blend/add-masks.md) · [add-bbs](docs/nodes/blend/add-bbs.md) |
| Specialized nodes | [cropBB](docs/nodes/specialized/cropBB.md) · [image-align](docs/nodes/specialized/image-align.md) · [heat-diff](docs/nodes/specialized/heat-diff.md) |

The same pages are published at <https://docs.rosepetal.ai/flows/node-red-contrib-image-tools>.

## 🚀 Quick Start

### Installation (users)
**From Node-RED (Palette Manager)**
- Open Palette Manager → Install → search `@rosepetal/node-red-contrib-image-tools`.

**From npm (CLI, default)**
```bash
cd ~/.node-red
npm install @rosepetal/node-red-contrib-image-tools
```

### Development installation (build from source)
For contributors or platforms without a prebuilt binary:
```bash
./install.sh
# or manual steps:
cd rosepetal-image-engine && npm install && npm run build
cd .. && npm install
cd ~/.node-red && npm install /path/to/node-red-contrib-image-tools
```
> npm installs fetch prebuilt binaries for common platforms; build from source only if you are developing or a prebuilt is unavailable.

### Requirements
- Node.js 18.17+ (required by the `sharp` dependency) and Node-RED 1.0.0+
- Only to build from source: node-gyp build tools, OpenCV 4.x (auto-detected via pkg-config) and a C++ compiler with C++17 support

## 📋 Node Categories

### I/O Nodes - Data Flow Management
| Node | Purpose | Key Features |
|------|---------|-------------|
| **[image-in](docs/nodes/io/image-in.md)** | Load images from filesystem | JPEG/PNG/WebP/etc. support, Sharp backend |
| **[folder-in](docs/nodes/io/folder-in.md)** | Emit the images of a folder one by one | Configurable interval, loop or one-shot mode, start/stop button |
| **[image-out](docs/nodes/io/image-out.md)** | Save images with timestamp naming | Auto-naming, overwrite protection, format conversion |

### Transform Nodes - Image Processing
| Node | Purpose | Key Features |
|------|---------|-------------|
| **[resize](docs/nodes/transform/resize.md)** | Scale images with aspect ratio control | Proportional scaling, multiple modes |
| **[rotate](docs/nodes/transform/rotate.md)** | Rotate images with custom padding | Sub-pixel precision, background colors |
| **[crop](docs/nodes/transform/crop.md)** | Extract regions with pixel/normalized coords | Dynamic coordinates, bounds checking |
| **[padding](docs/nodes/transform/padding.md)** | Add configurable margins | Color customization, flexible sizing |
| **[filter](docs/nodes/transform/filter.md)** | Apply enhancement filters | Blur, sharpen, edge, emboss, Gaussian |
| **[draw](docs/nodes/transform/draw.md)** | Overlay points and lines on images | Normalized coordinates, typed inputs, C++ compositing |
| **[color-convert](docs/nodes/transform/color-convert.md)** | Convert images between color spaces | GRAY, RGB, RGBA, BGR and BGRA |

### Mix Nodes - Image Composition
| Node | Purpose | Key Features |
|------|---------|-------------|
| **[concat](docs/nodes/mix/concat.md)** | Combine images horizontally/vertically | Direction control, padding strategies |
| **[mosaic](docs/nodes/mix/mosaic.md)** | Position images on canvas | Coordinate positioning, manual placement |
| **[advanced-mosaic](docs/nodes/mix/advanced-mosaic.md)** | Complex layouts with custom positioning | Pixel-perfect control, layering |

### Blend Nodes - Advanced Compositing  
| Node | Purpose | Key Features |
|------|---------|-------------|
| **[blend](docs/nodes/blend/blend.md)** | Alpha blend two images with opacity | Background removal, color tolerance |
| **[add-mask](docs/nodes/blend/add-mask.md)** | Apply polygon-based masks to regions | Coordinate arrays, mask strength control |
| **[add-masks](docs/nodes/blend/add-masks.md)** | Apply multiple polygon masks colored by class | Class color mappings, global mask strength |
| **[add-bbs](docs/nodes/blend/add-bbs.md)** | Draw bounding boxes with labels on images | Tag and confidence labels, color per tag |

### Specialized Nodes - AI/ML Integration
| Node | Purpose | Key Features |
|------|---------|-------------|
| **[cropBB](docs/nodes/specialized/cropBB.md)** | Extract crops from AI bounding boxes | Object detection integration, confidence filtering |
| **[image-align](docs/nodes/specialized/image-align.md)** | Align an image to a reference with ECC, ORB/SIFT features, or both | Translation to homography models, speed presets |
| **[heat-diff](docs/nodes/specialized/heat-diff.md)** | Heat map of the difference between two images | Colormaps, blur and threshold, difference statistics |

## Architecture & Performance

### Two-Tier Design
- **JavaScript Layer**: Node-RED integration, validation, I/O handling
- **C++ Layer**: OpenCV-powered image processing for maximum performance

### Performance Benefits
- **Faster!** than pure JavaScript implementations
- **Parallel processing** for array operations
- **Memory efficient** with optimized algorithms
- **Async operations** with real-time performance timing
- **Never blocks the event loop**: decoding, processing and encoding (including BMP and colour-space swaps) all run on the libuv thread pool; the JavaScript layer only validates and routes messages

### Image Data Format
```javascript
{
  data: Buffer,        // Raw pixel data
  width: number,       // Image width in pixels  
  height: number,      // Image height in pixels
  channels: number,    // Channel count (1, 3, 4)
  colorSpace: string,  // "GRAY", "RGB", "RGBA", "BGR", "BGRA"
  dtype: string        // "uint8" (standard)
}
```

## 🛠️ Development

### Project Structure
```
docs/nodes/                    # Individual node documentation
assets/                        # Documentation assets

node-red-contrib-image-tools/  # Node-RED package sources
├── nodes/                     # Node-RED node implementations
│   ├── io/                    # I/O nodes
│   ├── transform/             # Transform nodes
│   ├── mix/                   # Composition nodes
│   ├── blend/                 # Blending nodes
│   └── specialized/           # AI/ML nodes
└── lib/                       # Shared utilities

rosepetal-image-engine/        # C++ processing engine
├── src/                       # OpenCV implementations
├── binding.gyp                # Node.js addon configuration
└── package.json
```

### Adding New Nodes
1. Implement C++ processing in `rosepetal-image-engine/src/`
2. Export function in `src/main.cpp`
3. Create Node-RED wrapper in appropriate `nodes/` category
4. Add comprehensive documentation following existing patterns

## 🚨 Troubleshooting

### Common Issues
- **Build Failures**: Ensure OpenCV 4.x installed and pkg-config available
- **Missing Nodes**: Check Node-RED installation path and package registration  
- **Performance Issues**: Verify C++ addon built with optimization flags
- **Memory Problems**: Process large images in smaller batches

### Performance Tips
- Use **raw format** for processing chains (fastest)
- Process **arrays when possible** for parallel execution
- Choose **appropriate output formats** for your destination
- Monitor **node status displays** for timing information
- Under heavy concurrency, raise the libuv thread pool (default 4 threads, shared with `sharp` and file I/O) before starting Node-RED, e.g. `UV_THREADPOOL_SIZE=8 node-red`

## 🤝 Contributing

- **Issues**: Report bugs and feature requests via GitHub
- **Pull Requests**: Contributions welcome for new nodes and optimizations
- **Documentation**: Help improve guides and examples

## 📄 License

Apache-2.0 (see the `LICENSE` file). This project is part of the Rosepetal development toolkit for Node-RED image processing applications.

---

**💡 Pro Tip**: Start with simple single-node workflows, then combine them into complex processing pipelines as you become familiar with the image format and node interactions.
