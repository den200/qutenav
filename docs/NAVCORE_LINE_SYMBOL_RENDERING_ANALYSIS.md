# NavCore Line & Symbol Rendering: Deep Analysis

This document provides a detailed analysis of how lines and symbols are rendered in QuteNav, from SENC parsing to pixel output. This serves as a reference for implementing correct rendering in NavCore.

## Table of Contents
1. [Coordinate Systems](#coordinate-systems)
2. [SENC Parsing Pipeline](#senc-parsing-pipeline)
3. [Line Geometry Resolution](#line-geometry-resolution)
4. [Vertex Transformation Pipeline](#vertex-transformation-pipeline)
5. [Line Rendering](#line-rendering)
6. [Symbol Rendering](#symbol-rendering)
7. [Critical Implementation Details for NavCore](#critical-implementation-details-for-navcore)
8. [Common Bugs and Solutions](#common-bugs-and-solutions)

---

## 1. Coordinate Systems

### Simple Mercator (SM) Coordinates

QuteNav uses Simple Mercator (SM) projection for all chart data:

```
z0 = 6378137 * 0.9996 ≈ 6375584.9 meters

x = (lon - ref_lon) * π/180 * z0
y = 0.5 * ln((1 + sin(lat))/(1 - sin(lat))) * z0 - y_ref
```

Where `y_ref` is the same formula applied to `ref_lat`.

**Reference Point**: Calculated from CELL_EXTENT_RECORD as center:
```cpp
ref = WGS84Point::fromLL(
    0.5 * (sw.lng() + ne.lng()),
    0.5 * (sw.lat() + ne.lat())
);
```

### Coordinate System Summary

| Data Type | Stored As | Units | Relative To |
|-----------|-----------|-------|-------------|
| Point Geometry (OSENC) | lat/lon | degrees | WGS84 absolute |
| Line/Area Vertices | x, y | meters | Chart reference point (SM) |
| Edge Node Table | x, y | meters | Chart reference point (SM) |
| Connected Nodes | x, y | meters | Chart reference point (SM) |
| Triangle Vertices | x, y | meters | Chart reference point (SM) |
| Symbol Pivot | x, y | meters | Chart reference point (SM) |
| Symbol Offset | x, y | mm | Display units |

---

## 2. SENC Parsing Pipeline

### Record Types

```rust
enum SencRecordType {
    HEADER_SENC_VERSION = 1,
    HEADER_CELL_NAME = 2,
    HEADER_CELL_PUBLISHDATE = 3,
    HEADER_CELL_EDITION = 4,
    HEADER_CELL_UPDATEDATE = 5,
    HEADER_CELL_UPDATE = 6,
    HEADER_CELL_NATIVESCALE = 7,
    HEADER_CELL_SENCCREATEDATE = 8,
    HEADER_CELL_SOUNDINGDATUM = 9,
    FEATURE_ID_RECORD = 64,
    FEATURE_ATTRIBUTE_RECORD = 65,
    FEATURE_GEOMETRY_RECORD_POINT = 80,
    FEATURE_GEOMETRY_RECORD_LINE = 81,
    FEATURE_GEOMETRY_RECORD_AREA = 82,
    FEATURE_GEOMETRY_RECORD_MULTIPOINT = 83,
    VECTOR_EDGE_NODE_TABLE_RECORD = 96,
    VECTOR_CONNECTED_NODE_TABLE_RECORD = 97,
    CELL_COVR_RECORD = 98,
    CELL_NOCOVR_RECORD = 99,
    CELL_EXTENT_RECORD = 100,
}
```

### Point Geometry Record (Type 80)

**CRITICAL**: Point geometry stores lat/lon in WGS84 degrees, NOT SM coordinates!

```cpp
struct OSENC_PointGeometry_Record_Payload {
    double lat;  // WGS84 latitude in degrees
    double lon;  // WGS84 longitude in degrees
};
```

**QuteNav Conversion** (osenc.cpp:434-443):
```cpp
auto p = reinterpret_cast<const OSENC_PointGeometry_Record_Payload*>(b.constData());
const auto c0 = WGS84Point::fromLL(p->lon, p->lat);  // WGS84 point
const auto p0 = gp->fromWGS84(c0);  // Convert to SM meters
// p0 is now in SM meters relative to chart reference
```

### Line Geometry Record (Type 81)

```cpp
struct OSENC_LineGeometry_Record_Payload {
    double extent_s_lat;
    double extent_n_lat;
    double extent_w_lon;
    double extent_e_lon;
    uint32_t edgeVector_count;
    char edge_data;  // Array of edge references
};
```

Edge reference format (stride = 3 or 4):
```
[begin_node_id, edge_index, end_node_id, (reversed?)]
```

- SENC version ≤ 200: stride = 3, reversed = (edge_index < 0)
- SENC version > 200: stride = 4, explicit reversed flag

**Edge Index = 0**: Special case meaning NO intermediate vertices. The edge is just a straight line from begin to end node.

### Edge Node Table Record (Type 96)

Contains the intermediate vertices for edges (already in SM meters):

```cpp
// For each edge:
int32_t edge_index;
uint32_t point_count;
float points[point_count * 2];  // x, y pairs in SM meters
```

### Connected Node Table Record (Type 97)

Contains the start/end points that connect edges (already in SM meters):

```cpp
// For each node:
uint32_t node_index;
float x;  // SM meters
float y;  // SM meters
```

---

## 3. Line Geometry Resolution

### Edge Resolution Algorithm

```cpp
// For each edge reference in feature:
struct Edge {
    uint32_t begin;     // Index into connected nodes (start vertex)
    uint32_t end;       // Index into connected nodes (end vertex)
    uint32_t first;     // Index into vertices (first intermediate)
    uint32_t count;     // Number of intermediate vertices
    bool reversed;      // Whether to reverse order
};

// Edge index 0 means no intermediate vertices
if (ref.index == 0) {
    edge.first = 0;
    edge.count = 0;  // Just begin → end, no intermediates
} else {
    edge.first = edges[ref.index].first;
    edge.count = edges[ref.index].count;
}

// If reversed, swap begin and end
if (ref.reversed) {
    edge.begin = connected[ref.end];
    edge.end = connected[ref.begin];
} else {
    edge.begin = connected[ref.begin];
    edge.end = connected[ref.end];
}
```

### Line Element Creation (Critical!)

```cpp
// Add indices for one edge to the index buffer
int addIndices(const Edge& e, IndexVector& indices) {
    if (!e.reversed) {
        indices << e.begin;
        for (int i = 0; i < e.count; i++) {
            indices << e.first + i;
        }
    } else {
        indices << e.end;  // Note: using END because begin/end already swapped
        for (int i = 0; i < e.count; i++) {
            indices << e.first + e.count - 1 - i;  // Reverse order
        }
    }
    return e.count + 1;
}
```

### Adjacency Vertices

For thick line rendering, QuteNav uses `GL_LINE_STRIP_ADJACENCY`:
- Each line segment needs prev and next vertices for miter calculation
- Virtual adjacency vertices are created by extrapolation:

```cpp
// For endpoints, create virtual adjacent vertex by extrapolation
int addAdjacent(int endpoint, int neighbor, VertexVector& vertices) {
    float x1 = vertices[2 * endpoint];
    float y1 = vertices[2 * endpoint + 1];
    float x2 = vertices[2 * neighbor];
    float y2 = vertices[2 * neighbor + 1];

    // Extrapolate: adjacent = 2 * endpoint - neighbor
    vertices << (2 * x1 - x2) << (2 * y1 - y2);
    return (vertices.size() - 1) / 2;
}
```

### Line Element Format

Final format for each line element:
```
[adj_prev, vertex_0, vertex_1, ..., vertex_n, adj_next]
```

Where:
- For closed polygons: adj_prev = last vertex, adj_next = second vertex
- For open lines: adj_prev/adj_next are extrapolated virtual vertices

---

## 4. Vertex Transformation Pipeline

### Matrix Chain

```
SM coords → Model Matrix → View Space → Projection Matrix → NDC → Screen
```

### Model Matrix

Transforms chart coordinates (relative to chart reference) to camera coordinates (relative to camera reference):

```cpp
void updateModelTransform(const Camera* cam) {
    m_modelMatrix.setToIdentity();
    // Get chart reference point in camera's coordinate system
    QPointF p = cam->geoprojection()->fromWGS84(chartGeoProjection()->reference());
    m_modelMatrix.translate(p.x(), p.y());
}
```

### Projection Matrix

Orthographic projection:
```cpp
m_projection.ortho(-x1, x1, -y1, y1, -1., 1.);
```

Where x1/y1 are calculated from scale and display size.

### View Matrix

2D rotation for north angle:
```cpp
m_view.setRow(IX, QVector2D(cos(angle), -sin(angle)));
m_view.setRow(IY, QVector2D(sin(angle), cos(angle)));
```

---

## 5. Line Rendering

### Thick Line Technique

QuteNav uses a geometry-shader-free approach for thick lines with miter joins:

```glsl
// Vertex shader processes 3 adjacent points at a time
const vec4 p0 = m_model * vec4(vertices[i0], depth, 1.);
const vec4 p1 = m_model * vec4(vertices[i1], depth, 1.);
const vec4 p2 = m_model * vec4(vertices[i2], depth, 1.);

// Direction vectors
vec2 v0 = normalize(p1.xy - p0.xy);
vec2 v1 = normalize(p2.xy - p1.xy);

// Normal vectors (perpendicular)
vec2 n0 = vec2(-v0.y, v0.x);
vec2 n1 = vec2(-v1.y, v1.x);

// Miter calculation
vec2 miter = normalize(n0 + n1);
float len_m = HW / dot(miter, n0);  // Half-width / cosine of angle

// Output position (top or bottom of thick line)
if (gl_VertexID % 2 == 0) {
    gl_Position = m_p * vec4(p1.xy + len_m * miter, p1.z, 1.);
} else {
    gl_Position = m_p * vec4(p1.xy - len_m * miter, p1.z, 1.);
}
```

### Draw Call Structure

```cpp
// Bind vertex buffer as SSBO (binding 0)
// Bind index buffer as SSBO (binding 1)

// Set uniforms
shader.setUniform("vertexOffset", vertexOffset / 2 / sizeof(float));
shader.setUniform("indexOffset", elementOffset / sizeof(uint));
shader.setUniform("lineWidth", lineWidth);  // in mm
shader.setUniform("windowScale", mmPerMeter);

// Draw as triangle strip
// 2 vertices per joint, (count - 2) joints per line segment
glDrawArrays(GL_TRIANGLE_STRIP, 0, 2 * (count - 2));
```

---

## 6. Symbol Rendering

### Symbol Pivot Points

Symbol pivots are in SM coordinates (chart units):

```cpp
// From s52functions.cpp
QPointF loc = obj->geometry()->center();  // Already in SM meters

// For point geometry with single point:
// loc = converted SM coordinate from original lat/lon

// Create symbol paint data
new RasterSymbolPaintData(symbolIndex, symbolOffset, loc, element);
```

### Symbol Vertex Shader

```glsl
layout (location = 0) in vec2 vertex;   // Symbol quad vertex (mm)
layout (location = 1) in vec2 texin;    // Texture coordinates
layout (location = 2) in vec2 pivot;    // Instance pivot (SM meters)

uniform vec2 offset;        // Symbol offset from pivot (mm)
uniform float windowScale;  // mm per meter ratio
uniform mat4 m_model;       // Chart → Camera transform
uniform mat4 m_p;           // Projection

void main() {
    tex = texin;

    // Convert display units (mm) to chart units (m)
    float a = 1.0 / windowScale;

    // Position: offset + vertex in chart units, plus pivot
    vec2 v = a * (vertex + offset) + pivot;

    gl_Position = m_p * m_model * vec4(v, depth, 1.);
}
```

### Window Scale Calculation

```cpp
// scaleFactor = mm_per_meter ratio
qreal S57Chart::scaleFactor(const QRectF& va, quint32 scale) const {
    WGS84Point sw = proj->toWGS84(va.topLeft());
    WGS84Point nw = proj->toWGS84(va.bottomLeft());

    // Meters of chart height / display mm / chart scale
    return 1000.0 * (nw - sw).meters() / scale / va.height();
}
```

---

## 7. Critical Implementation Details for NavCore

### Bug #1: Point Geometry Coordinates

**Problem**: OSENC point geometry stores WGS84 lat/lon, but NavCore might be using them directly as SM coordinates.

**Solution**:
```rust
fn parse_point_geometry(payload: &PointGeometryPayload, proj: &SimpleMercator) -> Point {
    // lat/lon are in WGS84 degrees!
    let wgs84 = WGS84Point::from_ll(payload.lon, payload.lat);

    // Must convert to SM coordinates
    let sm = proj.from_wgs84(&wgs84);

    Point {
        center: sm,
        center_ll: wgs84,
    }
}
```

### Bug #2: Edge Index Zero

**Problem**: Edge index 0 means no intermediate vertices, not "edge doesn't exist".

**Solution**:
```rust
fn resolve_edge(ref: &EdgeRef, edges: &HashMap<u32, RawEdge>, connected: &HashMap<u32, u32>) -> Edge {
    let (first, count) = if ref.index == 0 {
        // No intermediate vertices - just begin to end
        (0, 0)
    } else {
        let edge = &edges[&ref.index];
        (edge.first, edge.count)
    };

    Edge {
        begin: if ref.reversed { connected[&ref.end] } else { connected[&ref.begin] },
        end: if ref.reversed { connected[&ref.begin] } else { connected[&ref.end] },
        first,
        count,
        reversed: ref.reversed,
    }
}
```

### Bug #3: Reversed Edge Vertex Order

**Problem**: When edge is reversed, must both swap begin/end AND reverse intermediate vertex order.

**Solution**:
```rust
fn add_edge_indices(edge: &Edge, indices: &mut Vec<u32>) {
    // Always start with begin node
    indices.push(edge.begin);

    if edge.count > 0 {
        if edge.reversed {
            // Iterate backwards through intermediate vertices
            for i in (0..edge.count).rev() {
                indices.push(edge.first + i);
            }
        } else {
            // Iterate forwards
            for i in 0..edge.count {
                indices.push(edge.first + i);
            }
        }
    }
    // Note: end node will be added as begin of next edge, or explicitly for last edge
}
```

### Bug #4: Missing Adjacency Vertices

**Problem**: Thick line rendering requires adjacency vertices for miter calculation.

**Solution**: For each line element:
1. First index is adjacency prev (virtual extrapolated for open lines)
2. Last index is adjacency next (virtual extrapolated for open lines)
3. Actual line is indices[1..n-1]

### Bug #5: Model Matrix Offset

**Problem**: Chart and camera have different reference points.

**Solution**:
```rust
fn update_model_matrix(chart_ref: &WGS84Point, camera_proj: &SimpleMercator) -> Mat4 {
    // Get chart reference in camera's coordinate system
    let offset = camera_proj.from_wgs84(chart_ref);

    Mat4::from_translation(vec3(offset.x, offset.y, 0.0))
}
```

---

## 8. Common Bugs and Solutions

### Diagonal Lines Across Screen

**Cause**: Reading edges with index 0 as if they had vertices at index 0.

**Solution**: Check for index 0 specially - it means no intermediate vertices.

### Lines Not Connecting

**Cause**: Not swapping begin/end when edge is reversed.

**Solution**: If reversed, use `connected[ref.end]` as begin and vice versa.

### Symbols at Wrong Position

**Cause**: Using lat/lon directly as x/y instead of converting to SM.

**Solution**: Always convert point geometry lat/lon through `proj.from_wgs84()`.

### Line Ends Not Meeting

**Cause**: Edges being chained incorrectly due to wrong vertex order.

**Solution**: Ensure reversed edges have their intermediate vertices iterated in reverse order.

### Thick Lines Have Gaps

**Cause**: Missing or incorrect adjacency vertices.

**Solution**: Add virtual extrapolated vertices at line endpoints.

---

## Quick Reference: WGSL Shader Equivalents

### Line Vertex Shader
```wgsl
@vertex
fn vs_main(@builtin(vertex_index) vertex_id: u32) -> VertexOutput {
    let half_width = 0.5 * line_width / window_scale;

    let i0 = vertex_offset + indices[index_offset + vertex_id / 2];
    let i1 = vertex_offset + indices[index_offset + vertex_id / 2 + 1];
    let i2 = vertex_offset + indices[index_offset + vertex_id / 2 + 2];

    let p0 = m_model * vec4f(vertices[i0], depth, 1.0);
    let p1 = m_model * vec4f(vertices[i1], depth, 1.0);
    let p2 = m_model * vec4f(vertices[i2], depth, 1.0);

    let v0 = normalize(p1.xy - p0.xy);
    let v1 = normalize(p2.xy - p1.xy);

    let n0 = vec2f(-v0.y, v0.x);
    let n1 = vec2f(-v1.y, v1.x);

    let miter = normalize(n0 + n1);
    let len_m = half_width / dot(miter, n0);

    var pos: vec2f;
    if (vertex_id % 2 == 0) {
        pos = p1.xy + len_m * miter;
    } else {
        pos = p1.xy - len_m * miter;
    }

    var out: VertexOutput;
    out.position = m_p * vec4f(pos, p1.z, 1.0);
    return out;
}
```

### Symbol Vertex Shader
```wgsl
struct SymbolVertex {
    @location(0) vertex: vec2f,   // mm
    @location(1) texcoord: vec2f,
    @location(2) pivot: vec2f,    // SM meters
}

@vertex
fn vs_main(in: SymbolVertex) -> VertexOutput {
    let a = 1.0 / window_scale;
    let v = a * (in.vertex + offset) + in.pivot;

    var out: VertexOutput;
    out.position = m_p * m_model * vec4f(v, depth, 1.0);
    out.texcoord = in.texcoord;
    return out;
}
```
