---
title: "Advanced Mosaic Node - Canvas Layout with Resize and Rotation"
description: "Compose the images of an array on a canvas in Node-RED with a position, size, rotation and stacking order per image. Part of @rosepetal/node-red-contrib-image-tools."
head:
  - - meta
    - name: "keywords"
      content: "node-red, advanced mosaic, canvas, image layout, rotation, layering, mask propagation, opencv, rosepetal"
  - - meta
    - property: "og:title"
      content: "Advanced Mosaic Node - @rosepetal/node-red-contrib-image-tools"
  - - meta
    - property: "og:description"
      content: "Compose the images of an array on a canvas in Node-RED with a position, size, rotation and stacking order per image."
---

# Advanced Mosaic Node

## Purpose & Use Cases

The `advanced-mosaic` node provides sophisticated layout capabilities beyond basic grids, supporting custom positioning, variable cell sizes, and complex arrangement patterns. It enables pixel-perfect control over image placement for professional layouts and artistic compositions.

**Real-World Applications:**
- **Magazine Layouts**: Create complex editorial layouts with varying image sizes
- **Web Design Mockups**: Arrange elements in custom positions for design prototypes
- **Artistic Compositions**: Create asymmetric, visually interesting image arrangements
- **Dashboard Design**: Position charts and data visualizations with precise control
- **Poster Creation**: Design promotional materials with sophisticated image arrangements

## Input/Output Specification

### Inputs
- **Image Array**: Array of image objects with optional position metadata
- **Layout Configuration**: Custom positioning data via message properties or configuration
- **Canvas Parameters**: Target canvas dimensions and layout constraints

### Outputs
- **Complex Layout**: Single composite image with precisely positioned elements
- **Custom Canvas**: Canvas sized according to layout requirements or fixed dimensions
- **Format Options**: Raw image object or encoded file formats
- **Mask Propagation (optional)**: When enabled, a second output array containing masks transformed with the exact same resize/rotate/position steps. Each mask index aligns with the source image index and is ready for downstream drawing/overlay nodes.

## Configuration Options

### Input/Output Paths
- **Input From**: `msg.payload` (default), `flow.*`, `global.*`
- **Output To**: `msg.payload` (default), `flow.*`, `global.*`

### Canvas Configuration

#### Canvas Size
- **Fixed Dimensions**: Specify exact width and height
- **Auto Size**: Calculate canvas size based on image positions and sizes
- **Aspect Ratio**: Maintain specific aspect ratio while auto-sizing

#### Background Settings
- **Background Color**: Color for empty canvas areas
- **Background Image**: Use image as background behind positioned elements
- **Transparency**: Support for transparent backgrounds (PNG output)

### Positioning System

#### Position Modes
- **Absolute**: X,Y coordinates in pixels
- **Relative**: Positions as percentages of canvas size (0-1)
- **Grid-Based**: Enhanced grid system with custom cell sizes
- **Flow Layout**: Automatic positioning with spacing rules

#### Position Sources
- **Configuration**: Set positions in node editor
- **Message Data**: Position data provided per image in input array
- **Dynamic Calculation**: Calculate positions using function nodes

### Advanced Layout Features

#### Variable Cell Sizes
- **Custom Width/Height**: Each image can have different cell dimensions
- **Aspect Ratio Preservation**: Maintain image proportions within custom cells
- **Scaling Options**: Fit, fill, or stretch images within designated areas

#### Layering and Z-Order
- **Layer Control**: Specify which images appear above others
- **Overlap Handling**: Manage overlapping elements gracefully
- **Transparency Support**: Proper alpha blending for overlapping areas

### Output Format Options
- **Raw**: Standard image object (fastest for processing chains)
- **JPEG**: Compressed (transparency becomes background color)
- **PNG**: Full transparency and quality preservation
- **WebP**: Modern format with transparency and compression
- **Mask Propagation**: Enable the *Propagate masks* checkbox to feed a mask array (one per input image index). The node applies the same resize/rotate/position to each mask and emits a transformed mask array for downstream overlay/drawing without manual conversions.

## Mask Propagation Input Format

When *Propagate masks* is enabled, supply `msg.masks` (or the configured path) as an **array aligned to the input images array**. Index `i` is the mask for image `i`. Each entry can be one of:

- **Polygon mask**: `{ tag?: "class", polygons: [ [ [x,y], [x,y], ... ] , ... ] }` where points are normalized 0–1.
- **Raw mask image**: `{ mask: { data, width, height, channels, colorSpace, dtype } , tag?: "class" }` (RGBA/LA/GRAY accepted; alpha preferred).
- **2D matrix mask**: `{ mask: [[0,1,1,...], ...], tag?: "class" }` or simply a 2D/array-of-2D numeric matrix.

Rules:
- Array length should match the images array; missing entries are ignored.
- `tag`/`class`/`class_name`/`label` are passed through if present.
- Masks are binarized, resized with `INTER_NEAREST`, rotated, and positioned exactly like their source image. The output `masks` array mirrors the input order and is ready for downstream drawing nodes.

## Performance Notes

### C++ Backend Processing
- **Optimized Rendering**: Efficient composite rendering with layer management
- **Memory Management**: Smart memory allocation for complex layouts
- **Overlap Detection**: Optimized algorithms for handling overlapping elements
- **High-Quality Scaling**: Professional-grade scaling for variable-sized elements

### Layout Engine
- **Position Validation**: Automatic bounds checking and constraint handling
- **Layer Ordering**: Efficient z-order processing for overlapping elements
- **Canvas Optimization**: Minimal canvas size calculation for auto-sizing modes

## Real-World Examples

### Magazine Article Layout
```javascript
// In function node: set custom positions for magazine layout
msg.images.forEach((img, i) => {
  if (i === 0) { // Hero image
    img.position = { x: 0, y: 0, width: 800, height: 400 };
  } else if (i === 1) { // Side image
    img.position = { x: 820, y: 0, width: 300, height: 200 };
  } else { // Thumbnail strip
    img.position = { x: 100 * i, y: 420, width: 80, height: 80 };
  }
});
```
Create professional magazine-style layouts with hero images and supporting visuals.

### Dashboard Layout
```
[Chart Images] → [Function: Set Positions] → [Advanced Mosaic: Custom layout] → [Dashboard]
```
Position charts, graphs, and KPI displays in precise dashboard arrangements.

### Product Showcase
```javascript
// Custom product display with hero and detail shots
msg.productImages[0].position = { x: 0, y: 0, width: 600, height: 600 }; // Main product
msg.productImages[1].position = { x: 620, y: 0, width: 280, height: 200 }; // Detail 1
msg.productImages[2].position = { x: 620, y: 220, width: 280, height: 200 }; // Detail 2
msg.productImages[3].position = { x: 620, y: 440, width: 280, height: 160 }; // Detail 3
```

### Artistic Collage Creation
```
[Art Images] → [Function: Random Positions] → [Advanced Mosaic: Overlap enabled] → [Artistic Collage]
```
Create visually interesting overlapping compositions with controlled randomness.

### Web Design Mockup
```
[UI Elements] → [Function: Responsive Positions] → [Advanced Mosaic: Fixed canvas] → [Mockup]
```
Arrange UI elements for web design prototypes and mockups.

## Common Issues & Troubleshooting

### Position Conflicts
- **Issue**: Images overlapping unintentionally or awkward positioning
- **Solution**: Validate position calculations and implement collision detection
- **Prevention**: Use position validation functions before processing

### Canvas Size Problems
- **Issue**: Canvas too small for positioned elements or excessive whitespace
- **Solution**: Use auto-sizing or carefully calculate required canvas dimensions
- **Planning**: Consider element positions and sizes when planning canvas

### Performance with Complex Layouts
- **Issue**: Slow processing with many overlapping elements
- **Optimization**: Minimize unnecessary overlaps, use appropriate image sizes
- **Monitoring**: Track processing times and optimize layout complexity

### Memory Issues
- **Issue**: Large canvases with high-resolution images consume excessive memory
- **Solution**: Use appropriately sized images for final output requirements
- **Strategy**: Consider final display size when sizing source images

## Integration Patterns

### Professional Layout Workflow
```
Content → Analyze Layout → Calculate Positions → Advanced Mosaic → Output
```
Automated professional layout generation based on content analysis.

### Interactive Design System
```
User Input → Position Calculator → Advanced Mosaic → Preview → Iterate
```
Interactive layout system for design applications.

### Template-Based Generation
```
Layout Template → Apply to Content → Advanced Mosaic → Branded Output
```
Use predefined templates for consistent branded layouts.

### Responsive Layout System
```
Content → Detect Screen Size → Calculate Responsive Positions → Advanced Mosaic → Adaptive Output
```
Create layouts that adapt to different output dimensions.

## Advanced Usage

### Template-Based Positioning
```javascript
// Define layout templates
const templates = {
  hero: {
    main: { x: 0, y: 0, width: 0.7, height: 1.0 },
    sidebar: [  
      { x: 0.72, y: 0, width: 0.28, height: 0.3 },
      { x: 0.72, y: 0.32, width: 0.28, height: 0.3 },
      { x: 0.72, y: 0.64, width: 0.28, height: 0.36 }
    ]
  },
  grid: {
    // Define grid-based template
  }
};

msg.layout = templates[msg.layoutType];
```

### Responsive Position Calculation
```javascript
// Calculate positions based on canvas size
function calculateResponsivePositions(images, canvasWidth, canvasHeight) {
  return images.map((img, i) => {
    // Implement responsive positioning logic
    const position = {
      x: Math.floor((i % 3) * canvasWidth / 3),
      y: Math.floor(Math.floor(i / 3) * canvasHeight / 3),
      width: canvasWidth / 3 - 10,
      height: canvasHeight / 3 - 10
    };
    return { ...img, position };
  });
}
```

### Overlap Management
```javascript
// Detect and resolve position conflicts
function resolveOverlaps(elements) {
  for (let i = 0; i < elements.length; i++) {
    for (let j = i + 1; j < elements.length; j++) {
      if (elementsOverlap(elements[i], elements[j])) {
        // Implement overlap resolution strategy
        adjustPosition(elements[j], elements[i]);
      }
    }
  }
  return elements;
}
```

### Dynamic Layout Optimization
```javascript
// Optimize layout based on content analysis
function optimizeLayout(images, canvasSize) {
  // Analyze image importance, size requirements, etc.
  const important = images.filter(img => img.priority === 'high');
  const secondary = images.filter(img => img.priority === 'medium');
  const background = images.filter(img => img.priority === 'low');
  
  // Position important images first, then fill with secondary
  return arrangeByPriority(important, secondary, background, canvasSize);
}
```

## Best Practices

### Layout Design
- Plan layouts on paper or design tools before implementing
- Use consistent spacing and alignment principles
- Consider visual hierarchy and reading patterns
- Test layouts with different content variations

### Position Management
- Use relative positioning for responsive layouts
- Implement position validation to prevent errors
- Consider overlap handling strategies from the start
- Document position calculation logic for maintenance

### Performance Optimization
- Use appropriate image sizes for final output requirements
- Minimize unnecessary overlapping elements
- Consider memory implications of large canvas sizes
- Profile performance with realistic content loads

### Quality Considerations
- Use high-quality source images for scaling operations
- Choose appropriate output formats for intended use
- Consider compression settings for web vs. print
- Test final output in intended display environment

### Development Workflow
- Start with simple positioning and add complexity gradually
- Use debug output to verify position calculations
- Implement position validation and error handling
- Create reusable positioning templates for consistency
