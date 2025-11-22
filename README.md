This README combines the comprehensive pipeline explanation with specific code references (line numbers and excerpts) derived from the source code.
Markdown

# Sound Object Average Shape Analyzer (v9.6)

![Version](https://img.shields.io/badge/version-9.6.0-blue)
![Type](https://img.shields.io/badge/type-Client_Side-green)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

**"Early Marker Removal & Clean Processing Edition"**

## 📋 Overview

The **Sound Object Analyzer v9.6** is a specialized research tool designed to quantify "sound drawings"—visual representations of sound perception.

**The v9.6 Difference:**
Unlike previous iterations, this version utilizes a **"Clean Processing Pipeline."** It recognizes that participants often draw "centroid markers" (crosshairs/plus signs) to indicate the center of a sound. These markers act as noise in mathematical averaging. This version algorithmically identifies and surgically removes these markers *before* calculating geometric properties, ensuring the final average is based solely on the sound object itself.

---

## 🚀 The Processing Pipeline

The application processes every image through a strict linear sequence:

```mermaid
graph TD
    A[Input Ingestion] --> B[Pixel Classification]
    B --> C[Marker Detection & Removal]
    C --> D[Convex Hull Wrapper]
    D --> E[Physical Calculations]
    E --> F[Statistical Filtering (MAD)]
    F --> G[Radial Binning]
    G --> H[Gaussian Smoothing]
    H --> I[Visualization]


⚙️ Technical Implementation & Code References


1. Recursive Ingestion Engine

The system does not rely on flat file structures. It utilizes a recursive engine to dive into nested ZIP files to locate every valid PNG.
	•	Methodology: If a file is a ZIP, open it. If a ZIP contains another ZIP, call extractFromZip again (recursion).
	•	Location: Lines 146–195
JavaScript

// Lines 150-151: Recursive function definition
async function extractFromZip(zipFile, zipName = '') {
    // ...

    // Lines 169-173: Recursive step for nested zips
    } else if (lowerName.endsWith('.zip')) {
        console.log(`  Found nested ZIP: ${filename}`);
        // ...
        // RECURSIVE CALL:
        await extractFromZip(nestedFile, nestedPath);
    }

2. Pixel Classification (Color Physics)

To distinguish "Red" (In-Phase) from "Blue" (Out-of-Phase) drawings, the system calculates Channel Dominance to account for brush opacity and anti-aliasing. Transparency is filtered out.
	•	Logic: Channel - max(Other_Channels) > 30
	•	Location: Lines 252–267
JavaScript

// Line 257: Alpha check (ignore transparency)
if (a < 128) continue; 

// Lines 260-261: Calculate Dominance
const redDominance = r - Math.max(g, b);
const blueDominance = b - Math.max(r, g);

// Lines 263-267: Assign to Phase
if (redDominance > 30) {
    redPixels.push({ x, y });
} else if (blueDominance > 30) {
    bluePixels.push({ x, y });
}

3. Early Marker Removal (The v9.6 Core)

This is the critical cleaning step. It detects participant-drawn crosshairs (+) and removes them to prevent geometric distortion.

3.1 Detection & Intersection

	•	Methodology: Detect vertical/horizontal lines (>10px), find where they cross, and verify they are symmetrical (similar lengths).
	•	Location: Lines 408–425
JavaScript

// Lines 408-425: Finding Crossings
function findCrossingPoints(verticalLines, horizontalLines) {
    // ...
    // Line 417: Geometry Check - Lines must be similar length (< 10px diff)
    if (Math.abs(vLength - hLength) < 10) { 
        crossings.push({ ... });
    }
}

3.2 Surgical Removal

	•	Methodology: Identify a ~20x20px box around the intersection and filter those pixels out of the array.
	•	Location: Lines 319–334
JavaScript

// Lines 319-327: Define removal zone
crossPoints.forEach(cross => {
    for (let y = cross.y - markerHalfLength; y <= cross.y + markerHalfLength; y++) {
        for (let x = cross.x - markerThickness; x <= cross.x + markerThickness; x++) {
            markerPixels.add(`${x},${y}`);
        }
    }
});

// Line 334: Filter the original array
return pixels.filter(p => !markerPixels.has(`${p.x},${p.y}`));

4. Convex Hull (Monotone Chain)

To handle gaps left by marker removal or "open" drawings, the system wraps the pixels in a Convex Hull (like a rubber band).
	•	Methodology: Sort points X-wise, then build Upper and Lower hulls using the Cross Product to determine turn direction.
	•	Location: Lines 430–462
JavaScript

// Lines 437-439: Cross Product Function (checks turn direction)
function cross(O, A, B) {
    return (A.x - O.x) * (B.y - O.y) - (A.y - O.y) * (B.x - O.x);
}

// Lines 442-445: Build Lower Hull
for (let i = 0; i < points.length; i++) {
    while (lower.length >= 2 && cross(lower[lower.length - 2], lower[lower.length - 1], points[i]) <= 0) {
        lower.pop(); // Remove concave points
    }
    lower.push(points[i]);
}

5. Physical Calculations

The system calculates physical properties based on the Hull.
	•	Area: Uses the Shoelace Formula.
	•	Location: Lines 464–476
JavaScript

// Lines 469-473: Shoelace Formula
for (let i = 0; i < hull.length; i++) {
    const j = (i + 1) % hull.length;
    area += hull[i].x * hull[j].y;
    area -= hull[j].x * hull[i].y;
}
// Line 475: Convert to Grid Units (Scale = 50)
return Math.abs(area / 2) / (SCALE * SCALE);

6. Statistical Filtering (MAD)

To reject outliers (drawings that are too big/small or off-center), the system uses Median Absolute Deviation (MAD), which is more robust than Standard Deviation.
	•	Threshold: Reject if |Area - Median| > 3 * MAD.
	•	Location: Lines 566–611
JavaScript

// Line 573: Calculate MAD
const areaMAD = medianAbsoluteDeviation(areas);

// Lines 575-578: Apply 3x MAD Filter
let filtered = shapes.filter(shape => {
    const areaDiff = Math.abs(shape.area - medianArea);
    return areaDiff < 3 * areaMAD;
});

// Lines 607-611: MAD Calculation Helper
function medianAbsoluteDeviation(values) {
    const med = median(values);
    const deviations = values.map(v => Math.abs(v - med));
    return median(deviations) * 1.4826; // Scale for normal distribution
}

7. Averaging & Smoothing

The final shape is a "Gaussian Smoothed Median Contour."

7.1 Radial Binning

	•	Methodology: Cast 360 rays (one per degree) and take the Median radius (ignores spikes).
	•	Location: Lines 616–661
JavaScript

// Lines 641-644: Median Calculation per Bin
if (bins[i].length > 0) {
    const radius = median(bins[i]); // Robust averaging
    // ...
}

7.2 Gaussian Smoothing

	•	Methodology: Convolve the contour with a Gaussian kernel ($\sigma=2.0$) to remove jagged edges.
	•	Location: Lines 663–688
JavaScript

// Line 669: Calculate Gaussian Weight
const weight = Math.exp(-(i * i) / (2 * sigma * sigma));

// Lines 678-685: Convolution Loop
for (let j = -kernelSize; j <= kernelSize; j++) {
    // ...
    sumX += contour[idx].x * weight;
    sumY += contour[idx].y * weight;
}


💻 Usage Guide

	1	Launch: Open Sound_Object_Analyzer_v9.6_Final.html in a modern browser (Chrome/Edge/Safari).
	2	Upload: Drag and drop your data.
	◦	Supports .png (1000x1000px).
	◦	Supports .zip (and nested .zip files).
	3	Analyze: Click the green "Analyze & Generate" button.
	4	Review:
	◦	Gray Lines: Individual cleaned shapes (Convex Hulls).
	◦	Red Line: Average In-Phase shape.
	◦	Blue Line: Average Out-of-Phase shape.
	5	Export: Click any graph to download the high-res composite PNG.


🛠 Technical Stack

	•	Core: HTML5 Canvas, ES6+ JavaScript.
	•	Libraries: * JSZip (Archive extraction).
	◦	TailwindCSS (UI Styling).
	•	Processing: Client-side (No server upload required).
