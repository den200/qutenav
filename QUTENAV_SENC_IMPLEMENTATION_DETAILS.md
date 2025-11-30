# QuteNav SENC/oeSENC Implementation Details

**For NavCore Rust/WGPU Porting Reference**

This document provides hard evidence from QuteNav's source code on how SENC/oeSENC files are parsed and processed.

---

## 1. EDGE DIRECTION ENCODING (AREAS & LINES)

### Core Data Structures

**File:** `qutenavlib/src/osenc.h`

**RawEdgeRef Struct** [osenc.h:75–80]:
```cpp
struct RawEdgeRef {
  quint32 begin;      // Begin node index
  quint32 end;        // End node index
  quint32 index;      // Edge table index
  bool reversed;      // Direction flag
};
```

**Edge Reference Storage:**
- Edges stored in `VECTOR_EDGE_NODE_TABLE_RECORD` (type 96)
- Connected nodes stored in `VECTOR_CONNECTED_NODE_TABLE_RECORD` (type 97)
- Edge references appear in line/area geometry records

### Edge Direction Encoding

**Answer: UNSIGNED ID + SEPARATE FLAG (version-dependent)**

**Version < 201 (hasReversed=false)** [osenc.cpp:458–462]:
- Edge index stored as **signed integer**
- `index < 0` → reversed edge
- Direction extracted via sign bit

```cpp
const uint stride = hasReversed ? 4 : 3;
// Old format: [begin_node, signed_edge_index, end_node]
ref.index = std::abs(es[stride * i + 1]);
ref.reversed = es[stride * i + 1] < 0;  // Sign indicates direction
```

**Version ≥ 201 (hasReversed=true)** [osenc.cpp:458–462]:
- Edge index stored as **unsigned integer**
- Separate `reversed` field (4th integer)
- More explicit representation

```cpp
// New format: [begin_node, edge_index, end_node, reversed_flag]
ref.index = std::abs(es[stride * i + 1]);
ref.reversed = es[stride * i + stride - 1] != 0;  // Explicit flag
```

**Version Detection** [osenc.cpp:381–384]:
```cpp
{SencRecordType::HEADER_SENC_VERSION, new Handler([&hasReversed] (const Buffer& b) {
    auto version = *reinterpret_cast<const quint16*>(b.constData());
    hasReversed = version > 200;  // Critical threshold
    return true;
  })
}
```

### Reversing Edges During Assembly

**File:** `qutenavlib/src/osenc.cpp`

**Edge Reversal Logic** [osenc.cpp:716–725]:
```cpp
if (ref.reversed) {
  // OSENC-specific: swap begin/end nodes for reversed edges
  e.begin = connected[ref.end];
  e.end = connected[ref.begin];
} else {
  e.begin = connected[ref.begin];
  e.end = connected[ref.end];
}
e.reversed = ref.reversed;
```

**Critical Comment** [osenc.cpp:717–718]:
> "Osenc does not reverse begin and end like the other formats, so do it here"

**Vertex Iteration** [chartfilereader.cpp:295–308]:
```cpp
int ChartFileReader::addIndices(const Edge& e, GL::IndexVector& indices) {
  const int N = e.count;
  if (!e.reversed) {
    indices << e.begin;
    for (int i = 0; i < N; i++) {
      indices << e.first + i;  // Forward: 0, 1, 2, ..., N-1
    }
  } else {
    indices << e.end;
    for (int i = 0; i < N; i++) {
      indices << e.first + e.count - 1 - i;  // Reverse: N-1, N-2, ..., 0
    }
  }
  return e.count + 1;
}
```

### Proposed Rust Structs

```rust
/// Edge reference as stored in SENC file
struct SencEdgeRef {
    begin_node: u32,
    end_node: u32,
    edge_index: u32,
    reversed: bool,
}

/// Resolved edge with vertex data
struct Edge {
    begin_vertex_idx: u32,  // Index into vertex buffer
    end_vertex_idx: u32,    // Index into vertex buffer
    first_coord: u32,       // Start offset in edge table
    count: u32,             // Number of intermediate vertices
    reversed: bool,
}

/// Edge table entry
struct EdgeTableEntry {
    vertices: Vec<[f32; 2]>,  // Coordinate pairs
}

/// Parse edge ref from SENC (version-aware)
fn parse_edge_ref(data: &[i32], version: u16) -> SencEdgeRef {
    if version > 200 {
        // Explicit flag format
        SencEdgeRef {
            begin_node: data[0] as u32,
            end_node: data[2] as u32,
            edge_index: data[1].unsigned_abs(),
            reversed: data[3] != 0,
        }
    } else {
        // Sign-bit format
        SencEdgeRef {
            begin_node: data[0] as u32,
            end_node: data[2] as u32,
            edge_index: data[1].unsigned_abs(),
            reversed: data[1] < 0,
        }
    }
}

/// Iterate edge vertices in correct order
fn emit_edge_indices(edge: &Edge, edge_table: &HashMap<u32, EdgeTableEntry>) -> Vec<u32> {
    let mut indices = Vec::new();

    if !edge.reversed {
        indices.push(edge.begin_vertex_idx);
        for i in 0..edge.count {
            indices.push(edge.first_coord + i);
        }
    } else {
        indices.push(edge.end_vertex_idx);
        for i in 0..edge.count {
            indices.push(edge.first_coord + (edge.count - 1 - i));
        }
    }

    indices
}
```

---

## 2. SENC/oeSENC VERSION FIELD

### Version Header Parsing

**File:** `qutenavlib/src/osenc.cpp`

**Version Record Type** [osenc.h:113]:
```cpp
enum class SencRecordType: quint16 {
  HEADER_SENC_VERSION = 1,  // First record in file
  // ... other types
};
```

**Reading Version** [osenc.cpp:55–56, 155–156, 381–384]:

**In readOutline()** [osenc.cpp:55–56]:
```cpp
read_record<quint16>(buffer, stream, SencRecordType::HEADER_SENC_VERSION);
// qCDebug(CENC) << "senc version =" << *p16;
```

**In readChart()** [osenc.cpp:381–384]:
```cpp
{SencRecordType::HEADER_SENC_VERSION, new Handler([&hasReversed] (const Buffer& b) {
    auto version = *reinterpret_cast<const quint16*>(b.constData());
    // qCDebug(CENC) << "senc version" << version;
    hasReversed = version > 200;  // Key: version > 200 uses explicit reversed flag
    return true;
  })
}
```

### Version Values

**Critical Threshold: 200 vs 201+**

- **Version ≤ 200**: Uses signed edge indices (old format)
- **Version > 200**: Uses unsigned edge indices + explicit reversed flag (new format)
- **Expected value**: **201** (implied from `version > 200` check)

**No explicit validation** - any version accepted, only affects parsing behavior.

### Proposed Rust Implementation

```rust
#[repr(C, packed)]
struct SencHeader {
    record_type: u16,    // Should be 1 (HEADER_SENC_VERSION)
    record_length: u32,  // Total record size
    version: u16,        // Version number
}

fn parse_senc_header(bytes: &[u8]) -> Result<u16, ParseError> {
    if bytes.len() < 8 {
        return Err(ParseError::InsufficientData);
    }

    let record_type = u16::from_le_bytes([bytes[0], bytes[1]]);
    if record_type != 1 {
        return Err(ParseError::InvalidRecordType(record_type));
    }

    let record_length = u32::from_le_bytes([bytes[2], bytes[3], bytes[4], bytes[5]]);
    if record_length < 8 || bytes.len() < record_length as usize {
        return Err(ParseError::InvalidLength);
    }

    let version = u16::from_le_bytes([bytes[6], bytes[7]]);

    // Key decision point for parsing format
    let has_reversed_flag = version > 200;

    Ok(version)
}

struct SencParser {
    version: u16,
    use_explicit_reversed: bool,
}

impl SencParser {
    fn new(version: u16) -> Self {
        Self {
            version,
            use_explicit_reversed: version > 200,
        }
    }
}
```

---

## 3. AREAS PRE-TRIANGULATED?

### Area Geometry Record Structure

**File:** `qutenavlib/src/osenc.h`

**OSENC_AreaGeometry_Record_Payload** [osenc.h:199–208]:
```cpp
struct OSENC_AreaGeometry_Record_Payload {
  double extent_s_lat;
  double extent_n_lat;
  double extent_w_lon;
  double extent_e_lon;
  uint32_t contour_count;      // Number of rings (outer + holes)
  uint32_t triprim_count;       // Number of triangle primitives
  uint32_t edgeVector_count;    // Number of boundary edges
  char edge_data;               // Variable-length payload
};
```

### Decision Logic: Use Pre-Triangulated vs Fallback

**File:** `qutenavlib/src/osenc.cpp`

**Area Parsing** [osenc.cpp:471–550]:

```cpp
{SencRecordType::FEATURE_GEOMETRY_RECORD_AREA, new Handler([&features, &hasReversed] (const Buffer& b) {
    auto p = reinterpret_cast<const OSENC_AreaGeometry_Record_Payload*>(b.constData());

    const char* ptr = &p->edge_data;

    // Skip contour counts
    ptr += p->contour_count * sizeof(int);

    // DECISION POINT: Is area renderable?
    if (S52::IsAreaRenderable(features.last().object->classCode())) {
      TrianglePatchVector ts;

      // OPTIMIZATION: Small triangle counts → store separately
      if (p->triprim_count < 5) {
        ts.resize(p->triprim_count);
        for (uint i = 0; i < p->triprim_count; i++) {
          ts[i].mode = static_cast<GLenum>(*ptr);  // GL_TRIANGLES/STRIP/FAN
          ptr++;
          const auto nvert = read_and_advance<uint32_t>(&ptr);
          ptr += 4 * sizeof(double);  // skip bbox
          ts[i].vertices.resize(nvert * 2);
          memcpy(ts[i].vertices.data(), ptr, nvert * 2 * sizeof(float));
          ptr += nvert * 2 * sizeof(float);
        }
      }
      // OPTIMIZATION: Large triangle counts → batch in 3000-vertex blocks
      else {
        ts << TrianglePatch();
        for (uint i = 0; i < p->triprim_count; i++) {
          if (ts.last().vertices.size() > blockSize) {  // blockSize = 3000
            ts << TrianglePatch();
          }
          // Convert TRIANGLE_STRIP/FAN to plain TRIANGLES
          GLenum mode = static_cast<GLenum>(*ptr);
          // ... append converted triangles
        }
      }
      features.last().triangles.append(ts);
    } else {
      // SKIP: Not renderable, discard triangle data
      for (uint i = 0; i < p->triprim_count; i++) {
        ptr++; // skip mode
        const auto nvert = read_and_advance<uint32_t>(&ptr);
        ptr += 4 * sizeof(double) + nvert * 2 * sizeof(float);
      }
    }

    // Always parse edge references (for borders/contours)
    // ... edge parsing code
  })
}
```

**Assembly Decision** [osenc.cpp:732–750]:
```cpp
if (w.geom == S57::Geometry::Type::Area) {
  GLsizei offset = vertices.size() * sizeof(GLfloat);
  S57::ElementDataVector triangles;
  QPointF center;

  if (w.triangles.isEmpty()) {
    // NO PRE-TRIANGULATION: compute center from line border
    center = ChartFileReader::computeLineCenter(lines, vertices, indices);
  } else {
    // HAS PRE-TRIANGULATION: use it directly, compute center from triangles
    triangleGeometry(w.triangles, triangles);
    center = computeAreaCenterAndBboxes(triangles, vertices, offset);
  }

  helper.osEncSetGeometry(w.object,
                          new S57::Geometry::Area(lines,
                                                  triangles,  // May be empty!
                                                  center,
                                                  gp->toWGS84(center),
                                                  offset,
                                                  false),
                          bbox);
}
```

### Fallback Triangulation

**File:** `qutenavlib/src/chartfilereader.cpp`

**triangulate() Function** [chartfilereader.cpp:264–293]:
```cpp
void ChartFileReader::triangulate(S57::ElementDataVector& elems,
                                  GL::IndexVector& indices,
                                  const GL::VertexVector& vertices,
                                  const S57::ElementDataVector& edges) {

  if (edges.isEmpty()) return;

  auto tri = new Triangulator(vertices, indices);

  // First edge = outer ring
  int first = edges.first().offset / sizeof(GLuint) + 1;
  int count = edges.first().count - 3;
  tri->addPolygon(first, count);

  // Remaining edges = holes
  for (int i = 1; i < edges.size(); i++) {
    first = edges[i].offset / sizeof(GLuint) + 1;
    count = edges[i].count - 3;
    tri->addHole(first, count);
  }

  auto triangles = tri->triangulate();  // Earcut algorithm
  delete tri;

  S57::ElementData e(GL_TRIANGLES, indices.size() * sizeof(GLuint), triangles.size());
  elems.append(e);
  indices.append(triangles);
}
```

**When Fallback is Triggered:**
- When `w.triangles.isEmpty()` (no pre-triangulated data from SENC)
- Called explicitly from chart readers (e.g., S57Reader, CM93Reader)
- NOT called for OSENC areas (assumes OSENC always has triangles)

### Summary

**QuteNav's Behavior:**

1. **OSENC areas**: ALWAYS have `triprim_count > 0` → pre-triangulated
2. **Non-renderable areas**: Triangles discarded, only borders kept
3. **S57 areas**: No triangulation in file → fallback to earcut
4. **Empty check**: `w.triangles.isEmpty()` determines usage

### Proposed Rust Enum

```rust
enum AreaGeometry {
    /// Pre-triangulated (from OSENC)
    PreTriangulated {
        triangles: Vec<Triangle>,      // Already tessellated
        borders: Vec<EdgeRing>,        // For outline rendering
    },

    /// Edge rings only (from S57 or non-renderable OSENC)
    EdgeRings {
        outer: EdgeRing,
        holes: Vec<EdgeRing>,
    },
}

impl AreaGeometry {
    fn from_senc_area(
        triprim_count: u32,
        edge_refs: Vec<SencEdgeRef>,
        is_renderable: bool,
        triangle_data: Option<&[u8]>,
    ) -> Self {
        if is_renderable && triprim_count > 0 {
            // Parse pre-triangulated data
            let triangles = parse_triangle_primitives(triangle_data.unwrap());
            let borders = resolve_edge_rings(edge_refs);

            Self::PreTriangulated { triangles, borders }
        } else {
            // Use edge rings, will triangulate later if needed
            let rings = resolve_edge_rings(edge_refs);

            Self::EdgeRings {
                outer: rings[0].clone(),
                holes: rings[1..].to_vec(),
            }
        }
    }

    fn get_triangles(&self, triangulator: &mut Triangulator) -> Vec<Triangle> {
        match self {
            Self::PreTriangulated { triangles, .. } => triangles.clone(),
            Self::EdgeRings { outer, holes } => {
                // Fallback triangulation
                triangulator.add_polygon(outer);
                for hole in holes {
                    triangulator.add_hole(hole);
                }
                triangulator.triangulate()
            }
        }
    }
}
```

---

## 4. CONNECTOR SEGMENTS (CE/EC/CC) – AREAS VS LINES

### Findings: NOT USED IN QUTENAV

**Search Results:**
- **No references** to `CE`, `EC`, `CC` segment types found
- **No segment type enumeration** in OSENC code
- **No segment classification logic**

**What QuteNav Uses Instead:**

### Edge References Only

**For LINES** [osenc.cpp:446–468]:
```cpp
{SencRecordType::FEATURE_GEOMETRY_RECORD_LINE, new Handler([&features, &hasReversed] (const Buffer& b) {
    auto p = reinterpret_cast<const OSENC_LineGeometry_Record_Payload*>(b.constData());
    const uint stride = hasReversed ? 4 : 3;
    QVector<int> es(p->edgeVector_count * stride);
    memcpy(es.data(), &p->edge_data, p->edgeVector_count * stride * sizeof(int));

    RawEdgeRefVector refs;
    for (uint i = 0; i < p->edgeVector_count; i++) {
      RawEdgeRef ref;
      ref.begin = es[stride * i];
      ref.index = std::abs(es[stride * i + 1]);
      ref.end = es[stride * i + 2];
      ref.reversed = /* ... version-dependent ... */;
      refs.append(ref);
    }
    features.last().edgeRefs.append(refs);
    // NO segment type field
  })
}
```

**For AREAS** [osenc.cpp:530–547]:
```cpp
// Identical edge reference structure
const uint stride = hasReversed ? 4 : 3;
QVector<int> es(p->edgeVector_count * stride);
memcpy(es.data(), ptr, p->edgeVector_count * stride * sizeof(int));

RawEdgeRefVector refs;
for (uint i = 0; i < p->edgeVector_count; i++) {
  RawEdgeRef ref;
  ref.begin = es[stride * i];
  ref.index = std::abs(es[stride * i + 1]);
  ref.end = es[stride * i + 2];
  ref.reversed = /* ... */;
  refs.append(ref);
}
// NO segment type, NO curve handling
```

### Edge Table Structure

**VECTOR_EDGE_NODE_TABLE_RECORD** [osenc.cpp:565–587]:
```cpp
{SencRecordType::VECTOR_EDGE_NODE_TABLE_RECORD, new Handler([&vertices, &edges] (const Buffer& b) {
    const char* ptr = b.constData();
    const auto cnt = read_and_advance<int>(&ptr);

    for (int i = 0; i < cnt; i++) {
      const auto index = read_and_advance<int>(&ptr);
      const auto pcnt = read_and_advance<quint32>(&ptr);

      // ONLY: point count + vertex array
      QVector<float> points(2 * pcnt);
      memcpy(points.data(), ptr, 2 * pcnt * sizeof(float));
      ptr += 2 * pcnt * sizeof(float);

      RawEdge edge;
      edge.first = vertices.size() / 2;
      edge.count = pcnt;
      edges[std::abs(index)] = edge;
      vertices.append(points);
    }
    // NO segment type metadata
  })
}
```

### Interpretation

**QuteNav's OSENC format assumes:**
- All edges are **simple polylines** (straight segments between vertices)
- No arc/curve encoding
- No connector type classification

**This means:**
- **CE (Curve Edge)**: Not used
- **EC (Edge Continuation)**: Not used
- **CC (Curve Continuation)**: Not used
- **Only straight-line edges** with vertex arrays

### NavCore Recommendation

```rust
// QuteNav-compatible: simple edge references
struct EdgeRef {
    begin_node: u32,
    end_node: u32,
    edge_index: u32,
    reversed: bool,
    // NO segment_type field needed
}

// Edge table: just vertex arrays
struct EdgeTableEntry {
    vertices: Vec<[f32; 2]>,  // Straight-line polyline
    // NO curve data
}

// For both lines and areas: identical treatment
fn assemble_geometry(edge_refs: &[EdgeRef], edge_table: &HashMap<u32, EdgeTableEntry>) -> Vec<Vec<[f32; 2]>> {
    edge_refs.iter().map(|ref| {
        let edge = &edge_table[&ref.edge_index];
        let mut vertices = edge.vertices.clone();

        if ref.reversed {
            vertices.reverse();
        }

        vertices
    }).collect()
}
```

**Conclusion for NavCore:**
- **Do NOT implement CE/EC/CC** for QuteNav-compatible OSENC
- **Only implement simple edge references** with reversed flag
- **Assume all edges are polylines** (no curves)
- If CE/EC/CC are needed (for S-57 compatibility), implement in a separate S-57 parser, NOT OSENC

---

## 5. FINAL SUMMARY FOR QUTENAV

### Critical Implementation Details

1. **Edge Direction Encoding**
   - **Version ≤ 200**: Signed edge index (negative = reversed) [osenc.cpp:461]
   - **Version > 200**: Unsigned index + explicit reversed flag [osenc.cpp:459, 541]
   - **Detection**: `hasReversed = version > 200` [osenc.cpp:384]
   - **Reversal**: Swap begin/end nodes when reversed [osenc.cpp:716–720]
   - **Iteration**: Reverse vertex order when reversed [chartfilereader.cpp:302–306]

2. **SENC Version Storage**
   - **Record type**: `HEADER_SENC_VERSION` (type 1) [osenc.h:113]
   - **Data type**: `quint16` (unsigned 16-bit) [osenc.cpp:55, 155, 382]
   - **Expected value**: **201** (or any > 200 for new format)
   - **Critical threshold**: 200 [osenc.cpp:384]
   - **No validation**: Any version accepted, only affects parsing

3. **Pre-Triangulated Areas**
   - **Always present in OSENC**: `triprim_count > 0` [osenc.cpp:482]
   - **Optimization**: `triprim_count < 5` → separate patches; `≥ 5` → batched [osenc.cpp:482, 493]
   - **Discarded if non-renderable**: `!S52::IsAreaRenderable()` [osenc.cpp:480, 518–525]
   - **Fallback check**: `w.triangles.isEmpty()` [osenc.cpp:736]
   - **Triangle primitives**: GL_TRIANGLES, GL_TRIANGLE_STRIP, GL_TRIANGLE_FAN [osenc.cpp:485, 499, 506–512]
   - **Converted to plain triangles**: Strip/Fan → Triangles [osenc.cpp:844–868]

4. **Connector Segments (CE/EC/CC)**
   - **NOT USED in QuteNav OSENC**
   - **No segment type field** in edge records
   - **Only polylines**: Edges are vertex arrays [osenc.cpp:576–584]
   - **Same for lines and areas**: Identical edge reference structure [osenc.cpp:446–468, 530–547]
   - **No curve support**: All edges are straight-line segments

### Gotchas for NavCore

#### Must Replicate Exactly

1. **Version-Dependent Parsing**
   - MUST detect version > 200 to choose parsing format
   - MUST handle both signed-index and explicit-flag formats
   - [osenc.cpp:384, 458–462]

2. **OSENC-Specific Reversal**
   - MUST swap begin/end nodes when `reversed=true`
   - Different from other formats (per code comment) [osenc.cpp:717–718]
   - [osenc.cpp:716–725]

3. **Triangle Conversion**
   - MUST convert TRIANGLE_STRIP and TRIANGLE_FAN to plain TRIANGLES
   - Winding order matters: strip alternates, fan centers [osenc.cpp:850–868]
   - [osenc.cpp:844–868]

4. **Renderable Check**
   - MUST check `S52::IsAreaRenderable()` before using triangles
   - Non-renderable areas: discard triangles, keep borders only
   - [osenc.cpp:480, 518–525]

#### Can Simplify

1. **Triangle Batching**
   - QuteNav batches by 3000 vertices; NavCore can use different size
   - Or skip batching entirely (simpler but more draw calls)
   - [osenc.cpp:493–516]

2. **Fallback Triangulation**
   - QuteNav assumes OSENC always has triangles
   - NavCore can add fallback for corrupt/missing triangle data
   - [osenc.cpp:736–741]

#### CPU vs GPU

**CPU (Rust Parsing):**
- Version detection [osenc.cpp:381–384]
- Edge reference decoding [osenc.cpp:446–468, 530–547]
- Node lookup [osenc.cpp:590–608]
- Edge reversal [osenc.cpp:716–725]
- Triangle primitive conversion [osenc.cpp:844–868]
- Renderable check [osenc.cpp:480]

**GPU (WGPU Rendering):**
- Final vertex buffers (converted triangles)
- Index buffers (line strips)
- No topology knowledge needed

#### Data Structures

**File Parsing:**
- `qutenavlib/src/osenc.{h,cpp}` [osenc.h:30–108, osenc.cpp:359–762]

**Key Types:**
- `RawEdgeRef` [osenc.h:75–80]
- `RawEdge` [osenc.h:54–57]
- `TrianglePatch` [osenc.h:64–71]
- `OSENC_AreaGeometry_Record_Payload` [osenc.h:199–208]
- `OSENC_LineGeometry_Record_Payload` [osenc.h:190–197]

**Assembly:**
- `ChartFileReader::addIndices()` [chartfilereader.cpp:295–309]
- `ChartFileReader::triangulate()` [chartfilereader.cpp:264–293]
- `appendTriangles/Strip/Fan()` [osenc.cpp:844–868]

#### Edge Cases

1. **Zero-length edges**: `edge.count == 0` allowed [osenc.cpp:709–714]
2. **Missing edge indices**: `!edges.contains(ref.index)` handled [osenc.cpp:708]
3. **Empty triangle list**: `w.triangles.isEmpty()` checked [osenc.cpp:736]
4. **No renderable areas**: Triangles discarded [osenc.cpp:518–525]

---

## Appendix: Key Code Locations

| Topic | File | Lines |
|-------|------|-------|
| **Edge Direction** |
| RawEdgeRef struct | osenc.h | 75–80 |
| Version detection | osenc.cpp | 381–384 |
| Old format parsing | osenc.cpp | 458–462 |
| New format parsing | osenc.cpp | 458–462, 540–542 |
| Node swap on reversal | osenc.cpp | 716–725 |
| Vertex iteration | chartfilereader.cpp | 295–309 |
| **SENC Version** |
| Record type enum | osenc.h | 113 |
| Version read (outline) | osenc.cpp | 55–56 |
| Version read (chart) | osenc.cpp | 381–384 |
| Threshold check | osenc.cpp | 384 |
| **Pre-Triangulation** |
| Area record struct | osenc.h | 199–208 |
| Triangle parsing | osenc.cpp | 471–526 |
| Batch optimization | osenc.cpp | 482–516 |
| Renderable check | osenc.cpp | 480, 518–525 |
| isEmpty() decision | osenc.cpp | 736–741 |
| Triangle conversion | osenc.cpp | 844–868 |
| Fallback triangulation | chartfilereader.cpp | 264–293 |
| **Edge References** |
| Line edge parsing | osenc.cpp | 446–468 |
| Area edge parsing | osenc.cpp | 530–547 |
| Edge table parsing | osenc.cpp | 565–587 |
| Connected nodes | osenc.cpp | 590–608 |

---

**End of Document**
