# Instructions for Creating a 3D Munsell Color Space Visualizer

## Overview
Create a web-based 3D interactive visualizer for the Munsell color space using Three.js. The visualization should accurately represent the irregular shape of human color perception, with Value on the Z-axis, Hue as the angular position, and Chroma as the radial distance.

## Implementation Requirements

### 1. Core Visualization Structure

**Coordinate System:**
- **Y-axis (vertical):** Munsell Value (0-10, lightness from black to white)
- **Angular position (X/Z plane):** Munsell Hue (0-360 degrees, cycling through Red → Yellow → Green → Blue → Purple → Red)
- **Radial distance from center:** Munsell Chroma (0-18+, saturation from gray to vivid)

**Key Point:** The Munsell color space has an IRREGULAR boundary. Different hues reach different maximum chroma at different value levels. This is critical to visualize correctly.

### 2. Color Science - Accurate Conversion Pipeline

You MUST implement proper color conversion to ensure accurate color rendering:

**Munsell → CIE XYZ → Linear RGB → sRGB**

#### Step 1: Munsell (H, V, C) → CIE Lab
The Munsell system correlates closely with CIE Lab:
- L* = V × 10 (Munsell Value 0-10 maps to L* 0-100)
- a* = C × 5 × cos(H_radians)
- b* = C × 5 × sin(H_radians)

#### Step 2: CIE Lab → XYZ (D65 illuminant)
Use the standard CIE Lab to XYZ conversion:
```
fy = (L* + 16) / 116
fx = fy + (a* / 500)
fz = fy - (b* / 200)

For each f-value:
  If f > 0.206897: value = f³
  Else: value = (f - 16/116) / 7.787

X = value_x × 0.95047  (D65 white point)
Y = value_y × 1.00000
Z = value_z × 1.08883
```

#### Step 3: XYZ → Linear RGB (sRGB primaries)
Use the sRGB transformation matrix:
```
R_linear =  3.2404542×X - 1.5371385×Y - 0.4985314×Z
G_linear = -0.9692660×X + 1.8760108×Y + 0.0415560×Z
B_linear =  0.0556434×X - 0.2040259×Y + 1.0572252×Z
```

#### Step 4: Linear RGB → sRGB (gamma correction)
Apply sRGB gamma correction:
```
For each channel:
  If linear ≤ 0.0031308: sRGB = 12.92 × linear
  Else: sRGB = 1.055 × (linear^(1/2.4)) - 0.055
```

#### Step 5: Gamut Handling
- Check if RGB values are in [0, 1] range
- Clamp out-of-gamut colors for display
- Optionally track which colors are out of sRGB gamut
- Consider wide gamut support for Display P3 and Rec.2020 displays

### 3. Munsell Color Space Boundary (Critical!)

The maximum chroma varies by hue AND value. Implement a `getMaxChroma(hue, value)` function:

**Base Chroma by Hue (at optimal value ~5-6):**
- Yellow (60-100°): 16 (highest chroma)
- Yellow-Red (20-60°), Yellow-Green (100-130°): 14
- Red (0-20°, 340-360°): 14
- Green (130-180°): 12
- Blue-Green to Blue (180-230°): 10
- Purple-Blue (230-260°): 9
- Purple (260-300°): 10
- Red-Purple (300-340°): 12

**Value Factor (modulates maximum chroma):**
- Value 0-1: 30% of base
- Value 2: 50% of base
- Value 3: 75% of base
- Value 4-5: 95% of base
- Value 6: 100% of base (peak)
- Value 7: 90% of base
- Value 8: 70% of base (50% for purple/blue)
- Value 9: 40% of base (25% for purple/blue)
- Value 10: 20% of base

This creates the characteristic "bulging" shape where yellows extend far out but purples are very constrained, especially at high values.

### 4. 3D Rendering with Three.js

**Scene Setup:**
- Dark background (e.g., #1a1a1a)
- Perspective camera with FOV ~60°
- Ambient light + directional light for depth
- WebGL renderer with antialiasing

**Point Cloud Generation:**
Generate points at regular intervals:
- Value: steps of 0.5 to 1 (user adjustable)
- Hue: steps of 9° to 36° (user adjustable)
- Chroma: steps of 0.5 to 2, from 0 to getMaxChroma(hue, value)

For each point:
1. Convert Munsell (hue, value, chroma) to RGB
2. Convert cylindrical coordinates to Cartesian:
   - x = chroma × cos(hue_radians)
   - y = value
   - z = chroma × sin(hue_radians)
3. Store position and color

**Point Material:**
- Use THREE.PointsMaterial with vertexColors
- Size: 0.4 with sizeAttenuation
- Transparency: 0.9 opacity
- Additive blending for vibrant appearance

**Boundary Mesh (Optional but Recommended):**
Create a semi-transparent surface mesh connecting the maximum chroma points at each value level:
1. For each value level (0-10 in steps of 1)
2. For each hue (0-360 in steps of 18°)
3. Create vertex at maximum chroma
4. Connect vertices between adjacent value levels with triangular faces
5. Use MeshPhongMaterial with vertexColors, opacity 0.3, DoubleSide

### 5. Axes and Visual Guides

**Value Axis:**
- White line from (0,0,0) to (0,12,0)
- Small white spheres at values 0, 2, 4, 6, 8, 10

**Chroma Circles:**
- Horizontal circles at y=0 showing chroma levels (radius 5, 10, 15)
- Gray, semi-transparent

**Hue Direction Indicators:**
- Lines radiating from center along cardinal hues:
  - 5R (0°), 5YR (36°), 5Y (72°), 5GY (108°), 5G (144°)
  - 5BG (180°), 5B (216°), 5PB (252°), 5P (288°), 5RP (324°)
- Color each line approximately to match its hue direction

### 6. Interactive Controls

**Camera Controls (Manual Implementation Recommended):**
- **Rotation:** Click and drag to orbit around the center point (0, 5, 0)
  - Spherical coordinates: theta (horizontal), phi (vertical)
  - Prevent phi from reaching 0 or π (avoid gimbal lock)
- **Zoom:** Mouse wheel to adjust camera distance (10-60 units)
- **Touch support:** Single-finger drag for rotation
- **Auto-rotate:** Optional continuous rotation

**UI Controls:**
- Point Size slider (1-10, affects material.size)
- Point Density (Low/Medium/High):
  - Low: value step 2, hue step 36°, chroma step 2
  - Medium: value step 1, hue step 18°, chroma step 1
  - High: value step 0.5, hue step 9°, chroma step 0.5
- Show/Hide Axes checkbox
- Show/Hide Boundary Mesh checkbox
- Reset Camera button
- Auto Rotate toggle button

**Status Display:**
- Point count (e.g., "Points: 2,185")
- Optional: Camera position/zoom info

### 7. Performance Considerations

- Start with Medium density (2,000-3,000 points)
- High density can produce 10,000+ points (may be slow on some devices)
- Dispose of geometries/materials when regenerating point cloud
- Use BufferGeometry for efficiency
- Consider using Float32Array for position/color attributes

### 8. User Experience

**Loading State:**
- Show loading spinner while Three.js CDN loads
- Display error message if CDN fails to load
- Console logging for debugging

**Information Panel:**
- Explain the coordinate system
- Note the irregular shape and why it matters
- Highlight interesting features (yellow's high chroma, purple's limitations)

**Error Handling:**
- Try/catch around initialization
- Check if THREE is defined before using
- Provide helpful error messages for common issues

### 9. CDN and Dependencies

Use Three.js from CDN:
- Primary: `https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js`
- Fallback: `https://cdnjs.cloudflare.com/ajax/libs/three.js/r152/three.min.js`

Load dynamically with error handling:
```javascript
const script = document.createElement('script');
script.src = 'CDN_URL';
script.onload = () => { /* initialize app */ };
script.onerror = () => { /* show error */ };
document.head.appendChild(script);
```

### 10. Color Rendering Best Practices

**Critical for Accuracy:**
1. Use proper color space conversions (no shortcuts!)
2. Apply gamma correction correctly
3. Use D65 white point consistently
4. Handle out-of-gamut colors gracefully (clamp for display)
5. Consider display color profiles:
   - sRGB: standard (most displays)
   - Display P3: wide gamut (modern Apple devices)
   - Rec.2020: ultra-wide gamut (future displays)

**Color Management:**
- Render in linear space when possible
- Apply tone mapping if needed
- Avoid double gamma correction

### 11. Expected Visual Result

When working correctly, users should observe:
- A 3D "tree" or "bulging spindle" shape
- Yellow region (90° area) bulges outward significantly
- Blue/Purple regions (210-270°) are much more constrained
- The shape "pinches" at Value 0 (black) and Value 10 (white)
- Maximum "girth" around Value 5-6
- Smooth color transitions when rotating
- Vibrant, accurate colors that match Munsell notation

### 12. Testing and Validation

**Visual Checks:**
- Pure achromatic axis (Chroma=0) should be gray at all values
- Hue transitions should be smooth and circular
- Maximum chroma colors should look vivid but realistic
- No banding or posterization

**Functional Tests:**
- Controls respond smoothly
- No memory leaks when changing density
- Responsive to window resize
- Works on mobile (touch controls)

### 13. Optional Enhancements

- Click on point to show Munsell notation (H V/C)
- Filter by value range (show only certain lightness levels)
- Filter by hue range (show only warm/cool colors)
- Export view as image
- Toggle between different color space visualizations
- Show gamut comparison overlays
- Animation of value slices
- Label hue directions with text

## Technical Architecture

**File Structure:**
- Single self-contained HTML file with embedded CSS and JavaScript
- OR separate HTML, CSS, JS files if preferred

**Execution Flow:**
1. Load HTML/CSS
2. Dynamically load Three.js from CDN
3. Initialize scene, camera, renderer
4. Generate Munsell point cloud with accurate colors
5. Create axes and guides
6. Set up event listeners
7. Start animation loop
8. User interacts via controls

## Key Implementation Details

### Color Conversion Function Signatures

```javascript
function getMaxChroma(hueAngle, value) → number
function munsellToXYZ(hue, value, chroma) → {X, Y, Z}
function XYZToLinearRGB(X, Y, Z) → {r, g, b}
function linearToSRGB(linear) → number
function munsellToRGB(hue, value, chroma) → {r, g, b, inGamut}
```

### 3D Generation Function Signatures

```javascript
function createMunsellSpace(density) → void
function createBoundaryMesh(density) → void
function createAxes() → void
function updateCameraPosition() → void
```

### Event Handlers

```javascript
// Mouse/touch controls
onMouseDown(event) → set isDragging, store position
onMouseMove(event) → if dragging, update camera rotation
onMouseUp(event) → clear isDragging
onWheel(event) → adjust camera distance

// UI controls
onPointSizeChange(value) → update material.size
onDensityChange(value) → regenerate point cloud
onAxesToggle(checked) → set axes.visible
onBoundaryToggle(checked) → set boundaryMesh.visible
onResetCamera() → reset camera position/rotation
onToggleRotation() → toggle autoRotate flag
```

### Animation Loop

```javascript
function animate() {
  requestAnimationFrame(animate);
  if (autoRotate && !isDragging) {
    // Rotate camera
  }
  renderer.render(scene, camera);
}
```

## Common Pitfalls to Avoid

1. **Wrong coordinate mapping:** Ensure Value is on Y-axis (vertical), not Z
2. **Incorrect color conversion:** Don't use simple HSL/HSV - must go through XYZ
3. **Missing gamma correction:** Linear RGB ≠ sRGB
4. **Uniform chroma boundary:** Must implement irregular boundary based on hue/value
5. **Wrong white point:** Use D65, not D50 or others
6. **Performance issues:** Start with lower density, allow user to increase
7. **No error handling:** Check if Three.js loaded, handle WebGL failures
8. **Camera gimbal lock:** Clamp phi angle to avoid looking directly up/down

## Success Criteria

The visualization is successful when:
- ✅ Colors are accurate and match Munsell notation
- ✅ Shape clearly shows irregular boundary (yellow bulges, purple pinches)
- ✅ Smooth, responsive interaction
- ✅ Works across different browsers and devices
- ✅ Helpful UI with clear controls
- ✅ Educational value - helps understand human color perception
- ✅ Performance is acceptable (smooth at 30+ fps)

## Educational Context

The visualization should help users understand:
- Human color perception is not uniform
- Different hues have different saturation limits
- Saturation limits vary with lightness
- The "color solid" is far from a regular cylinder or cone
- Why certain color combinations don't exist (out of gamut)
- The relationship between Munsell, CIE Lab, and RGB color spaces
