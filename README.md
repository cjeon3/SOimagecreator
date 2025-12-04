# Sound Object Analyzer v2.2.55 - Technical Documentation

## Overview

The Sound Object Analyzer is a web-based tool developed for the UCI Hearing & Speech Lab to process and analyze participant drawings from Sound Object Phenomenon studies. Participants draw shapes representing their perception of sounds at different frequencies, using red (in-phase) and blue (out-of-phase) colors. This tool extracts those shapes from PNG images, calculates their geometric properties, and generates composite visualizations across all participants.

## Table of Contents

1. [Coordinate System](#coordinate-system)
2. [Pipeline Overview](#pipeline-overview)
3. [Step 1: Color Detection](#step-1-color-detection)
4. [Step 2: Connected Component Analysis](#step-2-connected-component-analysis)
5. [Step 3: Filtering](#step-3-filtering)
6. [Step 4: Centroid Marker Removal](#step-4-centroid-marker-removal)
7. [Step 5: Shape Filling (treatAsFilled)](#step-5-shape-filling-treatasfilled)
8. [Step 6: Component Merging](#step-6-component-merging)
9. [Step 7: Boundary Extraction & Ordering](#step-7-boundary-extraction--ordering)
10. [Step 8: Contour Resampling](#step-8-contour-resampling)
11. [Step 9: Centroid Calculation](#step-9-centroid-calculation)
12. [Step 10: Translation to Ground Truth](#step-10-translation-to-ground-truth)
13. [Shape Averaging](#shape-averaging)
14. [Google Sheets Integration](#google-sheets-integration)
15. [Output Formats](#output-formats)

---

## Coordinate System

The analyzer uses two coordinate systems that must be carefully converted between:

### Canvas Coordinates (Pixels)
- **Size**: 1000 × 1000 pixels
- **Origin**: Top-left corner (0, 0)
- **Y-axis**: Increases downward
- **Used for**: Image processing, pixel manipulation

### Unit Coordinates (Grid)
- **Range**: -10 to +10 on both axes (20 units total)
- **Origin**: Center of canvas (0, 0)
- **Y-axis**: Increases upward (mathematical convention)
- **Used for**: Data storage, averaging, output

### Conversion Formulas

```
Scale Factor = 1000 / 20 = 50 pixels per unit
Center = 500 pixels

Canvas → Unit:
  x_unit = (x_pixel - 500) / 50
  y_unit = (500 - y_pixel) / 50    ← Note Y inversion

Unit → Canvas:
  x_pixel = x_unit × 50 + 500
  y_pixel = 500 - y_unit × 50      ← Note Y inversion
```

**Example**: Grid position (3, 3) corresponds to pixel position (650, 350).

---

## Pipeline Overview

```
PNG Image
    │
    ▼
┌─────────────────────────────────┐
│  1. COLOR DETECTION             │  Identify red and blue pixels
│     (6-tier saturation system)  │  using HSV analysis
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  2. CONNECTED COMPONENTS        │  Group pixels into distinct
│     (8-connectivity flood fill) │  connected regions
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  3. FILTERING                   │  Remove tiny components (<10px)
│     (size + location)           │  and text region artifacts
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  4. MARKER REMOVAL              │  Remove centroid marker overlay
│     (template + location)       │  using geometric pattern matching
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  5. SHAPE FILLING               │  Fill interior holes to get
│     (flood fill from exterior)  │  solid shape representation
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  6. COMPONENT MERGING           │  Connect disconnected parts
│     (smart merge algorithm)     │  into unified contour
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  7. BOUNDARY EXTRACTION         │  Find outer edge pixels and
│     (contour tracing)           │  order them sequentially
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  8. RESAMPLING                  │  Create uniform point spacing
│     (arc-length interpolation)  │  for consistent analysis
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  9. CENTROID CALCULATION        │  Compute shape center as
│     (mean of resampled points)  │  average of all contour points
└─────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────┐
│  10. TRANSLATION                │  Align to ground truth centroid
│      (offset correction)        │  from Google Sheets data
└─────────────────────────────────┘
    │
    ▼
Extracted Shape Data
```

---

## Step 1: Color Detection

### Purpose
Identify which pixels belong to red (in-phase) or blue (out-of-phase) drawn shapes, distinguishing them from the background grid, anti-aliased edges, and overlapping colors.

### Challenge
The drawing tool produces anti-aliased edges where colors blend with the background or each other. Additionally, when blue is drawn over red, the overlap creates purple pixels that contain both colors.

### Method: 6-Tier Saturation System

Colors are detected using HSV (Hue, Saturation, Value) analysis with progressively relaxed thresholds:

#### For Red Detection:

| Tier | Saturation | Additional Criteria | Description |
|------|------------|---------------------|-------------|
| 1 | Any | RGB close to (239, 68, 68) | Exact core red match |
| 2 | > 0.40 | Red hue, R > G, R > B, R > 150 | High saturation red |
| 3 | 0.25-0.40 | Red hue, R > G+20, R > B+20, R > 180 | Medium saturation |
| 4 | 0.15-0.25 | Red hue, pink tone OR distance-based | Low saturation edges |
| 5 | 0.12-0.15 | Red hue, distance to red < 0.75× distance to blue | Very low saturation |
| 6 | > 0.25 | Purple hue (0.75-0.92), R > 100 | Purple overlap pixels |

#### Hue Ranges:
- **Red**: < 0.08 or > 0.92 (wraps around 0/1)
- **Blue**: 0.55 - 0.72
- **Purple**: 0.75 - 0.92 (classified as BOTH red and blue)

### Key Insight: Purple Pixels
When blue is drawn over red, purple pixels result. These are classified as **both** red and blue because:
- The red shape exists underneath
- The blue shape exists on top
- Both shapes should include these pixels for accurate area calculation

---

## Step 2: Connected Component Analysis

### Purpose
Group detected pixels into distinct connected regions, separating the main drawn shape from the centroid marker and any noise.

### Method: 8-Connectivity Flood Fill

Two pixels are considered connected if they share an edge OR a corner (8 neighbors total):

```
┌───┬───┬───┐
│ ↖ │ ↑ │ ↗ │
├───┼───┼───┤
│ ← │ X │ → │
├───┼───┼───┤
│ ↙ │ ↓ │ ↘ │
└───┴───┴───┘
```

### Algorithm
```
1. Create a Set of all pixel coordinates for O(1) lookup
2. For each unvisited pixel:
   a. Start new component
   b. Flood fill using stack (not recursion, to avoid stack overflow)
   c. Add all 8-connected neighbors to component
   d. Mark all visited pixels
3. Return array of components (each component is array of {x, y})
```

### Output
Array of components, sorted by size (largest first). Typically:
- Largest component = main drawn shape
- Smaller component(s) = centroid marker, noise

---

## Step 3: Filtering

### Purpose
Remove artifacts that aren't part of the actual drawn shape.

### Filters Applied

#### Size Filter
Components smaller than 10 pixels are discarded as noise (anti-aliasing artifacts, stray pixels).

#### Text Region Filter
The PNG images include statistical text in the top-left corner. Components with average position (avgX < 350, avgY < 120) are filtered out.

```
┌──────────────────────────┐
│ Statistics    │          │
│ Area: 12.3    │          │
│ Centroid: ... │  ← Filtered region
├───────────────┘          │
│                          │
│      [Drawing Area]      │
│                          │
└──────────────────────────┘
```

---

## Step 4: Centroid Marker Removal

### Purpose
The drawing tool overlays a centroid marker (cross + circle) on each shape. This marker must be removed to get the pure drawn shape.

### Marker Geometry
From the drawing tool source code:
- **Cross**: Arms extend ±12 pixels, line width 3 pixels
- **Circle**: Radius 14 pixels, line width 2 pixels
- **Total bounding box**: ~28-30 pixels
- **Total pixel count**: ~250-350 pixels typically

### Two-Phase Detection

#### Phase 1: Component-Level Detection
Check if an entire component looks like a marker based on geometry:

```javascript
Criteria:
- Pixel count: 100-600 pixels
- Bounding box: 20-45 pixels on each side
- Aspect ratio: < 1.8 (roughly square)
- Fill ratio: 0.12-0.65 (sparse, not solid)
```

If a component matches, the entire component is removed.

#### Phase 2: Pixel-Level Removal (using Sheets centroid)
When Google Sheets provides the known centroid location, pixels matching the marker pattern at that location are removed:

```javascript
function isMarkerPixel(px, py, centerX, centerY) {
    // Check if pixel is on:
    // - Horizontal arm: |dy| ≤ 4.5, |dx| ≤ 15
    // - Vertical arm: |dx| ≤ 4.5, |dy| ≤ 15
    // - Diagonal arm: ||dx| - |dy|| ≤ 4.5, dist ≤ 15
    // - Circle: 11 ≤ dist ≤ 17
    // - Center: dist ≤ 4.5
}
```

### Overlap Preservation
When marker pixels overlap with actual shape pixels, we check for "shape neighbors" - if a marker pixel has ≥5 neighboring pixels that aren't part of the marker pattern, it's preserved as part of the shape.

---

## Step 5: Shape Filling (treatAsFilled)

### Purpose
Convert an outline or partially-filled shape into a solid filled region. This helps:
1. Identify the true outer boundary
2. Reconnect fragments created by marker removal
3. Handle shapes with internal holes consistently

### Method: Exterior Flood Fill

```
1. Create binary image of component (1 = shape pixel, 0 = empty)
2. Flood fill from corner (0,0) marking all EXTERIOR pixels
3. Everything not marked as exterior is INTERIOR (filled shape)
```

### Visual Example
```
Before (outline):          After (filled):
  ████████                   ████████
  █      █                   ████████
  █      █        →          ████████
  █      █                   ████████
  ████████                   ████████
```

This ensures the boundary extraction gets the outer contour, not internal edges.

---

## Step 6: Component Merging

### Purpose
When marker removal or overlapping colors create multiple disconnected components from what should be a single shape, merge them back together intelligently.

### Smart Merge Algorithm

```
1. Trace each component's boundary separately
2. For each pair of components, find optimal connection points:
   a. Score = alignment × 100 - distance × 0.5
   b. Alignment = dot product of tangent vectors
   c. Prefer connections where curves are "flowing toward" each other
3. Merge components greedily, starting with best-scoring connections
4. Result: Single unified contour
```

### Tangent Calculation
At each point, calculate the local direction by averaging the last N segments:

```javascript
function getContourTangent(contour, idx, lookback) {
    // Average direction of last 'lookback' segments
    // Returns normalized vector (dx, dy)
}
```

### Connection Scoring
```
score = (tangent_alignment × 100) - (distance × 0.5)

Where:
- tangent_alignment: -1 to +1 (dot product of tangent vectors)
- distance: pixels between endpoints
```

High score = endpoints are close AND curves point toward each other.

---

## Step 7: Boundary Extraction & Ordering

### Purpose
Extract the outer edge pixels from a filled shape and order them sequentially around the perimeter.

### Boundary Extraction
A pixel is on the boundary if it has at least one 4-connected neighbor that is NOT part of the shape:

```
For each pixel in filled shape:
    if any of (left, right, up, down) is empty:
        pixel is boundary
```

### Contour Ordering (Moore-Neighbor Tracing)

```
1. Start at topmost-leftmost boundary pixel
2. Search for next boundary pixel in clockwise order:
   - Start from direction opposite to how we arrived
   - Check 8 neighbors in clockwise order
3. Mark visited, move to next pixel
4. Repeat until back at start or no unvisited neighbors
```

### Direction Encoding
```
Direction indices (clockwise from right):
  7  0  1
   ↖ ↑ ↗
6 ← X → 2
   ↙ ↓ ↘
  5  4  3
```

### Gap Jumping
If no adjacent boundary pixel is found (component has gaps), the algorithm finds the nearest unvisited boundary pixel and "jumps" to it, logging the connection.

---

## Step 8: Contour Resampling

### Purpose
Create a contour with uniformly-spaced points. This is essential for:
1. Accurate centroid calculation
2. Shape averaging across participants
3. Matching the drawing tool's methodology

### Adaptive Sample Count

```javascript
// From drawing tool: 1 sample per 0.1 units of perimeter
numSamples = max(100, floor(perimeterInUnits / 0.1))
```

For a shape with perimeter 15 units: `max(100, 150) = 150 samples`

### Arc-Length Resampling Algorithm

```
1. Calculate total perimeter length
2. Calculate target spacing = totalLength / numSamples
3. For each sample point:
   a. Calculate target distance along perimeter
   b. Find which segment contains that distance
   c. Interpolate position within segment
```

This ensures points are evenly distributed by arc-length, not by index.

---

## Step 9: Centroid Calculation

### Purpose
Calculate the center point of the shape, matching exactly how the drawing tool calculates it.

### Method: Mean of Resampled Points

```javascript
centroid.x = sum(all_points.x) / num_points
centroid.y = sum(all_points.y) / num_points
```

### Why This Works
Because points are uniformly distributed by arc-length, the simple mean gives the correct "center of mass" of the contour. This matches the drawing tool's implementation exactly.

### Note on Coordinate Systems
The centroid is calculated in pixel coordinates, then converted to unit coordinates for storage and comparison.

---

## Step 10: Translation to Ground Truth

### Purpose
Align the extracted shape to match the ground truth centroid from Google Sheets data. This ensures 100% centroid accuracy when validation data is available.

### Method

```
1. Calculate offset = (sheets_centroid - extracted_centroid)
2. Translate all pixels by offset
3. Translate all contour points by offset
4. New centroid = sheets_centroid exactly
```

### Why Translate?
Small extraction errors (anti-aliasing, marker removal artifacts) can shift the calculated centroid slightly. Translation corrects this without distorting the shape.

---

## Shape Averaging

### Purpose
Create a composite "average" shape from all participants' drawings at each frequency.

### Two Modes

#### Contour Averaging (Default)
```
1. Resample all shapes to same number of points
2. Align centroids (translate each shape so centroid is at origin)
3. Average corresponding points: avg[i] = mean(shape1[i], shape2[i], ...)
4. Translate result to average centroid position
```

#### Area-Normalized Averaging
```
1. Calculate median area of all input shapes
2. Perform contour averaging (no individual scaling)
3. Scale final result to match median area
```

This prevents very large or small shapes from dominating the average.

### Alignment Before Averaging
Shapes are aligned by centroid before averaging to prevent position differences from distorting the result:

```
Before alignment:          After alignment:
   ○                           ○
      ○                        ○
   ○                →          ○
         ○                     ○
   ○                           ○
```

---

## Google Sheets Integration

### Purpose
Load ground truth data (centroids, areas) from the drawing tool's data collection spreadsheet.

### Data Format (from Apps Script)
```javascript
{
  participant: "John D",
  trial: "1",
  frequency: 500,
  color: "red",
  area: 12.34,           // in grid squares (units²)
  centroidX: 1.5,        // in units (-10 to +10)
  centroidY: -0.3        // in units (-10 to +10)
}
```

### Matching Logic
Records are matched to PNG files by:
```
Filename: "John D_500Hz_80dB.png"
         → participant="John D", frequency=500
         
Match where:
  sheets.participant == "John D"
  AND sheets.frequency == 500
  AND sheets.trial == currentTrial
```

### Benefits
- Known centroid location improves marker removal accuracy
- Centroid translation ensures perfect centroid match
- Area data enables fallback circle generation if extraction fails

---

## Output Formats

### Per-Image Data
```javascript
{
  contour: [{x, y}, ...],      // Unit coordinates, resampled
  pixelContour: [{x, y}, ...], // Pixel coordinates
  rawPixels: [{x, y}, ...],    // Original cleaned pixels
  centroid: {x, y},            // Unit coordinates
  pixelCentroid: {x, y},       // Pixel coordinates
  pixelCount: 1234,            // Number of raw pixels
  sampleCount: 150,            // Resampling resolution
  usedSheetsCentroid: true,    // Whether Sheets data was used
  wasTranslated: true,         // Whether translation was applied
  translationOffset: {x, y}    // Offset applied (pixels)
}
```

### Composite Visualization
For each frequency, generates:
- Individual shape overlays (all participants)
- Average contour (red and blue)
- Centroid markers
- Reference circle (radius = 3 units)

### CSV Export
```
Frequency,Color,Area,CentroidX,CentroidY
31,red,12.34,1.50,-0.30
31,blue,8.76,-0.50,2.10
...
```

---

## Constants Reference

| Constant | Value | Description |
|----------|-------|-------------|
| CANVAS_SIZE | 1000 | Canvas dimensions in pixels |
| UNIT_RANGE | 10 | Grid range (±10 units) |
| SCALE_FACTOR | 50 | Pixels per unit |
| CENTER | 500 | Canvas center in pixels |
| MIN_CONTOUR_SAMPLES | 100 | Minimum resampling points |
| UNITS_PER_SAMPLE | 0.1 | Target spacing for resampling |
| RED_COLOR | (239, 68, 68) | Core red RGB |
| BLUE_COLOR | (59, 130, 246) | Core blue RGB |
| MARKER_CROSS_ARM | 12 | Marker cross arm length |
| MARKER_CIRCLE_RADIUS | 14 | Marker circle radius |

---

## Version History

**v2.2.55** - Current stable version
- Fixed pixelCount calculation to use cleaned outline pixels
- Cleaned up redundant code paths
- Stable pipeline before experimental refactoring

---

## Troubleshooting

### Common Issues

**No shapes detected**
- Check color detection thresholds
- Verify image format (must be PNG with expected colors)

**Centroid mismatch**
- Ensure Google Sheets data is connected
- Check participant name matching (case-sensitive)

**Fragmented shapes**
- Marker removal may have split the shape
- Smart merge should reconnect, check console logs

**Wild composite shapes**
- Check for failed extractions being included
- Verify all shapes have reasonable areas

---

## Contact

UCI Hearing & Speech Lab
Sound Object Phenomenon Research
