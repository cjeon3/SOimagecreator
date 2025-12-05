# Sound Object Analyzer v2.2.56 — Comprehensive Documentation

## Overview

The Sound Object Analyzer is a browser-based research tool developed for the UCI Hearing & Speech Lab's Sound Object Phenomenon studies. It processes participant drawings that represent how people perceive sound objects at various frequencies, generating publication-ready composite visualizations and statistical analyses.

Participants use a companion drawing application to create **red shapes** (representing in-phase perception) and **blue shapes** (representing out-of-phase perception) around a reference circle at 11 standard frequencies.

---

## Table of Contents

1. [Supported Frequencies](#supported-frequencies)
2. [Coordinate System](#coordinate-system)
3. [Sampling System](#sampling-system)
4. [Processing Pipeline](#processing-pipeline)
5. [Color Detection](#color-detection)
6. [Contour Extraction](#contour-extraction)
7. [Centroid Calculation](#centroid-calculation)
8. [Shape Resampling](#shape-resampling)
9. [Extreme Overlap Handling](#extreme-overlap-handling)
10. [Shape Averaging](#shape-averaging)
11. [Area Calculations](#area-calculations)
12. [Statistics (All Mean-Based)](#statistics-all-mean-based)
13. [Google Sheets Integration](#google-sheets-integration)
14. [Output Visualizations](#output-visualizations)
15. [Key Constants](#key-constants)

---

## Supported Frequencies

The analyzer processes drawings at 11 standard frequencies (in Hz):

```
31, 62.5, 125, 250, 500, 1000, 2000, 4000, 8000, 12000, 16000
```

**Important**: There is no 5 Hz frequency. Files are parsed from filenames in the format:
```
ParticipantName_FrequencyHz_dB.png
Example: "Diego G_31Hz_100dB.png"
```

---

## Coordinate System

The analyzer uses two coordinate systems that must be precisely matched to the drawing tool:

### Canvas Coordinates (Pixel Space)
- **Canvas size**: 1000 × 1000 pixels
- **Origin**: Top-left corner (0, 0)
- **Y-axis**: Increases downward

### Unit/Grid Coordinates (Mathematical Space)
- **Range**: -10 to +10 units in both X and Y
- **Origin**: Center (0, 0) at pixel (500, 500)
- **Y-axis**: Increases upward (standard mathematical convention)
- **Scale**: 50 pixels per unit

### Conversion Formulas

**Canvas to Unit:**
```javascript
function canvasToUnit(x, y) {
    return {
        x: (x - 500) / 50,
        y: (500 - y) / 50  // Y is inverted
    };
}
```

**Unit to Canvas:**
```javascript
function unitToCanvas(x, y) {
    return {
        x: x * 50 + 500,
        y: 500 - y * 50  // Y is inverted
    };
}
```

### Example
Grid position (3, 3) corresponds to pixel position (650, 350):
- x_pixel = 3 × 50 + 500 = 650
- y_pixel = 500 - 3 × 50 = 350

---

## Sampling System

### Fixed N=1000 Samples

All contours in v2.2.56 are resampled to exactly **1000 evenly-spaced points**. This replaces the previous adaptive sampling system that varied based on perimeter length.

### Starting Point: Topmost

Each contour starts from the **topmost point**:
- In **pixel/canvas coordinates**: Smallest Y value (top of screen)
- In **unit/grid coordinates**: Highest Y value (mathematically "up")

### Direction: Clockwise

Contours proceed **clockwise** from the starting point:
- Visual direction: Top → Right → Bottom → Left → Top
- Enforced via signed area calculation

### Signed Area Formula for Direction Detection

```javascript
// Calculate signed area using shoelace formula
let signedArea = 0;
for (let i = 0; i < contour.length; i++) {
    const p1 = contour[i];
    const p2 = contour[(i + 1) % contour.length];
    signedArea += (p1.x * p2.y - p2.x * p1.y);
}
signedArea *= 0.5;

// In unit coordinates (Y up):
// Positive = counter-clockwise
// Negative = clockwise
if (signedArea > 0) {
    // Reverse to make clockwise (keep first point, reverse rest)
    contour = [contour[0], ...contour.slice(1).reverse()];
}
```

---

## Processing Pipeline

### Step 1: File Upload
- Accept PNG files or ZIP archives
- Parse filenames to extract participant name, frequency, and trial number
- Group files by frequency

### Step 2: Google Sheets Integration (Optional)
- Connect to Google Apps Script web app
- Fetch ground-truth centroid and area data
- Build trial mapping from ZIP folder names

### Step 3: Image Processing
For each PNG image:

1. **Color Detection**: Identify red and blue pixels using color thresholds
2. **Component Detection**: Find connected components using flood fill
3. **Marker Removal**: Remove centroid marker pixels (cross + circle)
4. **Boundary Extraction**: Trace outer contour of each shape
5. **Gap Filling**: Close any gaps in the contour
6. **Resampling**: Resample to N=1000 points, topmost start, clockwise
7. **Centroid Calculation**: Calculate centroid from resampled contour
8. **Translation**: Translate shape to match Sheets centroid (if available)

### Step 4: Shape Averaging
- Group shapes by frequency and color
- Align shapes to their centroids
- Average corresponding points across all shapes
- Apply area normalization (optional)
- Final resampling to N=1000

### Step 5: Visualization & Statistics
- Generate composite images for each frequency
- Calculate area and centroid statistics
- Display extreme overlap replacements
- Validate against Google Sheets data

---

## Color Detection

### Red Pixel Detection

Red pixels are detected using multiple criteria:

```javascript
function isRedPixel(r, g, b, a) {
    if (a < 128) return false;  // Skip transparent pixels
    
    // Primary: Strong red channel dominance
    if (r > 150 && r > g + 30 && r > b + 30) return true;
    
    // Secondary: Medium-strong red with clear dominance
    if (r > 100 && r > g - 20) {
        // Check for various red shades including darker reds
        if (r > b && g < 150) return true;
    }
    
    // Tertiary: HSV-based detection for edge pixels
    const max = Math.max(r, g, b);
    const min = Math.min(r, g, b);
    const saturation = max > 0 ? (max - min) / max : 0;
    
    if (saturation > 0.40 && r > g && r > b) {
        const distToRed = Math.sqrt((r-239)**2 + (g-68)**2 + (b-68)**2);
        if (distToRed < 120) return true;
    }
    
    return false;
}
```

### Blue Pixel Detection

Similar multi-criteria approach for blue:

```javascript
function isBluePixel(r, g, b, a) {
    if (a < 128) return false;
    
    // Primary: Strong blue channel dominance
    if (b > 150 && b > r + 20 && b > g - 20) return true;
    
    // Secondary: Medium-strong blue
    if (b > 100 && b > g - 20) {
        if (b > r && r < 180) return true;
    }
    
    // Tertiary: HSV-based detection
    const max = Math.max(r, g, b);
    const min = Math.min(r, g, b);
    const saturation = max > 0 ? (max - min) / max : 0;
    
    if (saturation > 0.40 && b > r && b > g) {
        const distToBlue = Math.sqrt((r-59)**2 + (g-130)**2 + (b-246)**2);
        if (distToBlue < 100) return true;
    }
    
    return false;
}
```

### Reference Colors
- **Red**: RGB(239, 68, 68) — Tailwind red-500 (#ef4444)
- **Blue**: RGB(59, 130, 246) — Tailwind blue-500 (#3b82f6)
- **Color Tolerance**: 80 units in RGB space

---

## Contour Extraction

### Connected Component Analysis

Pixels of each color are grouped into connected components using 8-connectivity flood fill:

```javascript
function findConnectedComponents(pixels, width, height) {
    // Create pixel set for O(1) lookup
    const pixelSet = new Set(pixels.map(p => `${p.x},${p.y}`));
    const visited = new Set();
    const components = [];
    
    for (const pixel of pixels) {
        if (visited.has(`${pixel.x},${pixel.y}`)) continue;
        
        // Flood fill to find connected component
        const component = floodFill(pixel.x, pixel.y);
        if (component.length > 0) {
            components.push(component);
        }
    }
    
    return components;
}
```

### Boundary Extraction

The outer boundary is extracted by identifying edge pixels:

```javascript
function extractBoundaryFromFilledPixels(filledPixels) {
    const pixelSet = new Set(filledPixels.map(p => `${p.x},${p.y}`));
    const boundary = [];
    
    // 8-direction neighbors
    const dx = [1, 1, 0, -1, -1, -1, 0, 1];
    const dy = [0, 1, 1, 1, 0, -1, -1, -1];
    
    for (const p of filledPixels) {
        // Check if any neighbor is NOT in the shape
        let isEdge = false;
        for (let d = 0; d < 8; d++) {
            const nx = p.x + dx[d];
            const ny = p.y + dy[d];
            if (!pixelSet.has(`${nx},${ny}`)) {
                isEdge = true;
                break;
            }
        }
        
        if (isEdge) {
            boundary.push({ x: p.x, y: p.y });
        }
    }
    
    return boundary;
}
```

### Boundary Ordering

Boundary pixels are ordered sequentially by following adjacent pixels:

1. Start with topmost-leftmost point
2. Find nearest unvisited neighbor (8-connectivity)
3. Prefer neighbors that continue in the same direction
4. Continue until all boundary pixels are visited

---

## Centroid Calculation

The centroid is calculated as the **arithmetic mean** of all N=1000 resampled contour points:

```javascript
function calculateContourCentroidLikeDrawingTool(resampledContour) {
    if (!resampledContour || resampledContour.length === 0) {
        return { x: 0, y: 0 };
    }
    
    let sumX = 0, sumY = 0;
    for (const p of resampledContour) {
        sumX += p.x;
        sumY += p.y;
    }
    
    return {
        x: sumX / resampledContour.length,
        y: sumY / resampledContour.length
    };
}
```

This matches the drawing tool's approach where the centroid is the mean of uniformly-sampled contour points.

---

## Shape Resampling

### Main Resampling Function

```javascript
function resampleContour(contour, targetSamples) {
    if (contour.length < 2) return contour;
    
    // Step 1: Find topmost point (smallest Y in canvas coordinates)
    let topmostIndex = 0;
    let topmostY = contour[0].y;
    for (let i = 1; i < contour.length; i++) {
        if (contour[i].y < topmostY) {
            topmostY = contour[i].y;
            topmostIndex = i;
        }
    }
    
    // Step 2: Rotate contour to start from topmost point
    let rotatedContour = [
        ...contour.slice(topmostIndex),
        ...contour.slice(0, topmostIndex)
    ];
    
    // Step 3: Ensure clockwise direction
    let signedArea = 0;
    for (let i = 0; i < rotatedContour.length; i++) {
        const p1 = rotatedContour[i];
        const p2 = rotatedContour[(i + 1) % rotatedContour.length];
        signedArea += (p1.x * p2.y - p2.x * p1.y);
    }
    signedArea *= 0.5;
    
    if (signedArea > 0) {  // Counter-clockwise, reverse
        rotatedContour = [rotatedContour[0], ...rotatedContour.slice(1).reverse()];
    }
    
    // Step 4: Calculate total path length
    let totalLength = 0;
    const segmentLengths = [];
    for (let i = 0; i < rotatedContour.length; i++) {
        const p1 = rotatedContour[i];
        const p2 = rotatedContour[(i + 1) % rotatedContour.length];
        const len = Math.sqrt((p2.x - p1.x)**2 + (p2.y - p1.y)**2);
        segmentLengths.push(len);
        totalLength += len;
    }
    
    // Step 5: Resample to exactly targetSamples points
    const resampled = [];
    let currentDistance = 0;
    let segmentIndex = 0;
    
    for (let sampleNum = 1; sampleNum <= targetSamples; sampleNum++) {
        const targetDistance = ((sampleNum - 1) / targetSamples) * totalLength;
        
        // Find segment containing target distance
        while (segmentIndex < segmentLengths.length - 1 && 
               currentDistance + segmentLengths[segmentIndex] < targetDistance) {
            currentDistance += segmentLengths[segmentIndex];
            segmentIndex++;
        }
        
        // Interpolate within segment
        const remainingDist = targetDistance - currentDistance;
        let progress = segmentLengths[segmentIndex] > 0 ? 
            remainingDist / segmentLengths[segmentIndex] : 0;
        progress = Math.max(0, Math.min(1, progress));
        
        const p1 = rotatedContour[segmentIndex];
        const p2 = rotatedContour[(segmentIndex + 1) % rotatedContour.length];
        
        resampled.push({
            x: p1.x + (p2.x - p1.x) * progress,
            y: p1.y + (p2.y - p1.y) * progress
        });
    }
    
    return resampled;
}
```

### Geometric Properties Preserved

The resampling preserves all geometric properties:

1. **Shape**: Points are interpolated along the original path
2. **Area**: Shoelace formula on 1000 points gives accurate area
3. **Centroid**: Mean of 1000 evenly-spaced points
4. **Perimeter**: Sum of distances between consecutive points

---

## Extreme Overlap Handling

When red shapes are severely occluded by blue shapes (drawn on top), the analyzer detects and handles this:

### Detection Criteria
```javascript
const EXTREME_OVERLAP_THRESHOLD = 0.30;

// Red has < 30% of blue's pixel count = extreme overlap
if (redPixelCount < bluePixelCount * EXTREME_OVERLAP_THRESHOLD) {
    // Handle extreme overlap
}
```

### Replacement Strategy

When extreme overlap is detected:

1. Use blue's already-traced contour
2. Translate it to red's centroid position (from Sheets data)
3. Mark the shape as `copiedFromBlue: true`
4. Track the replacement for reporting

```javascript
const dx = redCentroid.x - blueCentroid.x;
const dy = redCentroid.y - blueCentroid.y;

const translatedContour = blueShape.contour.map(p => ({
    x: p.x + dx,
    y: p.y + dy
}));
```

### Replacement Tracking

All replacements are tracked and displayed:
- Total count by color (red/blue)
- Breakdown by frequency
- Participant and trial details

---

## Shape Averaging

### Contour-Only Averaging (Default)

```javascript
function calculateAverageShape(shapes) {
    // 1. Resample all shapes to N=1000
    const resampledShapes = shapes.map(shape => {
        const pixelContour = shape.contour.map(p => unitToCanvas(p.x, p.y));
        const resampledPixels = resampleContour(pixelContour, 1000);
        return resampledPixels.map(p => canvasToUnit(p.x, p.y));
    });
    
    // 2. Align to centroids (translate each to origin)
    const alignedShapes = alignContoursToCentroid(resampledShapes);
    
    // 3. Average corresponding points
    const avgContour = [];
    for (let i = 0; i < 1000; i++) {
        let sumX = 0, sumY = 0;
        for (const shape of alignedShapes) {
            sumX += shape.contour[i].x;
            sumY += shape.contour[i].y;
        }
        avgContour.push({
            x: sumX / shapes.length + avgOffsetX,
            y: sumY / shapes.length + avgOffsetY
        });
    }
    
    // 4. Final resample (ensures topmost start, clockwise)
    return resampleContourLikeDrawingTool(avgContour);
}
```

### Area-Normalized Averaging (Optional)

When "Contour + Area Normalization" mode is selected:

```javascript
function calculateAverageShapeWithAreaNormalization(shapes) {
    // 1. Calculate mean area of all input shapes
    const areas = shapes.map(s => calculateContourArea(s.contour));
    const meanArea = areas.reduce((a, b) => a + b, 0) / areas.length;
    
    // 2. Average contours (same as above)
    const avgContour = calculateAverageShape(shapes);
    
    // 3. Calculate area of averaged contour
    const avgArea = calculateContourArea(avgContour);
    
    // 4. Scale to match mean area
    const scaleFactor = Math.sqrt(meanArea / avgArea);
    
    // 5. Scale around centroid
    const scaledContour = avgContour.map(p => ({
        x: cx + (p.x - cx) * scaleFactor,
        y: cy + (p.y - cy) * scaleFactor
    }));
    
    return resampleContourLikeDrawingTool(scaledContour);
}
```

---

## Area Calculations

### Shoelace Formula

Area is calculated using the Shoelace formula on the contour points:

```javascript
function calculateContourArea(contour) {
    if (!contour || contour.length < 3) return 0;
    
    let area = 0;
    for (let i = 0; i < contour.length; i++) {
        const j = (i + 1) % contour.length;
        area += contour[i].x * contour[j].y;
        area -= contour[j].x * contour[i].y;
    }
    
    return Math.abs(area / 2);
}
```

### Unit System for Area
- Area is in **units²** (square grid units)
- 1 unit = 50 pixels on the 1000×1000 canvas
- Reference circle: radius = 3 units, area = π × 3² ≈ 28.27 units²

---

## Statistics (All Mean-Based)

All statistics in v2.2.56 use **mean** (arithmetic average), not median.

### Area Statistics

```javascript
function calculateAreaStatistics(shapes) {
    const areas = shapes.map(s => calculateContourArea(s.contour));
    
    // Mean
    const sum = areas.reduce((acc, a) => acc + a, 0);
    const mean = sum / areas.length;
    
    // Standard Deviation (using mean)
    const squaredDiffs = areas.map(a => (a - mean) ** 2);
    const variance = squaredDiffs.reduce((sum, d) => sum + d, 0) / areas.length;
    const stdDev = Math.sqrt(variance);
    
    // Min and Max
    const min = Math.min(...areas);
    const max = Math.max(...areas);
    
    return { mean, stdDev, min, max, areas };
}
```

### Centroid Statistics

```javascript
function calculateCentroidStatistics(shapes) {
    const xCoords = shapes.map(s => s.centroid.x);
    const yCoords = shapes.map(s => s.centroid.y);
    
    // Mean X and Y
    const meanX = xCoords.reduce((a, b) => a + b, 0) / xCoords.length;
    const meanY = yCoords.reduce((a, b) => a + b, 0) / yCoords.length;
    
    // Standard Deviations
    const stdDevX = Math.sqrt(
        xCoords.map(x => (x - meanX)**2).reduce((a, b) => a + b, 0) / xCoords.length
    );
    const stdDevY = Math.sqrt(
        yCoords.map(y => (y - meanY)**2).reduce((a, b) => a + b, 0) / yCoords.length
    );
    
    return { meanX, meanY, stdDevX, stdDevY };
}
```

### Mean Radius

```javascript
function calculateMeanRadius(shapes) {
    const shapeRadii = shapes.map(shape => {
        const cx = shape.centroid.x;
        const cy = shape.centroid.y;
        
        // Mean distance from centroid to all contour points
        const distances = shape.contour.map(p => 
            Math.sqrt((p.x - cx)**2 + (p.y - cy)**2)
        );
        return distances.reduce((a, b) => a + b, 0) / distances.length;
    });
    
    // Mean of all shape radii
    return shapeRadii.reduce((a, b) => a + b, 0) / shapeRadii.length;
}
```

---

## Google Sheets Integration

### Data Structure

The analyzer expects Google Sheets data with these columns:
- **Column A**: Participant name
- **Column B**: Trial number
- **Column C**: Frequency (Hz)
- **Column D**: Color ("red" or "blue")
- **Column E**: Centroid X (units)
- **Column F**: Centroid Y (units)
- **Column G**: Area (units²)

### Lookup Function

```javascript
function lookupSheetsData(participant, trial, frequency, color) {
    if (!sheetsData) return null;
    
    const record = sheetsData.find(row =>
        row.participant?.toLowerCase() === participant.toLowerCase() &&
        row.trial?.toString() === trial.toString() &&
        parseFloat(row.frequency) === frequency &&
        row.color?.toLowerCase() === color.toLowerCase()
    );
    
    if (record) {
        return {
            centroid: { x: parseFloat(record.centroidX), y: parseFloat(record.centroidY) },
            area: parseFloat(record.area)
        };
    }
    
    return null;
}
```

### Centroid Translation

When Sheets data is available, shapes are translated to match the ground-truth centroid:

```javascript
function translateShapeToSheetsCentroid(rawPixels, contour, extractedCentroid, sheetsCentroid) {
    const targetPixel = unitToCanvas(sheetsCentroid.x, sheetsCentroid.y);
    
    const offsetX = targetPixel.x - extractedCentroid.x;
    const offsetY = targetPixel.y - extractedCentroid.y;
    
    const translatedContour = contour.map(p => ({
        x: p.x + offsetX,
        y: p.y + offsetY
    }));
    
    return { contour: translatedContour, centroid: sheetsCentroid };
}
```

---

## Output Visualizations

### Composite Image Components

Each frequency composite includes:

1. **Grid**: -10 to +10 unit grid with axis labels
2. **Reference Circle**: Gray dashed circle, radius = 3 units
3. **Individual Shapes**: Semi-transparent gray outlines (α=0.3)
4. **Red Average**: Bold red contour (5px line)
5. **Blue Average**: Bold blue contour (5px line)
6. **Statistics**: N and mean radius for each color
7. **Frequency Label**: Displayed at top center

### Drawing Order

```
1. White background
2. Grid lines and labels
3. Reference circle (dashed gray)
4. Individual red shapes (gray, α=0.3)
5. Individual blue shapes (gray, α=0.3)
6. Red average contour (solid red, 5px)
7. Blue average contour (solid blue, 5px)
8. Text labels (frequency, statistics)
```

---

## Key Constants

```javascript
// Canvas and coordinate system
const CANVAS_SIZE = 1000;              // 1000×1000 pixels
const UNIT_RANGE = 10;                 // -10 to +10 units
const SCALE_FACTOR = 50;               // 50 pixels per unit
const CENTER = 500;                    // Center pixel coordinate

// Reference circle
const REFERENCE_CIRCLE_RADIUS_UNITS = 3;   // 3 grid units
const REFERENCE_CIRCLE_RADIUS_PX = 150;    // 150 pixels

// Sampling
const MIN_CONTOUR_SAMPLES = 1000;      // Fixed N=1000 samples

// Color detection
const RED_COLOR = { r: 239, g: 68, b: 68 };    // #ef4444
const BLUE_COLOR = { r: 59, g: 130, b: 246 };  // #3b82f6
const COLOR_TOLERANCE = 80;

// Extreme overlap
const EXTREME_OVERLAP_THRESHOLD = 0.30;  // 30%

// Marker detection
const MARKER_CROSS_ARM = 12;           // Cross arm length in pixels
const MARKER_CIRCLE_RADIUS = 14;       // Circle radius in pixels
const MARKER_BBOX_MIN = 20;            // Minimum bounding box
const MARKER_BBOX_MAX = 45;            // Maximum bounding box
const MARKER_PIXELS_MIN = 100;         // Minimum pixel count
const MARKER_PIXELS_MAX = 600;         // Maximum pixel count
```

---

## Filename Conventions

### Input Files
```
{Participant}_{Frequency}Hz_{dB}dB.png
Example: "Diego G_31Hz_100dB.png"
```

### ZIP Archives
```
{Participant}_drawings.zip
Contents: Multiple PNG files for that participant
```

### Output Files
```
composite_{Frequency}Hz.png
Example: "composite_1000Hz.png"
```

---

## Version History

### v2.2.56 (Current)
- Fixed N=1000 samples for all shapes (replaces adaptive sampling)
- Topmost starting point for all contours
- Clockwise direction enforced
- All statistics changed from median to mean
- Extreme overlap replacement tracking and display
- Geometric properties preserved through resampling

### Previous Versions
- v2.2.55: Code review and cleanup
- v2.2.24: Small area threshold adjustment
- v2.2.x: Various bug fixes and improvements

---

## Technical Notes

### Browser Compatibility
- Requires modern browser with ES6+ support
- Uses HTML5 Canvas API
- Requires File API for uploads
- Uses async/await for file processing

### Dependencies (CDN-loaded)
- Tailwind CSS (styling)
- JSZip (ZIP file handling)
- FileSaver.js (download functionality)

### Performance Considerations
- Large ZIP files may take time to process
- N=1000 samples provides good balance of accuracy and performance
- Web Workers could be added for parallel processing

---

## Contact

UCI Hearing & Speech Lab  
Sound Object Analyzer — Cartesian Pipeline v2.2.56

For questions about methodology or calculations, refer to this documentation or the inline code comments.
