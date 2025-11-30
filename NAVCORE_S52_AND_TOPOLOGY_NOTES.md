# NavCore S-52 and Topology Notes

**Source Repository:** QuteNav (C++/Qt/OpenGL)
**Target:** NavCore (Rust + WGPU)
**Purpose:** Document S-57 topology assembly and S-52 conditional symbology for porting reference

---

## 1. Repository Overview

### Key Modules

**S-57/SENC Parsing:**
- `qutenavlib/src/osenc.{h,cpp}` – OSENC binary format parser [osenc.cpp:359–762]
- `qutenavlib/src/s57object.{h,cpp}` – Feature object model with geometry types [s57object.h:299–364]
- `osencreader/src/osencreader.cpp` – Unencrypted SENC reader plugin [osencreader.cpp:30–73]
- `oesureader/src/oesureader.cpp` – Encrypted OESU/O-Charts reader with IPC decryption [oesureader.cpp:44–172]

**S-52 Styling:**
- `src/s52presentation.{h,cpp}` – Core S-52 lookup tables and bytecode execution [s52presentation.cpp:28–143]
- `src/s52functions.{h,cpp}` – All conditional symbology procedures (2619 lines) [s52functions.cpp:1–2619]
- `data/chartsymbols.xml` – Symbol catalog, color palettes, lookup tables (3000+ lines)

**Rendering:**
- `src/chartrenderer.cpp` – OpenGL frame orchestrator [chartrenderer.cpp:93–208]
- `src/s57chart.cpp` – Per-chart drawing with priority-based passes [s57chart.cpp:648–1064]
- `src/chartpainter.cpp` – Multi-pass rendering (opaque → translucent → text → patterns) [chartpainter.cpp:120–225]
- `shaders/opengl-desktop/` – GLSL shaders (14 files: areas, lines, symbols, text)

**Symbol/Text Management:**
- `src/rastersymbolmanager.cpp` – Symbol atlas loading and UV mapping [rastersymbolmanager.cpp:106–124]
- `src/vectorsymbolmanager.cpp` – HPGL vector symbol parsing [vectorsymbolmanager.cpp:85–193]
- `src/glyphmanager.cpp` – FreeType + HarfBuzz text shaping with SDF atlas [glyphmanager.cpp:356–392]

---

## 2. Vector Edge Topology (Area Assembly)

### Main Assembly Flow

**Entry Point:** `Osenc::readChart()` [osenc.cpp:359–762]

**Key Steps:**

1. **Load Edge Tables** [osenc.cpp:565–608]
   - `VECTOR_EDGE_NODE_TABLE_RECORD` (type 96): Maps edge index → vertex array
   - `VECTOR_CONNECTED_NODE_TABLE_RECORD` (type 97): Maps node index → single point
   - Edges stored as: `{index, point_count, float[2*point_count]}`

2. **Parse Area Geometry Records** [osenc.cpp:471–526]
   - `FEATURE_GEOMETRY_RECORD_AREA` (type 82):
     - `contour_count` – number of rings (outer + holes)
     - `triprim_count` – pre-triangulated primitives
     - `edgeVector_count` – boundary edge references
   - Each edge ref: `{begin_node, edge_table_index, end_node, reversed_flag}`

3. **Resolve Edge References** [osenc.cpp:710–752]
   ```cpp
   for (RawEdgeRef ref : area.edges) {
     EdgeElement e;
     if (ref.reversed) {
       e.begin = connected[ref.end];    // Swap for reversal
       e.end = connected[ref.begin];
     } else {
       e.begin = connected[ref.begin];
       e.end = connected[ref.end];
     }
     e.first = edges[ref.index].first;  // Offset into vertex buffer
     e.count = edges[ref.index].count;  // Point count
   }
   ```

4. **Hole Detection** [chartfilereader.cpp:264–293]
   - First edge vector → outer boundary
   - Subsequent edge vectors → holes
   - Uses earcut triangulator: `Triangulator::addPolygon()` / `addHole()`

5. **Winding Order**
   - OSENC reverses winding when `reversed_flag=true` [osenc.cpp:717–720]
   - Triangulator expects CCW outer, CW holes [earcuttessellator.cpp:76–91]

6. **Triangle Optimization** [osenc.cpp:482–516]
   - If `triprim_count < 5`: Store patches separately
   - If `triprim_count ≥ 5`: Batch in 3000-vertex blocks to reduce draw calls

### Key Takeaways for NavCore

- **Edge table is sparse**: Not all indices are present; use HashMap<u32, EdgeData>
- **Reversed flag is critical**: Must swap begin/end nodes, not just reverse vertex order
- **Holes are implicit**: Determined by contour position in edge vector list, no explicit flag
- **Pre-triangulated data**: SENC includes triangles; NavCore can use or re-triangulate
- **Connected nodes map integers to points**: Build `HashMap<u32, (f32, f32)>` first
- **Winding consistency**: Validate CCW outer / CW holes before GPU upload

---

## 3. S-52 Conditional Symbology (CS Procedures)

### CS Catalog

| CS Name | Feature Classes | Key Inputs | Effect | Code Location |
|---------|----------------|------------|--------|---------------|
| **CSDepthArea01** | DEPARE | DRVAL1, DRVAL2, safety depth | Selects area color based on depth range vs safety threshold | [s52functions.cpp:598–610] |
| **CSDepthContours02** | DEPCNT | VALDCO, safety contour | Emphasizes safety contour with bold line | [s52functions.cpp:240–261] |
| **CSLights05** | LIGHTS | SECTR1/2, VALNMR, COLOUR | Renders light sectors with rotation & color | [s52functions.cpp:1176–1236] |
| **CSObstruction04** | OBSTRN, UWTROC | VALSOU, WATLEV, safety depth | Adds danger symbols for shallow obstructions | [s52functions.cpp:1671–1688] |
| **CSResArea02** | RESARE | CATREA | Restricted area with priority modifiers | [s52functions.cpp:777–794] |
| **CSWrecks02** | WRECKS | CATWRK, VALSOU, WATLEV | Danger/non-danger wreck symbols | [s52functions.cpp:509–544] |
| **CSSoundings02** | SOUNDG | Depth values | Renders sounding points with depth labels | [s52functions.cpp:453–479] |

### Depth Area (CSDepthArea01) Pseudocode

```rust
fn cs_depth_area_01(drval1: Option<f64>, drval2: Option<f64>, safety_depth: f64) -> (Color, Priority) {
    let shallow = drval1.unwrap_or(0.0);
    let deep = drval2.unwrap_or(f64::MAX);

    if deep <= safety_depth {
        // Entire area is shallow/dangerous
        (DEPVS_color, AREA_PLAIN)  // Very shallow pink
    } else if shallow >= safety_depth {
        // Entire area is safe
        (DEPMD_color, AREA_PLAIN)  // Medium depth blue
    } else {
        // Area spans safety threshold
        (DEPMS_color, AREA_PLAIN)  // Mixed depth cyan
    }
}
```
[s52functions.cpp:598–610]

### Depth Contours (CSDepthContours02) Pseudocode

```rust
fn cs_depth_contours_02(valdco: f64, safety_contour: f64) -> LineStyle {
    if (valdco - safety_contour).abs() < 0.01 {
        // This IS the safety contour
        LineStyle {
            pattern: SOLD,
            width: 0.64,  // 2× normal
            color: DEPSC,  // Safety contour color
        }
    } else {
        // Regular depth contour
        LineStyle {
            pattern: SOLD,
            width: 0.32,
            color: DEPCN,
        }
    }
}
```
[s52functions.cpp:240–261]

### CS Execution Model

**Bytecode Interpreter:** [s52presentation.cpp:28–73]
- CS procedures compiled to bytecode at load time
- Bytecode ops: `SY` (symbol), `LS` (line), `AC` (area color), `AP` (area pattern), `TX` (text)
- `Lookup::run_bytecode()` executes ops, returns `PaintData` objects

**Priority Modifiers:** [s52functions.cpp:809–827, 1703–1765]
- `CSResArea02::modifiers()` and `CSObstruction04::modifiers()` return `PriorityData`
- Override default display priority (0–9) based on safety relevance

### Porting Notes for NavCore

- **CPU-side evaluation**: All CS logic stays in Rust; evaluate per-feature before GPU upload
- **Output to GPU**: Color index, symbol ID, line style enum, z-order/priority
- **Mariner params**: Safety depth, shallow/deep contours must be configurable at runtime
- **Bytecode unnecessary**: Rust can use direct function dispatch or match expressions
- **Priority system critical**: Implement 10-level priority (0–9) with separate opaque/translucent passes
- **Missing/stubbed**: Check for unimplemented CS (e.g., AREA_EXT record type 84 not handled)

---

## 4. Symbol Atlas Layout (Point Symbols & Patterns)

### Atlas Structure

**Raster Symbols:** [rastersymbolmanager.cpp:106–124]
- **Source**: `data/rastersymbols-{day,dusk,dark}.png` – pre-rendered symbol atlases
- **Format**: Single PNG per color table mode, packed rectangles (not uniform grid)
- **Metadata**: Parsed from `chartsymbols.xml` `<symbol>` entries

**XML Mapping Example:**
```xml
<symbol RCID="1234" name="BOYLAT13">
  <bitmap>
    <graphics-location x="512" y="256" width="32" height="40"/>
    <pivot x="16" y="36"/>  <!-- Anchor point: 4px from bottom -->
    <distance min="0" max="0"/>  <!-- Always visible -->
  </bitmap>
  <color-ref>CHMGDCHBLK</color-ref>
</symbol>
```

**UV Coordinate Calculation:** [rastersymbolmanager.cpp:106–124]
```cpp
// Normalized texture coords (0–1 range)
texUL = QPointF(x / atlas_width, y / atlas_height);
texLR = QPointF((x + width) / atlas_width, (y + height) / atlas_height);

// Pivot in GL coordinates (relative to symbol origin)
pivot_offset = QPointF(pivot_x - width/2, height/2 - pivot_y);
```

### Symbol ID Mapping

**Hash Table Lookup:** [s52presentation_p.cpp:166–188]
- `QHash<QString, SymbolData>` keyed by symbol name (e.g., "BOYLAT13")
- `SymbolData` contains: UV rect, pivot, advance (for patterns), element list (vector symbols)

### Sample Symbol Table

| Symbol ID | Atlas Rect | Pivot (px) | UV Coords | Code Ref |
|-----------|------------|------------|-----------|----------|
| BOYLAT13 | (512,256,32,40) | (16,36) | UL:(0.5,0.25) LR:(0.531,0.289) | [chartsymbols.xml:L2340] |
| TOWERS65 | (64,128,24,48) | (12,44) | UL:(0.0625,0.125) LR:(0.0859,0.172) | [chartsymbols.xml:L2680] |

### Vector Symbols (HPGL)

**HPGL Parsing:** [hpglopenglparser.cpp:29–311]
- Parses HPGL commands from `<vector>` XML elements
- Outputs colored triangle strips (no texture, pure geometry)
- Uses earcut triangulator for filled polygons [hpglopenglparser.cpp:290–307]

**Example HPGL:**
```
SP1;PU512,512;PD768,512,768,768,512,768,512,512;  // Draw square
PM0;FP;PM2;                                        // Fill polygon
```

### Porting Notes for NavCore

- **Separate atlases per mode**: Day/Dusk/Night need distinct textures (different colors)
- **Pivot is critical**: Incorrect pivot causes symbols to float/sink relative to map position
- **No uniform grid**: Use rect-packing or accept XML layout as-is
- **Scale-dependent visibility**: `<distance>` min/max controls zoom-based culling (CPU-side)
- **Vector symbols**: Store as static geometry, instance per feature with rotation matrix
- **Color modulation**: Raster symbols use `<color-ref>` for palette-based recoloring

---

## 5. Line Pattern Definitions (Dash Styles)

### Pattern Encoding

**LineType Enum:** [types.h:322–325]
```cpp
enum class LineType: uint {
  Solid  = 0x3ffff,   // 18 bits all on
  Dashed = 0x3ffc0,   // 10 on, 6 off, 2 on
  Dotted = 0x30c30,   // 2 on, 4 off, 2 on, 4 off, 2 on, 4 off
};
```

**Bitwise Pattern Matching:** [shaders/opengl-desktop/chartpainter-lines.frag:7–30]
```glsl
uniform uint pattern;           // 18-bit pattern
in float texCoord;              // Distance along line
flat in float segmentLength;

void main() {
  const uint PAT_LEN = 18u;
  const uint SOLID_PATT = 0x3ffffu;

  if (pattern == SOLID_PATT) {
    color = base_color;
  } else {
    float s = mod(texCoord, segmentLength);
    uint bit = uint(PAT_LEN * s / segmentLength);
    if ((pattern & (1u << bit)) == 0u) {
      discard;  // Gap in dash pattern
    }
    color = base_color;
  }
}
```

### Line Width Conversion

**MM to GL Units:** [types.h:346–348, s52functions.cpp:177]
```cpp
constexpr float LineWidthMM(float lw) {
  return lw * 0.32;  // Calibrated multiplier
}
```

**Platform DPI Scaling:** [platform.cpp:88–94]
```cpp
float display_line_width_scaling() {
  const float base_dpi = 96.0;
  return dots_per_inch_x() / base_dpi;
}
```

### Lookup Table Integration

**S-52 Line Styles:** [chartsymbols.xml lookups]
- `LS(SOLD,1,CHBLK)` → Solid, 0.32mm, chart black
- `LS(DASH,2,CHMGD)` → Dashed, 0.64mm, magenta
- `LS(DOTT,1,CHRED)` → Dotted, 0.32mm, red

**Complex Line Styles (LC):** [s52functions.cpp:251–293]
- Uses vector symbol instances placed along line path
- `LineCalculator::calculate()` computes symbol transforms [linecalculator.cpp:35–83]
- Symbol advance distance determines spacing

### Porting Notes for NavCore

- **GPU fragment shader**: Implement 18-bit bitmask pattern matching
- **texCoord**: Requires line-integral distance passed from vertex shader
- **Miter joins**: Use geometry shader or CPU tessellation [geomutils.cpp:136–218]
- **DPI scaling**: Apply on CPU before uploading line width uniform
- **Complex lines**: CPU-side symbol placement, GPU instanced rendering
- **Pattern phase**: Consider line start offset for continuous patterns across segments

---

## 6. Color Palette Tables (Day/Dusk/Night)

### Palette Structure

**XML Format:** [chartsymbols.xml:3–336]
```xml
<color-table name="DAY_BRIGHT" index="0">
  <color name="CHBLK" r="7" g="7" b="7"/>
  <color name="DEPSC" r="82" g="90" b="92"/>   <!-- Safety contour -->
  <color name="DEPVS" r="255" g="204" b="204"/> <!-- Very shallow -->
  <color name="DEPDW" r="176" g="226" b="255"/> <!-- Deep water -->
  <!-- 40+ more colors -->
</color-table>
```

**Loading Code:** [s52presentation_p.cpp:190–223]
- Parses XML, builds `QHash<QString, QRgb>` per table
- Tables: `DAY_BRIGHT`, `DAY_BLACKBACK`, `DAY_WHITEBACK`, `DUSK`, `NIGHT`

### Key Color Tokens (Day RGB)

| Token | Day RGB | Dusk RGB | Night RGB | Usage | Code Ref |
|-------|---------|----------|-----------|-------|----------|
| CHBLK | (7,7,7) | (7,7,7) | (7,7,7) | Chart black (text, outlines) | [chartsymbols.xml:7] |
| DEPSC | (82,90,92) | (82,90,92) | (41,45,46) | Safety contour emphasis | [chartsymbols.xml:31] |
| DEPVS | (255,204,204) | (255,204,204) | (127,102,102) | Very shallow (danger) | [chartsymbols.xml:37] |
| DEPDW | (176,226,255) | (176,226,255) | (88,113,127) | Deep water (safe) | [chartsymbols.xml:33] |
| LANDA | (204,204,153) | (153,153,102) | (102,102,51) | Land area fill | [chartsymbols.xml:26] |
| LITRD | (255,0,0) | (255,85,85) | (127,0,0) | Red light | [chartsymbols.xml:19] |
| LITGN | (0,255,0) | (85,255,85) | (0,127,0) | Green light | [chartsymbols.xml:20] |

### Runtime Palette Switching

**Active Table:** [s52presentation_p.cpp:58–69]
```cpp
void Presentation::setColorTable(ColorTableType t) {
  m_activeColors = &m_colorTables[t];  // Pointer swap
}
```

**Access Pattern:** [s52presentation.cpp:156–165]
```cpp
QRgb GetColor(quint32 index);         // By palette index
QRgb GetColor(const QString& token);  // By S-52 name
```

### Alpha Variants

**Alpha Enum:** [types.h:319–320]
```cpp
enum class Alpha: quint8 {
  P0 = 0,    // Opaque
  P25 = 1,   // 75% opacity
  P50 = 2,   // 50% opacity
  P75 = 3,   // 25% opacity
  P100 = 4,  // Transparent
};
```

**Application:** [s52functions.cpp:36–59]
- `AC(DEPVS, P25)` → Area color with 75% opacity (for layering)

### Porting Notes for NavCore

- **Rust structure**: `HashMap<String, [RGBA; 5]>` where array index = ColorMode enum
- **GPU binding**: Upload active palette as SSBO or push constants (40–50 colors)
- **Token vs index**: Use token strings during parsing, convert to u8 indices for GPU
- **No gamma correction**: Colors are sRGB; ensure WGPU color space matches
- **Night mode dimming**: Verify multipliers (often 0.5× for night reds/greens)
- **Pattern recoloring**: Raster symbols use palette indices from `<color-ref>`, apply at draw time

---

## 7. Text Placement Rules (Label Decluttering)

### Text Shaping Pipeline

**Architecture:** [textmanager.cpp:28–150]
1. **Worker Thread**: Dedicated thread for HarfBuzz text shaping [textmanager.cpp:31–39]
2. **Glyph Atlas**: Single shared 1024×1024 SDF texture [glyphmanager.cpp:33–45]
3. **Skyline Packing**: Glyph rectangles packed dynamically [glyphmanager.cpp:103–176]
4. **200ms Debounce**: Timer batches shape requests [textmanager.cpp:41]

### Glyph Rendering (SDF)

**TinySDF:** [glyphmanager.cpp:70–78]
- FreeType rasterization → 8px spread signed distance field
- Stored as R8_UNorm texture
- Shader: `smoothstep(0.56, 0.94, distance)` for anti-aliasing [chartpainter-text.frag:L18]

### Text Anchor Calculation

**Justification:** [textmanager.cpp:162–189]
```cpp
// Horizontal pivot (Left=0.0, Centre=-0.5, Right=-1.0)
const float h_pivot = m_pivotHMap[hjust] * bbox.width();

// Vertical pivot (Top=0.0, Centre=0.5, Bottom=1.0)
const float v_pivot = m_pivotVMap[vjust] * bbox.height();

// Body size scaling (S-52 "bodySize" × 0.351mm)
const float scale = bodySize * 0.351 * display_text_size_scaling();

// Final position
pivot = scale * (h_pivot, v_pivot) + offset_mm;
```

### Placement Rules (No Decluttering Found)

**Current Implementation:**
- **No collision detection**: Labels drawn in z-order, may overlap [s57chart.cpp:793–819]
- **Fixed priority**: Text always at priority 8 [s57chart.cpp:461–469]
- **No spatial index**: No quadtree/grid for overlap testing

**Inference from Code:**
- S-52 `TE()` function provides `hjust`, `vjust`, `offset` hints [s52functions.cpp:141–158]
- Features with higher priority (lower number) drawn first → occlude later labels
- Point features: Label at symbol pivot + offset
- Line features: No curved text implementation found
- Area features: Label at computed center (weighted triangle average) [osenc.cpp:720–752]

### Porting Notes for NavCore

- **Implement decluttering**: QuteNav lacks this; NavCore should add:
  - Spatial hash grid for placed label bounding boxes
  - Priority queue: Process features by S-52 priority + feature importance
  - Collision test: Reject labels overlapping higher-priority labels
  - Padding: Add 2–4px margin around text bounds
- **SDF shaders**: Reuse Mapbox TinySDF or implement 8px spread SDF
- **Glyph atlas**: Dynamic packing (skyline/rect-pack) or fixed grid (simpler, wastes space)
- **Text rotation**: QuteNav doesn't rotate text; NavCore may need for tilted maps
- **Font fallback**: QuteNav uses system font; NavCore should embed fallback for consistency

---

## 8. Depth Contour & Safety Contour Generation

### Safety Contour Selection

**Mariner Parameters:** [conf_marinerparams.h:84–88]
- `safetyDepth` – User-configured safe depth (meters)
- `shallowContour` – Shallow water threshold
- `deepContour` – Deep water threshold

**CSDepthContours02 Logic:** [s52functions.cpp:240–261]
```rust
fn select_safety_contour(available_contours: &[f64], safety_depth: f64) -> Option<f64> {
    // Find smallest contour >= safety depth
    available_contours.iter()
        .filter(|&&c| c >= safety_depth)
        .min_by(|a, b| a.partial_cmp(b).unwrap())
        .copied()
}

fn style_contour(valdco: f64, safety_contour: Option<f64>) -> LineStyle {
    match safety_contour {
        Some(sc) if (valdco - sc).abs() < 0.01 => {
            // Emphasize safety contour
            LineStyle { width: 0.64, color: DEPSC, pattern: SOLID }
        },
        _ => {
            // Regular contour
            LineStyle { width: 0.32, color: DEPCN, pattern: SOLID }
        },
    }
}
```

### Depth Area Styling

**CSDepthArea01 Implementation:** [s52functions.cpp:598–610]
```rust
fn style_depth_area(drval1: Option<f64>, drval2: Option<f64>,
                    safety_depth: f64, shallow: f64, deep: f64) -> AreaStyle {
    let min_depth = drval1.unwrap_or(0.0);
    let max_depth = drval2.unwrap_or(f64::MAX);

    // Four depth zones
    if max_depth < shallow {
        AreaStyle { color: DEPVS, pattern: None }  // Very shallow (pink)
    } else if min_depth < shallow && max_depth < safety_depth {
        AreaStyle { color: DEPVS, pattern: None }  // Shallow danger
    } else if min_depth < safety_depth && max_depth > safety_depth {
        AreaStyle { color: DEPMS, pattern: None }  // Mixed (cyan)
    } else if min_depth < deep {
        AreaStyle { color: DEPMD, pattern: None }  // Medium (blue)
    } else {
        AreaStyle { color: DEPDW, pattern: None }  // Deep (dark blue)
    }
}
```

### Contour Collection

**Extracting Available Contours:** [s57chart.cpp:118–148]
```cpp
// Collect all DEPCNT features
for (S57::Object* obj : objects) {
  if (obj->featureTypeCode() == S57::DEPCNT) {
    if (obj->hasAttribute(S57::VALDCO)) {
      double depth = obj->getAttribute(S57::VALDCO).toReal();
      m_contours.append(depth);
    }
  }
}
m_contours.sort();  // Ascending order
```

### Hard-Coded Fallbacks

**Default Values:** [s52functions.cpp:598–610]
- Safety depth: 30m (if not user-configured)
- Shallow contour: 2m
- Deep contour: 30m

### Known Deviations from S-52

**TODO/FIXME Notes:**
- No dynamic contour generation (interpolation between existing contours)
- DRVAL1/DRVAL2 expected; missing attributes → default to 0/MAX (may over-emphasize danger)
- No handling for QUASOU (quality of sounding) attribute

### Porting Notes for NavCore

- **Inputs from user**:
  - `safety_depth_m: f64` (vessel draft + margin)
  - `shallow_contour_m: f64`
  - `deep_contour_m: f64`
- **CPU pre-processing**:
  - Scan chart for all DEPCNT features, build sorted contour list
  - Select safety contour = `min(contours >= safety_depth)`
  - Tag each DEPCNT feature: `is_safety_contour: bool`
- **GPU upload**:
  - DEPCNT: Line width (0.32 or 0.64), color index
  - DEPARE: Color index (DEPVS/DEPMS/DEPMD/DEPDW)
- **Runtime updates**:
  - User changes safety depth → re-evaluate CS, regenerate paint data
  - Don't re-parse geometry, only re-run CS functions
- **Edge case**: DEPARE with no DRVAL1/DRVAL2 → treat as unknown depth, use neutral color

---

## 9. Porting Notes for NavCore (Rust + WGPU)

### Section 2: Vector Edge Topology

**Must replicate exactly:**
- Reversed edge flag logic (swap begin/end nodes)
- Connected node lookup before building edge chains
- Hole detection via edge vector position (first=outer, rest=holes)

**Can simplify:**
- Re-triangulate all areas instead of using pre-triangulated data (cleaner, more robust)
- Use single triangulation library (Lyon, earcutr) instead of multiple tessellators

**CPU vs GPU:**
- CPU: Edge resolution, hole association, triangulation
- GPU: Static triangle mesh (no topology, just vertex/index buffers)

**Suggested Rust structures:**
```rust
struct EdgeTable {
    edges: HashMap<u32, Vec<[f32; 2]>>,  // index → points
    nodes: HashMap<u32, [f32; 2]>,       // index → single point
}

struct AreaGeometry {
    outer: Vec<[f32; 2]>,
    holes: Vec<Vec<[f32; 2]>>,
}
```

**Dangers:**
- Missing edge/node indices → crash; validate all refs exist before deref
- Non-closed rings → triangulation failure; check first == last point

### Section 3: S-52 Conditional Symbology

**Must replicate exactly:**
- CSDepthArea01, CSDepthContours02 depth thresholds
- CSLights05 sector rendering with rotation
- CSObstruction04 danger symbol logic
- Priority modifiers (affects draw order)

**Can simplify:**
- Skip bytecode interpreter; use Rust match expressions
- Pre-compute CS results during chart load (not per-frame)

**CPU vs GPU:**
- CPU: Evaluate all CS functions, produce style metadata
- GPU: Receives final color/symbol/line style, no logic

**Suggested Rust structure:**
```rust
enum CsResult {
    AreaStyle { color_token: u8, alpha: u8, priority: u8 },
    LineStyle { pattern: u32, width: f32, color_token: u8 },
    SymbolStyle { symbol_id: u16, rotation: f32 },
    Text { content: String, hjust: HJust, vjust: VJust },
    PriorityOverride(u8),
}

fn evaluate_cs(feature: &S57Object, mariner: &MarinerParams) -> Vec<CsResult>;
```

**TODO/Unclear:**
- Some CS functions reference external files (e.g., `CSYMBOL` lookup); confirm file locations
- Priority override merge logic (when multiple CS return overrides)

### Section 4: Symbol Atlas Layout

**Must replicate exactly:**
- Pivot/hotspot offsets (critical for alignment)
- Separate atlases for Day/Dusk/Night
- UV normalization (0–1 range)

**Can simplify:**
- Use single atlas per mode (no dynamic packing); accept XML layout
- Skip vector symbols (HPGL) initially; render only raster symbols

**CPU vs GPU:**
- CPU: Symbol ID → UV rect + pivot lookup
- GPU: Instanced quads with UV attrs, sampler binding

**Suggested Rust structure:**
```rust
struct SymbolAtlas {
    texture: wgpu::Texture,  // Per color mode
    symbols: HashMap<String, SymbolDef>,
}

struct SymbolDef {
    uv_rect: [f32; 4],  // u0, v0, u1, v1
    pivot: [f32; 2],    // Offset from center
    size: [u32; 2],     // Pixel width, height
}
```

### Section 5: Line Pattern Definitions

**Must replicate exactly:**
- 18-bit bitmask patterns (Solid, Dashed, Dotted)
- Line width scaling with DPI
- Miter join computation for thick lines

**Can simplify:**
- Omit complex line styles (LC) initially; focus on simple patterns (LS)

**CPU vs GPU:**
- CPU: Miter tessellation (or use geometry shader in WGPU)
- GPU: Fragment shader bitmask discard

**Suggested WGSL:**
```wgsl
fn apply_dash_pattern(pattern: u32, distance: f32, segment_len: f32) -> bool {
    let PAT_LEN: u32 = 18u;
    let s = distance % segment_len;
    let bit = u32(f32(PAT_LEN) * s / segment_len);
    return (pattern & (1u << bit)) != 0u;
}
```

### Section 6: Color Palette Tables

**Must replicate exactly:**
- All 40–50 S-52 color tokens
- Day/Dusk/Night RGB values
- Alpha blending variants (P0, P25, P50, P75)

**Can simplify:**
- Use single mode initially (Day); defer Dusk/Night

**CPU vs GPU:**
- CPU: Parse XML, build lookup table
- GPU: Uniform buffer or SSBO with `[RGBA; 50]`

**Suggested Rust:**
```rust
const PALETTE_SIZE: usize = 50;

struct ColorPalette {
    day: [RGBA; PALETTE_SIZE],
    dusk: [RGBA; PALETTE_SIZE],
    night: [PALETTE_SIZE],
}

fn resolve_color(token: &str, mode: ColorMode, alpha: Alpha) -> RGBA;
```

### Section 7: Text Placement Rules

**Must add (missing in QuteNav):**
- Spatial hash grid for collision detection
- Priority-based placement queue
- Bounding box overlap testing with padding

**Can simplify:**
- Fixed font size (24px); no multi-size support
- No curved text (horizontal only)

**CPU vs GPU:**
- CPU: Placement decisions, glyph shaping
- GPU: Instanced glyph rendering with SDF

**Suggested algorithm:**
```rust
struct PlacedLabel {
    bbox: Rect,
    priority: u8,
}

fn place_labels(candidates: &[LabelCandidate]) -> Vec<PlacedLabel> {
    let mut placed = vec![];
    let mut grid = SpatialHashGrid::new();

    for candidate in candidates.iter().sorted_by_key(|c| c.priority) {
        if !grid.intersects(candidate.bbox.expand(4.0)) {
            placed.push(candidate);
            grid.insert(candidate.bbox);
        }
    }
    placed
}
```

### Section 8: Depth Contour & Safety Contour

**Must replicate exactly:**
- Safety contour = smallest contour >= safety depth
- Four-zone depth area coloring (DEPVS/DEPMS/DEPMD/DEPDW)
- Contour emphasis (2× line width for safety contour)

**Can simplify:**
- Use hard-coded fallback defaults (30m, 2m, 30m) if user doesn't configure

**CPU vs GPU:**
- CPU: CS evaluation, contour selection
- GPU: Receives final line width + color

**Suggested Rust:**
```rust
struct MarinerParams {
    safety_depth_m: f64,
    shallow_contour_m: f64,
    deep_contour_m: f64,
}

fn select_safety_contour(chart_contours: &[f64], params: &MarinerParams) -> Option<f64> {
    chart_contours.iter()
        .filter(|&&c| c >= params.safety_depth_m)
        .min_by(|a, b| a.partial_cmp(b).unwrap())
        .copied()
}
```

**Edge cases:**
- DEPARE missing DRVAL1/DRVAL2: Default to (0.0, f64::MAX) → likely triggers "very shallow"
- No contours >= safety depth: All contours are shallow; emphasize shallowest?

---

## Summary Checklist for NavCore

### Parsing (CPU)
- [ ] SENC binary format reader (record types 1–200)
- [ ] Edge table resolution (types 96, 97)
- [ ] Area assembly with holes (earcut triangulation)
- [ ] S-57 attribute decoding (integer, real, string, list)

### Styling (CPU)
- [ ] Chartsymbols.xml parser (colors, symbols, lookups)
- [ ] CS function implementations (at least: CSDepthArea01, CSDepthContours02, CSLights05)
- [ ] Color palette lookup (Day/Dusk/Night modes)
- [ ] Symbol atlas UV mapping
- [ ] Line pattern encoding (18-bit bitmask)
- [ ] Text shaping (HarfBuzz or rustybuzz)
- [ ] Label decluttering (spatial hash grid)

### Rendering (WGPU)
- [ ] Triangle mesh buffers (areas)
- [ ] Line tessellation (miter joins)
- [ ] Instanced symbol rendering (UV quads)
- [ ] Dash pattern fragment shader
- [ ] SDF text rendering
- [ ] 10-priority multi-pass (opaque → translucent → text)
- [ ] Stencil-based pattern fills

### Runtime
- [ ] Mariner parameter updates (safety depth, etc.)
- [ ] Color table switching (Day/Dusk/Night)
- [ ] Chart zoom/pan viewport culling
- [ ] Incremental chart loading (multi-threaded)

---

**End of Report**
