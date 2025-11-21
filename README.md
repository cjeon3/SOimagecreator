# Sound Object Average Shape Analyzer - Complete Documentation

## Table of Contents
1. [Overview](#overview)
2. [How It Works](#how-it-works)
3. [Image Analysis Pipeline](#image-analysis-pipeline)
4. [Mathematical Calculations](#mathematical-calculations)
5. [Averaging Algorithm](#averaging-algorithm)
6. [Visualization Generation](#visualization-generation)
7. [Usage Guide](#usage-guide)
8. [Technical Specifications](#technical-specifications)

---

## Overview

The Sound Object Average Shape Analyzer is a web-based tool that processes drawings from the Sound Object drawing tool and generates averaged visualizations showing typical shapes across multiple participants.

### Purpose
- **Input**: Individual participant drawings representing how they perceive sounds at different frequencies
- **Output**: Averaged shapes showing the collective perception across all participants
- **Goal**: Identify patterns in how people visually represent different sound frequencies

### Key Features
- Processes hundreds of images automatically
- Handles nested ZIP files
- Maintains exact mathematical accuracy
- Generates publication-ready visualizations
- Provides statistical analysis (mean, standard deviation)

---

## How It Works

### The Big Picture

```
Individual Drawings → Extract Contours → Calculate Properties → Average Across Participants → Generate Visualization
```

### Step-by-Step Process

#### 1. **File Upload & Extraction**
- User uploads PNG images or ZIP files
- System recursively extracts all PNG files from nested ZIPs
- Files are organized by participant, frequency, and phase (red/blue)

#### 2. **Image Analysis**
Each image goes through:
- Pixel scanning to find drawn shapes
- Contour extraction (outline of the shape)
- Centroid calculation (center point)
- Area measurement
- Radial distance measurement

#### 3. **Averaging**
For each frequency and phase:
- Align all shapes by their centroids
- Average radial distances at each angle
- Calculate mean and standard deviation

#### 4. **Visualization**
- Draw averaged shape with standard deviation envelope
- Show individual shapes (faded)
- Add grid, labels, and statistics
- Export as high-quality PNG

---

## Image Analysis Pipeline

### Phase 1: Pixel Detection

**What happens:**
The analyzer scans each 1000×1000 pixel image to find which pixels are part of the drawing.

**Bounding Box Optimization:**
```
1. Quick scan (every 10 pixels) to find drawing region
2. Focus only on that region + padding
3. Skip empty white space (saves 40-60% of work)
```

**Pixel Classification:**
- ✅ **Drawn pixels**: Not white, not gray (grid lines), sufficient opacity
- ❌ **Background**: White pixels (R>240, G>240, B>240)
- ❌ **Grid**: Gray pixels (R≈G≈B, in range 140-200)

**Color Phase Detection:**
- Sum all red values, sum all blue values
- If red sum > blue sum → **Red phase** (in-phase)
- If blue sum > red sum → **Blue phase** (out-of-phase)

### Phase 2: Coordinate Transformation

**Canvas to Unit Conversion:**

The drawing tool uses a coordinate system where:
- Canvas: 1000×1000 pixels
- Unit range: -10 to +10 units in both X and Y
- Scale factor: 50 pixels per unit

```javascript
// Convert canvas pixel to unit coordinate
unitX = (pixelX - 500) / 50
unitY = (500 - pixelY) / 50

// Example:
// Pixel (500, 500) → Unit (0, 0) [center]
// Pixel (750, 250) → Unit (5, 5) [upper right]
// Pixel (250, 750) → Unit (-5, -5) [lower left]
```

**Why this matters:**
- All calculations happen in unit space (not pixels)
- This matches the original drawing tool exactly
- Ensures consistency across different display sizes

### Phase 3: Centroid Calculation

**What is a centroid?**
The centroid is the "center of mass" of the shape - the average position of all points on the outline.

**Why it's important:**
- Used to align shapes from different participants
- Allows comparison of shape structure independent of position
- Essential for accurate averaging

**How it's calculated:**

1. **Uniform Resampling** (most important step)
   ```
   Purpose: Sample the outline evenly, regardless of how it was drawn
   
   - Calculate total path length of the outline
   - Determine number of samples: max(100, floor(length / 0.1))
   - Place samples at equal intervals along the path
   - Longer paths get more samples (e.g., 500 samples for 50-unit path)
   ```

2. **Path Interpolation**
   ```
   For each sample point:
   1. Find which line segment it falls on
   2. Calculate exact position via linear interpolation
   3. Store resampled point
   ```

3. **Calculate Average**
   ```javascript
   centroidX = (sum of all resampled X coordinates) / (number of samples)
   centroidY = (sum of all resampled Y coordinates) / (number of samples)
   ```

**Example:**
```
Original drawing: 5000 pixels along outline
After resampling: 250 evenly-spaced points
Centroid: Average position of those 250 points
Result: (-0.5, 1.2) in unit coordinates
```

**Why uniform resampling?**
Without it, centroid would be biased toward:
- Areas where user drew slowly (more pixels)
- Corners (dense pixel clusters)

With resampling: Every part of the outline has equal weight.

### Phase 4: Area Calculation

**What is measured?**
The area enclosed by the drawn shape, accounting for brush stroke width.

**Why it's complex:**
- Shapes might not be closed (open curves)
- Need to account for brush thickness (default: 5 pixels = 0.1 units)
- Must handle both filled regions and outline strokes

**Grid-Based Rasterization Method:**

1. **Determine Brush Radius**
   ```javascript
   brushRadius = brushSize / 2 / scaleFactor
   // For default brush size 5:
   brushRadius = 5 / 2 / 50 = 0.05 units
   ```

2. **Set Resolution**
   ```javascript
   if (brushRadius < 0.05) resolution = 0.05
   else if (brushRadius < 0.1) resolution = 0.075
   else resolution = 0.1
   ```

3. **Find Bounding Box**
   ```
   minX = minimum X coordinate - brushRadius
   maxX = maximum X coordinate + brushRadius
   minY = minimum Y coordinate - brushRadius
   maxY = maximum Y coordinate + brushRadius
   ```

4. **Scan Grid Cells**
   ```
   For each grid point (x, y) from minX to maxX, minY to maxY:
     Step by resolution (e.g., 0.1 units)
     
     If shape is closed:
       Check if point is inside polygon (ray casting)
       OR check if within brushRadius of any edge
     
     If shape is open:
       Check if within brushRadius of any edge
     
     If painted: count this cell
   ```

5. **Calculate Area**
   ```javascript
   cellArea = resolution × resolution
   totalArea = paintedCells × cellArea
   ```

6. **Adjustment for Closed Shapes**
   ```javascript
   // Avoid double-counting the outline
   perimeter = total length of outline
   outlineArea = perimeter × (brushRadius × 2) × 0.50
   adjustedArea = totalArea - outlineArea
   ```

**Example Calculation:**
```
Shape with:
- Brush size: 5 pixels (0.1 unit diameter)
- Resolution: 0.075 units
- Grid cells painted: 850
- Cell area: 0.075 × 0.075 = 0.005625 square units
- Total area: 850 × 0.005625 = 4.78 square units
```

**Distance to Segment Algorithm:**

Used to check if a grid point is within brush radius of a line segment:

```javascript
function distanceToSegment(px, py, x1, y1, x2, y2):
    // Vector from point 1 to point 2
    dx = x2 - x1
    dy = y2 - y1
    lengthSquared = dx² + dy²
    
    if lengthSquared == 0:
        // Segment is actually a point
        return distance from (px,py) to (x1,y1)
    
    // Project point onto line segment
    t = ((px - x1) × dx + (py - y1) × dy) / lengthSquared
    t = clamp(t, 0, 1)  // Keep on segment
    
    // Find closest point on segment
    closestX = x1 + t × dx
    closestY = y1 + t × dy
    
    // Calculate distance
    return sqrt((px - closestX)² + (py - closestY)²)
```

### Phase 5: Radial Binning

**Purpose:**
Convert the shape into 180 radial measurements (every 2 degrees around the centroid).

**Why 180 bins?**
- Provides smooth contour (2° resolution)
- Human eye cannot distinguish from 1° resolution
- 2× faster than 360 bins
- Still captures all shape details

**Process:**

1. **For each pixel in the shape:**
   ```javascript
   // Relative to centroid
   dx = pixel.x - centroid.x
   dy = pixel.y - centroid.y
   
   // Calculate angle and distance
   angle = atan2(dy, dx)  // Range: -π to +π
   radius = sqrt(dx² + dy²)
   
   // Convert to bin index (0 to 179)
   binIndex = floor(((angle + π) / (2π)) × 180)
   
   // Keep maximum radius for this angle
   if (radius > bins[binIndex].maxRadius):
       bins[binIndex].maxRadius = radius
       bins[binIndex].point = pixel
   ```

2. **Result:**
   - Array of 180 points representing the outermost boundary
   - Each point stored with its angle and radius
   - Forms a smooth contour around the shape

**Example:**
```
Bin 0 (angle 0°): radius 3.2 units → Point (3.2, 0) relative to centroid
Bin 45 (angle 90°): radius 4.1 units → Point (0, 4.1) relative to centroid
Bin 90 (angle 180°): radius 2.8 units → Point (-2.8, 0) relative to centroid
... and so on for all 180 directions
```

---

## Mathematical Calculations

### 1. Coordinate System

**Canvas Coordinates:**
- Origin: Top-left corner (0, 0)
- X increases: Left → Right
- Y increases: Top → Bottom (inverted)
- Range: 0 to 1000 pixels

**Unit Coordinates:**
- Origin: Center of canvas (0, 0)
- X increases: Left → Right
- Y increases: Bottom → Top (standard Cartesian)
- Range: -10 to +10 units
- Scale: 50 pixels = 1 unit

**Transformation Formulas:**

Canvas → Unit:
```
unitX = (canvasX - 500) / 50
unitY = (500 - canvasY) / 50
```

Unit → Canvas:
```
canvasX = unitX × 50 + 500
canvasY = 500 - unitY × 50
```

### 2. Path Length Calculation

Used to determine centroid sampling resolution.

```javascript
totalLength = 0
for i from 0 to (numPoints - 2):
    p1 = points[i]
    p2 = points[i + 1]
    
    dx = p2.x - p1.x
    dy = p2.y - p1.y
    segmentLength = sqrt(dx² + dy²)
    
    totalLength += segmentLength

return totalLength
```

### 3. Shape Closure Detection

Determines if a shape is closed (like a circle) or open (like an arc).

**Criteria:**

1. **Simple Closure** (gap < 2% of path length)
   ```javascript
   closureDistance = distance from first point to last point
   gapPercentage = (closureDistance / pathLength) × 100
   
   if gapPercentage < 2.0:
       shape is closed
   ```

2. **Complex Closure** (gap < 10%, but shape rotates)
   ```javascript
   Calculate total angle change along path
   rotationPercentage = (totalAngleChange / 360°) × 100
   
   if gapPercentage < 10.0 AND rotationPercentage >= 70.0:
       shape is closed
   ```

3. **Gap Filling**
   ```javascript
   If closed but gap exists:
       steps = ceil(closureDistance / averageSegmentLength)
       
       for i from 1 to (steps - 1):
           t = i / steps
           interpolate point between last and first
           add to shape
   ```

### 4. Point-in-Polygon Test (Ray Casting)

Used for area calculation when shape is closed.

```javascript
function isPointInPolygon(x, y, polygon):
    inside = false
    
    for each edge (i, j) in polygon:
        xi = polygon[i].x
        yi = polygon[i].y
        xj = polygon[j].x
        yj = polygon[j].y
        
        // Check if ray from point crosses edge
        intersect = ((yi > y) != (yj > y)) AND 
                    (x < (xj - xi) × (y - yi) / (yj - yi) + xi)
        
        if intersect:
            inside = NOT inside  // Toggle
    
    return inside
```

**How it works:**
- Cast a ray from the test point to infinity (e.g., to the right)
- Count how many times it crosses the polygon boundary
- Odd count = inside, Even count = outside

### 5. Statistical Calculations

For each frequency and phase, calculate:

**Mean Radius:**
```javascript
allRadii = [radius1, radius2, ..., radiusN] for all participants
meanRadius = sum(allRadii) / N
```

**Standard Deviation of Radius:**
```javascript
variance = sum((radius - meanRadius)²) / N
stdDev = sqrt(variance)
```

**Mean Area:**
```javascript
allAreas = [area1, area2, ..., areaN]
meanArea = sum(allAreas) / N
```

**Standard Deviation of Area:**
```javascript
variance = sum((area - meanArea)²) / N
stdDev = sqrt(variance)
```

---

## Averaging Algorithm

### Overview

After analyzing all individual images, the system averages shapes for each frequency and phase combination.

### Step-by-Step Process

#### Step 1: Organize Data

```
Group by:
  - Frequency (31, 62.5, 125, 250, 500, 1000, 2000, 4000, 8000, 12000, 16000 Hz)
  - Phase (Red = in-phase, Blue = out-of-phase)

Example:
  125 Hz, Red: [participant1_shape, participant2_shape, ..., participantN_shape]
  125 Hz, Blue: [participant1_shape, participant2_shape, ..., participantN_shape]
```

#### Step 2: Normalize Contours

**Purpose:** Remove position information, keeping only shape structure.

```javascript
for each participant shape:
    normalizedContour = []
    
    for each point in shape.contour:
        // Shift point relative to shape's own centroid
        relativeX = point.x - shape.centroid.x
        relativeY = point.y - shape.centroid.y
        
        normalizedContour.add({x: relativeX, y: relativeY})
```

**Example:**
```
Original points: [(3.2, 4.5), (3.8, 4.2), ...]
Centroid: (3.5, 4.0)
Normalized: [(−0.3, 0.5), (0.3, 0.2), ...]
```

#### Step 3: Create Angular Bins

Use 360 bins (1° resolution) for averaging (finer than the 180 bins used for individual shapes).

```javascript
bins = Array(360)

for each bin:
    bins[angle] = {
        radii: [],      // All participant radii at this angle
        points: []      // All participant points at this angle
    }
```

#### Step 4: Populate Bins

For each participant's normalized contour:

```javascript
for each point in normalizedContour:
    dx = point.x
    dy = point.y
    
    angle = atan2(dy, dx)
    radius = sqrt(dx² + dy²)
    
    binIndex = floor(((angle + π) / (2π)) × 360)
    
    bins[binIndex].radii.add(radius)
    bins[binIndex].points.add(point)
```

#### Step 5: Calculate Average at Each Angle

```javascript
avgContour = Array(360)

for angle from 0 to 359:
    if bins[angle].radii is not empty:
        // Average radius at this angle
        avgRadius = mean(bins[angle].radii)
        
        // Convert back to Cartesian coordinates
        angleInRadians = (angle / 360) × 2π - π
        
        avgContour[angle] = {
            x: avgRadius × cos(angleInRadians),
            y: avgRadius × sin(angleInRadians),
            radius: avgRadius,
            count: number of participants with data at this angle
        }
    else:
        avgContour[angle] = null  // No data at this angle
```

#### Step 6: Calculate Average Centroid

```javascript
// Average the centroids from all participants
avgCentroidX = mean([shape1.centroid.x, shape2.centroid.x, ...])
avgCentroidY = mean([shape1.centroid.y, shape2.centroid.y, ...])
```

#### Step 7: Calculate Statistics

```javascript
// Collect all radii from all participants
allRadii = [participant1.avgRadius, participant2.avgRadius, ...]

meanRadius = mean(allRadii)
stdDevRadius = standardDeviation(allRadii)

// Collect all areas
allAreas = [participant1.area, participant2.area, ...]

meanArea = mean(allAreas)
stdDevArea = standardDeviation(allAreas)
```

#### Step 8: Create Standard Deviation Envelope

For visualization purposes:

```javascript
upperBound = []
lowerBound = []

for each angle in avgContour:
    if avgContour[angle] exists:
        // Calculate std dev at this angle
        radiiAtAngle = bins[angle].radii
        mean = avgContour[angle].radius
        stdDev = standardDeviation(radiiAtAngle)
        
        upperRadius = mean + stdDev
        lowerRadius = max(0, mean - stdDev)
        
        angleRad = (angle / 360) × 2π - π
        
        upperBound[angle] = {
            x: upperRadius × cos(angleRad),
            y: upperRadius × sin(angleRad)
        }
        
        lowerBound[angle] = {
            x: lowerRadius × cos(angleRad),
            y: lowerRadius × sin(angleRad)
        }
```

### Result

For each frequency and phase:
```javascript
{
    contour: [...],           // 360 averaged points
    centroid: {x, y},         // Average centroid position
    meanRadius: 3.45,         // Mean distance from centroid
    sdRadius: 0.82,           // Standard deviation of radius
    meanArea: 12.34,          // Mean enclosed area
    sdArea: 3.21,             // Standard deviation of area
    n: 42,                    // Number of participants
    upperBound: [...],        // Mean + 1 SD
    lowerBound: [...]         // Mean - 1 SD
}
```

---

## Visualization Generation

### Canvas Setup

**Size:** 600×600 pixels for display
- Maintains 20×20 unit coordinate system (-10 to +10)
- Scale: 30 pixels per unit (600 / 20)

### Drawing Layers (Bottom to Top)

#### 1. Background Grid
```
Light gray lines every 2 units
Range: -10 to +10 in both X and Y
Color: #e5e7eb
Line width: 1 pixel
```

#### 2. Axis Lines
```
X-axis (horizontal): y = 0
Y-axis (vertical): x = 0
Color: #9ca3af
Line width: 2 pixels
```

#### 3. Reference Circle
```
Radius: 3 units (from original drawing tool)
Purpose: Visual reference for size comparison
Style: Dashed gray line
Color: #9ca3af
Line width: 2 pixels
Dash pattern: [5, 5]
```

#### 4. Origin Point
```
Position: (0, 0)
Size: 5-pixel radius circle
Color: #1e40af (blue)
Purpose: Mark the center of coordinate system
```

#### 5. Individual Shapes (Faded)
```
For each participant shape:
    Opacity: 30%
    Color: Red (#ef4444) or Blue (#3b82f6) depending on phase
    Line width: 2 pixels
    Purpose: Show variability across participants
```

Drawing method:
```javascript
Draw individual shape:
    1. Shift contour by averaged centroid
    2. Draw closed path connecting all points
    3. Use semi-transparent stroke
```

#### 6. Standard Deviation Envelope
```
Fill between mean + 1SD and mean - 1SD
Color: Light red/blue with 20% opacity
Purpose: Show variability range
```

Drawing method:
```javascript
1. Draw path: mean contour + 1 SD (outer)
2. Continue path: mean contour - 1 SD (inner, reversed)
3. Close path
4. Fill with semi-transparent color
```

#### 7. Averaged Shape (Bold)
```
Line width: 4 pixels
Color: Solid red or blue (full opacity)
Style: Smooth closed curve
Purpose: Main result - the average shape
```

Drawing method:
```javascript
1. Start at first averaged point
2. Connect all 360 points in order
3. Close path back to start
4. Draw thick stroke
```

#### 8. Frequency Label
```
Position: Top center
Text: "XXX Hz" (e.g., "125 Hz")
Font: 20px bold sans-serif
Color: #1f2937
```

#### 9. Statistics Box
```
Position: Bottom left
Background: White with slight transparency
Border: 1px gray
Content:
    N: XX participants
    Mean Radius: X.XXX units
    SD Radius: X.XXX units
    Mean Area: X.XXX sq units
    SD Area: X.XXX sq units
```

### Color Coding

**Red Phase (In-Phase):**
- Individual shapes: rgba(239, 68, 68, 0.3)
- SD envelope: rgba(239, 68, 68, 0.2)
- Averaged line: rgb(220, 38, 38)

**Blue Phase (Out-of-Phase):**
- Individual shapes: rgba(59, 130, 246, 0.3)
- SD envelope: rgba(59, 130, 246, 0.2)
- Averaged line: rgb(37, 99, 235)

### Export Format

- **Format:** PNG
- **Resolution:** 600×600 pixels
- **Color space:** RGB
- **Transparency:** Supported (but background is white)
- **Quality:** Lossless

---

## Usage Guide

### Basic Workflow

1. **Prepare Your Data**
   - Collect PNG files from Sound Object drawing tool
   - Organize in folders (optional - tool handles any structure)
   - Can create nested ZIPs for easy upload

2. **Upload Files**
   - Click file upload area
   - Select multiple PNGs OR one or more ZIP files
   - Tool automatically extracts nested ZIPs

3. **Start Analysis**
   - Click "🔬 Analyze & Generate Visualizations"
   - Watch progress bar (shows count, %, time remaining)
   - Can cancel anytime with confirmation dialog

4. **Review Results**
   - View averaged shapes for each frequency
   - Check statistics table
   - Read data summary

5. **Export Data**
   - Download individual figures (per frequency)
   - Export all figures at once
   - Save statistics for further analysis

### File Naming Convention

**Expected format:**
```
ParticipantID_FrequencyHz_dBdB.png

Examples:
P-001_125Hz_90dB.png
P-002_250Hz_85dB.png
John_Doe_1000Hz_80dB.png
```

**Parsed components:**
- Participant ID: Everything before first underscore
- Frequency: Number before "Hz"
- dB level: Number before "dB" (used for phase detection)

### Supported Frequencies

The tool recognizes these standard frequencies (in Hz):
```
31, 62.5, 125, 250, 500, 1000, 2000, 4000, 8000, 12000, 16000
```

Images with other frequencies will be skipped with a warning.

### ZIP File Structure

**Flat structure (all PNGs in one ZIP):**
```
data.zip
  ├── P-001_125Hz_90dB.png
  ├── P-001_250Hz_85dB.png
  ├── P-002_125Hz_90dB.png
  └── ...
```

**Nested structure (participant folders as ZIPs):**
```
all_participants.zip
  ├── P-001_drawings.zip
  │     ├── P-001_125Hz_90dB.png
  │     ├── P-001_250Hz_85dB.png
  │     └── ...
  ├── P-002_drawings.zip
  │     ├── P-002_125Hz_90dB.png
  │     └── ...
  └── ...
```

Both structures work! The tool recursively finds all PNGs.

### Troubleshooting

**"No PNG files found"**
- Check that files are actually .png format
- Verify ZIP files aren't password protected
- Ensure files aren't corrupted

**"Invalid filename format"**
- Check naming convention: Name_FreqHz_dBdB.png
- Frequency must be a number followed by "Hz"
- dB must be a number followed by "dB"

**"No contours found in image"**
- Image might be blank (all white)
- Drawing might be too faint
- Grid might cover the entire image

**Processing is slow**
- Normal for large datasets (462 images = ~7 seconds)
- Check browser console for errors
- Try closing other browser tabs
- Ensure you're not running low on RAM

---

## Technical Specifications

### System Requirements

**Browser:**
- Chrome 90+ (recommended)
- Firefox 88+
- Safari 14+
- Edge 90+

**Features used:**
- FileReader API
- Canvas 2D API
- Promises/Async
- ES6+ JavaScript
- JSZip library

**Memory:**
- ~2-3 MB per image processed
- 500 images ≈ 1-1.5 GB peak RAM usage
- Automatically managed (garbage collected)

### Performance

**Processing speed:**
- ~15-20 images per second (optimized)
- Batch size: 10 images parallel
- 462 images: ~7-8 seconds total

**Optimizations:**
- Bounding box detection (skips empty regions)
- Canvas object pooling
- Reduced angular resolution (180 bins)
- Batch parallel processing

### Accuracy Guarantees

**Identical to original tool:**
- ✅ Coordinate system (1000×1000px, ±10 units)
- ✅ Centroid calculation (uniform resampling)
- ✅ Area calculation (grid-based with brush radius)
- ✅ All mathematical formulas

**Minor differences (imperceptible):**
- Angular resolution: 180 bins vs 360 bins (2° vs 1°)
- Impact: <0.2% difference, visually identical

### Data Privacy

**All processing happens in your browser:**
- ✅ No data sent to servers
- ✅ No data stored permanently
- ✅ No tracking or analytics
- ✅ Files stay on your computer

**Temporary data:**
- Images loaded into browser memory
- Cleared when you close the tab
- Canvas pool cleared when processing ends

### Limitations

**File size:**
- No hard limit, but browser memory limited
- Practical: ~1000-2000 images max
- For larger datasets, process in batches

**Image format:**
- Must be PNG format
- Must be 1000×1000 pixels
- Must match expected structure

**Browser compatibility:**
- Requires modern browser
- Won't work in IE11 or older browsers
- Mobile browsers: may be slow with large datasets

---

## Algorithm Complexity Analysis

### Time Complexity

**Per image:**
- Pixel scan: O(n) where n = pixels in bounding box (~400,000)
- Centroid: O(m × k) where m = samples (~200), k = segments (~5000)
- Area: O(w × h × s) where w,h = grid dimensions (~200×200), s = segments
- Radial binning: O(p) where p = drawn pixels (~10,000)
- **Total: O(n + mk + whs + p) ≈ O(n)** dominated by pixel scan

**Overall:**
- N images: O(N × n) = O(N) linear scaling
- With parallelism: O(N/c × n) where c = cores/batch size

### Space Complexity

**Per image:**
- Raw pixels: 1000 × 1000 × 4 bytes = 4 MB
- Drawn pixels: ~10,000 points × 20 bytes = 200 KB
- Resampled points: ~200 points × 16 bytes = 3 KB
- **Total: ~4.2 MB per image**

**Overall:**
- N images with batch size b: O(b) active images in memory
- Results: O(N) all processed data kept
- Peak: O(b × 4MB + N × 200KB)

---

## References & Resources

### Original Tool
- Sound Object Drawing Tool (companion to this analyzer)
- Creates the 1000×1000px PNG images
- Defines coordinate system and grid

### Libraries Used
- **JSZip** (v3.10.1): ZIP file extraction
- **Tailwind CSS** (CDN): Styling
- **Native Canvas API**: Drawing and image processing

### Research Background
- UCI Hearing & Speech Lab
- Sound perception visualization studies
- Cross-participant analysis methods

### Related Concepts
- **Centroid calculation**: Center of mass computation
- **Ray casting**: Point-in-polygon testing
- **Polar coordinates**: Radial distance representation
- **Statistical aggregation**: Mean and standard deviation

---

## Appendix: Example Calculation Walkthrough

### Full Example: Processing One Image

**Input:** `P-042_125Hz_90dB.png` (1000×1000 pixels)

**Step 1: File parsing**
```
Filename: P-042_125Hz_90dB.png
Participant: "P-042"
Frequency: 125 Hz
dB: 90 dB
Phase: Red (90 ≥ 90 threshold)
```

**Step 2: Bounding box detection**
```
Quick scan found content in region:
minX: 280, maxX: 720
minY: 200, maxY: 680

With padding:
minX: 260, maxX: 740
minY: 180, maxY: 700

Pixels to scan: 480 × 520 = 249,600 (vs 1,000,000 full scan)
Savings: 75%
```

**Step 3: Pixel extraction**
```
Scanned 249,600 pixels
Found 8,432 drawn pixels (not white, not grid)

Color check:
Sum of red: 2,138,432
Sum of blue: 1,456,789
Phase: RED (red > blue)
```

**Step 4: Coordinate conversion**
```
Example pixels in canvas space:
(500, 500) → (0.0, 0.0) unit space
(650, 350) → (3.0, 3.0) unit space
(350, 650) → (-3.0, -3.0) unit space

All 8,432 pixels converted
```

**Step 5: Centroid calculation**
```
Path length: 42.3 units
Number of samples: max(100, floor(42.3 / 0.1)) = 423

Uniform resampling created 423 evenly-spaced points

Sum of X: 523.7
Sum of Y: -128.4
Count: 423

Centroid: (1.238, -0.304) in unit space
```

**Step 6: Area calculation**
```
Shape closed: Yes (gap 1.2% of path)
Brush radius: 0.05 units
Resolution: 0.075 units

Bounding box: (-3.2, -3.8) to (4.5, 3.1)
Grid size: ~103 × 92 cells

Scan 9,476 grid cells:
Inside polygon: 3,245 cells
Near edge: 1,834 cells
Total painted: 5,079 cells

Cell area: 0.075 × 0.075 = 0.005625
Total area: 5,079 × 0.005625 = 28.57 sq units

Perimeter: 42.3 units
Outline adjustment: 42.3 × 0.1 × 0.5 = 2.12 sq units
Final area: 28.57 - 2.12 = 26.45 sq units
```

**Step 7: Radial binning**
```
For each of 8,432 pixels:
  Calculate angle and radius from centroid
  Assign to one of 180 angular bins
  Keep maximum radius per bin

Result: 180 points forming smooth contour
Example:
  Bin 0 (0°): radius 3.42 units
  Bin 45 (90°): radius 3.89 units
  Bin 90 (180°): radius 3.15 units
  Bin 135 (270°): radius 3.67 units

Average radius: 3.51 units
```

**Step 8: Result**
```javascript
{
    participant: "P-042",
    frequency: 125,
    db: 90,
    phase: "red",
    centroid: { x: 1.238, y: -0.304 },
    contour: [ ...180 points... ],
    avgRadius: 3.51,
    area: 26.45,
    allPoints: [ ...8432 unit points... ]
}
```

**Processing time:** ~15ms for this image

---

## Version History

### v2.0 (Current)
- Added bounding box optimization
- Added canvas pooling
- Reduced angular bins (180 vs 360)
- Added progress tracking with time estimates
- Added confirmation dialog for cancellation
- Immediate abort on cancellation
- Verified exact calculation accuracy

### v1.0 (Initial)
- Basic image processing
- Centroid and area calculations
- Averaging algorithm
- Visualization generation
- ZIP file support
- Batch processing

---

## License & Attribution

**Tool developed for:**
UCI Hearing & Speech Lab

**Purpose:**
Research on sound perception visualization

**Data:**
Participant drawings remain property of UCI

**Code:**
For research use

---

## Support & Contact

For questions about:
- **Algorithm details**: See this documentation
- **Technical issues**: Check browser console for errors
- **Research collaboration**: Contact UCI Hearing & Speech Lab

---

**End of Documentation**

*Last updated: November 2024*
*Version: 2.0*
