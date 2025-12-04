# NavCore M2 Phase 2: Symbol Rendering Implementation Plan

## Executive Summary

Based on analysis of QuteNav's symbol system (`rastersymbolmanager.cpp`, `s52functions.cpp`, `chartsymbols.xml`), this document proposes a simplified atlas-based instanced rendering system adapted for wgpu.

**Key insights from QuteNav:**
1. Symbol coordinates are stored in display units (mm), then scaled by `windowScale` at render time
2. Pivots are in world/chart units - the symbol is placed at the pivot, then offset by the symbol's internal offset
3. Instanced rendering with `glDrawElementsInstanced` per symbol type
4. Pre-baked quad geometry per symbol with embedded UV coords

---

## 1. Data Model

### 1.1 Core Rust Structs

```rust
// src/render/symbols.rs

/// Identifies a symbol in the atlas
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[repr(u16)]
pub enum SymbolId {
    // Lateral Buoys (BOYLAT)
    BuoyLatPort = 0,        // BOYLAT14 - red conical
    BuoyLatStarboard = 1,   // BOYLAT13 - green conical

    // Lateral Beacons (BCNLAT)
    BeaconLatPort = 2,      // BCNLAT21 - red
    BeaconLatStarboard = 3, // BCNLAT15 - green

    // Underwater Rocks (UWTROC)
    RockAwash = 4,          // UWTROC03 - rock awash (watlev=3)
    RockSubmerged = 5,      // UWTROC04 - submerged rock (watlev=4,5)

    // Fallback
    Unknown = 255,
}

/// Metadata for a single symbol in the atlas
#[derive(Debug, Clone, Copy)]
pub struct SymbolMeta {
    /// UV rect in atlas: (u_min, v_min, u_max, v_max) normalized [0..1]
    pub uv_rect: [f32; 4],

    /// Pivot offset from symbol top-left, in normalized symbol coords [0..1]
    /// e.g., [0.5, 0.9] = center-bottom
    pub pivot: [f32; 2],

    /// Symbol size in pixels (for reference, actual size is screen-space)
    pub size_px: [u32; 2],
}

/// Per-instance data sent to GPU
#[derive(Debug, Clone, Copy, bytemuck::Pod, bytemuck::Zeroable)]
#[repr(C)]
pub struct SymbolInstance {
    /// World position in SM (Simple Mercator) meters, relative to chart ref
    pub position: [f32; 2],

    /// Symbol ID (index into uniform array of SymbolMeta)
    pub symbol_id: u32,

    /// Rotation in radians (0 = north up) - for future use
    pub rotation: f32,
}

/// Atlas metadata loaded from JSON
#[derive(Debug)]
pub struct SymbolAtlas {
    pub texture: wgpu::Texture,
    pub view: wgpu::TextureView,
    pub sampler: wgpu::Sampler,
    pub bind_group: wgpu::BindGroup,
    pub metadata: Vec<SymbolMeta>,  // Indexed by SymbolId
}
```

### 1.2 S-57 Object to SymbolId Mapping

Based on QuteNav's `s52functions.cpp:1462-1592` (OBSTRN04) and chartsymbols.xml lookup tables:

```rust
// src/senc/symbols.rs

/// Maps S-57 object class + attributes to SymbolId
pub fn map_feature_to_symbol(feature: &SencFeature) -> Option<SymbolId> {
    match feature.object_class {
        // BOYLAT (Object class code: varies by chart)
        code if is_boylat(code) => {
            // COLOUR attribute: 3=red, 4=green
            // CATLAM attribute: 1=port, 2=starboard (IALA-A)
            let colour = feature.get_attr_int("COLOUR").unwrap_or(0);
            let catlam = feature.get_attr_int("CATLAM").unwrap_or(0);

            // Priority: check COLOUR first, then CATLAM
            if colour == 3 || colour == 1 || catlam == 1 {  // Red or port
                Some(SymbolId::BuoyLatPort)
            } else if colour == 4 || colour == 2 || catlam == 2 {  // Green or starboard
                Some(SymbolId::BuoyLatStarboard)
            } else {
                Some(SymbolId::BuoyLatPort)  // Default: red
            }
        }

        // BCNLAT
        code if is_bcnlat(code) => {
            let colour = feature.get_attr_int("COLOUR").unwrap_or(0);
            if colour == 4 {  // Green
                Some(SymbolId::BeaconLatStarboard)
            } else {
                Some(SymbolId::BeaconLatPort)  // Default: red
            }
        }

        // UWTROC
        code if is_uwtroc(code) => {
            // WATLEV: 2=dry, 3=awash, 4/5=submerged
            let watlev = feature.get_attr_int("WATLEV").unwrap_or(0);
            if watlev == 3 {
                Some(SymbolId::RockAwash)
            } else {
                Some(SymbolId::RockSubmerged)  // Default for 4, 5, or unknown
            }
        }

        _ => None,  // Not a supported symbol type
    }
}

/// Extract symbol position from feature geometry
pub fn get_symbol_position(feature: &SencFeature) -> Option<[f32; 2]> {
    match &feature.geometry {
        Geometry::Point { x, y } => Some([*x, *y]),

        Geometry::Line { vertices } if !vertices.is_empty() => {
            // Use midpoint for line features
            let mid = vertices.len() / 2;
            Some([vertices[mid * 2], vertices[mid * 2 + 1]])
        }

        Geometry::Area { centroid, .. } => {
            // Use pre-computed centroid (or compute from triangles)
            centroid.map(|c| [c.0, c.1])
        }

        _ => None,
    }
}
```

### 1.3 Symbol Mapping Reference Table

| S-57 Object | Attribute Check | SymbolId | S-52 Symbol Name | Notes |
|-------------|-----------------|----------|------------------|-------|
| BOYLAT | COLOUR=3 or CATLAM=1 | `BuoyLatPort` | BOYLAT14 | Red conical buoy |
| BOYLAT | COLOUR=4 or CATLAM=2 | `BuoyLatStarboard` | BOYLAT13 | Green conical buoy |
| BCNLAT | COLOUR=3 | `BeaconLatPort` | BCNLAT21 | Red beacon |
| BCNLAT | COLOUR=4 | `BeaconLatStarboard` | BCNLAT15 | Green beacon |
| UWTROC | WATLEV=3 | `RockAwash` | UWTROC03 | Rock awash at chart datum |
| UWTROC | WATLEV=4,5 or default | `RockSubmerged` | UWTROC04 | Submerged rock with depth |

---

## 2. Atlas Format and Loading

### 2.1 JSON Schema for `assets/symbols/atlas.json`

```json
{
  "version": 1,
  "atlas_size": [1024, 1024],
  "cell_size": [64, 64],
  "symbols": {
    "BuoyLatPort": {
      "cell": [0, 0],
      "size": [14, 14],
      "pivot": [0.64, 0.64],
      "description": "BOYLAT14 - Red lateral buoy"
    },
    "BuoyLatStarboard": {
      "cell": [1, 0],
      "size": [14, 14],
      "pivot": [0.64, 0.64],
      "description": "BOYLAT13 - Green lateral buoy"
    },
    "BeaconLatPort": {
      "cell": [2, 0],
      "size": [11, 15],
      "pivot": [0.45, 0.53],
      "description": "BCNLAT21 - Red lateral beacon"
    },
    "BeaconLatStarboard": {
      "cell": [3, 0],
      "size": [11, 15],
      "pivot": [0.45, 0.53],
      "description": "BCNLAT15 - Green lateral beacon"
    },
    "RockAwash": {
      "cell": [4, 0],
      "size": [12, 12],
      "pivot": [0.5, 0.5],
      "description": "UWTROC03 - Rock awash"
    },
    "RockSubmerged": {
      "cell": [5, 0],
      "size": [12, 12],
      "pivot": [0.5, 0.5],
      "description": "UWTROC04 - Submerged rock"
    }
  }
}
```

**Field meanings:**
- `cell`: Grid position [col, row] in 16x16 atlas grid
- `size`: Actual symbol size in pixels within the 64x64 cell
- `pivot`: Normalized offset within symbol [0..1], where the world position anchors

### 2.2 Atlas Loading Implementation

```rust
// src/render/symbols.rs

impl SymbolAtlas {
    pub fn load(
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        png_path: &Path,
        json_path: &Path,
    ) -> Result<Self, SymbolError> {
        // 1. Load PNG texture
        let img = image::open(png_path)?.to_rgba8();
        let (width, height) = img.dimensions();

        let texture_size = wgpu::Extent3d {
            width,
            height,
            depth_or_array_layers: 1,
        };

        let texture = device.create_texture(&wgpu::TextureDescriptor {
            label: Some("symbol_atlas"),
            size: texture_size,
            mip_level_count: 1,
            sample_count: 1,
            dimension: wgpu::TextureDimension::D2,
            format: wgpu::TextureFormat::Rgba8UnormSrgb,
            usage: wgpu::TextureUsages::TEXTURE_BINDING | wgpu::TextureUsages::COPY_DST,
            view_formats: &[],
        });

        queue.write_texture(
            wgpu::TexelCopyTextureInfo {
                texture: &texture,
                mip_level: 0,
                origin: wgpu::Origin3d::ZERO,
                aspect: wgpu::TextureAspect::All,
            },
            &img,
            wgpu::TexelCopyBufferLayout {
                offset: 0,
                bytes_per_row: Some(4 * width),
                rows_per_image: Some(height),
            },
            texture_size,
        );

        let view = texture.create_view(&wgpu::TextureViewDescriptor::default());

        let sampler = device.create_sampler(&wgpu::SamplerDescriptor {
            label: Some("symbol_sampler"),
            address_mode_u: wgpu::AddressMode::ClampToEdge,
            address_mode_v: wgpu::AddressMode::ClampToEdge,
            mag_filter: wgpu::FilterMode::Linear,
            min_filter: wgpu::FilterMode::Linear,
            ..Default::default()
        });

        // 2. Load JSON metadata
        let json_str = std::fs::read_to_string(json_path)?;
        let json: AtlasJson = serde_json::from_str(&json_str)?;

        let atlas_w = json.atlas_size[0] as f32;
        let atlas_h = json.atlas_size[1] as f32;
        let cell_w = json.cell_size[0] as f32;
        let cell_h = json.cell_size[1] as f32;

        // 3. Build metadata array indexed by SymbolId
        let mut metadata = vec![SymbolMeta::default(); 256];

        for (name, sym) in &json.symbols {
            let id: SymbolId = name.parse()?;
            let col = sym.cell[0] as f32;
            let row = sym.cell[1] as f32;

            // UV coordinates in atlas (normalized)
            let u_min = (col * cell_w) / atlas_w;
            let v_min = (row * cell_h) / atlas_h;
            let u_max = u_min + (sym.size[0] as f32) / atlas_w;
            let v_max = v_min + (sym.size[1] as f32) / atlas_h;

            metadata[id as usize] = SymbolMeta {
                uv_rect: [u_min, v_min, u_max, v_max],
                pivot: sym.pivot,
                size_px: sym.size,
            };
        }

        // 4. Create bind group (layout created in pipeline)
        // ...

        Ok(Self {
            texture,
            view,
            sampler,
            bind_group,
            metadata,
        })
    }
}
```

---

## 3. Instance Buffer and WGSL Shader

### 3.1 Instance Data Layout

```rust
// GPU buffer layout for symbol instances
#[repr(C)]
#[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
pub struct SymbolInstance {
    pub position: [f32; 2],    // World position (SM meters)
    pub symbol_id: u32,        // Index into metadata uniform array
    pub rotation: f32,         // Rotation in radians (future)
}

impl SymbolInstance {
    const ATTRIBS: [wgpu::VertexAttribute; 3] = wgpu::vertex_attr_array![
        0 => Float32x2,  // position
        1 => Uint32,     // symbol_id
        2 => Float32,    // rotation
    ];

    pub fn desc() -> wgpu::VertexBufferLayout<'static> {
        wgpu::VertexBufferLayout {
            array_stride: std::mem::size_of::<Self>() as wgpu::BufferAddress,
            step_mode: wgpu::VertexStepMode::Instance,
            attributes: &Self::ATTRIBS,
        }
    }
}
```

### 3.2 Uniform Buffer for Symbol Metadata

```rust
// Packed symbol metadata for GPU (16 bytes per symbol)
#[repr(C)]
#[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
pub struct SymbolMetaGpu {
    pub uv_rect: [f32; 4],  // u_min, v_min, u_max, v_max
}

// Camera/projection uniforms
#[repr(C)]
#[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
pub struct SymbolUniforms {
    pub view_proj: [[f32; 4]; 4],   // 4x4 view-projection matrix
    pub symbol_scale: f32,           // Pixels per meter (for screen-space sizing)
    pub _padding: [f32; 3],
}
```

### 3.3 WGSL Shader

```wgsl
// shaders/symbols.wgsl

// --- Uniforms ---

struct CameraUniforms {
    view_proj: mat4x4<f32>,
    symbol_scale: f32,  // screen pixels per world unit at current zoom
}

struct SymbolMeta {
    uv_rect: vec4<f32>,  // (u_min, v_min, u_max, v_max)
}

@group(0) @binding(0) var<uniform> camera: CameraUniforms;
@group(0) @binding(1) var<storage, read> symbol_meta: array<SymbolMeta>;

@group(1) @binding(0) var atlas_texture: texture_2d<f32>;
@group(1) @binding(1) var atlas_sampler: sampler;

// --- Vertex Shader ---

struct VertexInput {
    @builtin(vertex_index) vertex_index: u32,
    // Per-instance data
    @location(0) position: vec2<f32>,    // World position
    @location(1) symbol_id: u32,         // Index into symbol_meta
    @location(2) rotation: f32,          // Rotation (radians)
}

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) tex_coord: vec2<f32>,
}

// Symbol size in world units (meters) - fixed screen size
const SYMBOL_SIZE_PIXELS: f32 = 24.0;

@vertex
fn vs_main(in: VertexInput) -> VertexOutput {
    var out: VertexOutput;

    // Get symbol metadata
    let meta = symbol_meta[in.symbol_id];
    let uv_min = meta.uv_rect.xy;
    let uv_max = meta.uv_rect.zw;

    // Generate quad corners from vertex_index (0-5 for two triangles)
    // Triangle 1: 0,1,2  Triangle 2: 2,1,3
    // Vertices: 0=TL, 1=TR, 2=BL, 3=BR
    var corner: vec2<f32>;
    var uv: vec2<f32>;

    switch (in.vertex_index) {
        case 0u: { corner = vec2(-0.5, 0.5);  uv = vec2(uv_min.x, uv_min.y); }  // TL
        case 1u: { corner = vec2(0.5, 0.5);   uv = vec2(uv_max.x, uv_min.y); }  // TR
        case 2u: { corner = vec2(-0.5, -0.5); uv = vec2(uv_min.x, uv_max.y); }  // BL
        case 3u: { corner = vec2(-0.5, -0.5); uv = vec2(uv_min.x, uv_max.y); }  // BL
        case 4u: { corner = vec2(0.5, 0.5);   uv = vec2(uv_max.x, uv_min.y); }  // TR
        case 5u: { corner = vec2(0.5, -0.5);  uv = vec2(uv_max.x, uv_max.y); }  // BR
        default: { corner = vec2(0.0); uv = vec2(0.0); }
    }

    // Apply rotation (optional, for oriented symbols)
    let c = cos(in.rotation);
    let s = sin(in.rotation);
    let rotated = vec2(
        corner.x * c - corner.y * s,
        corner.x * s + corner.y * c
    );

    // Convert symbol to world units (fixed screen size)
    let world_size = SYMBOL_SIZE_PIXELS / camera.symbol_scale;
    let world_offset = rotated * world_size;

    // World position with symbol offset
    let world_pos = in.position + world_offset;

    // Apply view-projection
    out.clip_position = camera.view_proj * vec4(world_pos, 0.0, 1.0);
    out.tex_coord = uv;

    return out;
}

// --- Fragment Shader ---

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    let color = textureSample(atlas_texture, atlas_sampler, in.tex_coord);

    // Discard fully transparent pixels (alpha test)
    if (color.a < 0.1) {
        discard;
    }

    return color;
}
```

### 3.4 Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Vertex generation | `vertex_index` in shader | No vertex buffer needed; 6 vertices per quad generated procedurally |
| Symbol size | Screen-space fixed | Symbols stay constant size regardless of zoom (standard for charts) |
| UV lookup | Storage buffer | One indirection per instance, allows any number of symbols |
| Coordinate system | World space in shader | View-projection applied in shader; instances store world coords |
| Alpha handling | Discard in fragment | Avoids blending complexity for opaque symbols |

---

## 4. Symbol Pipeline Design

### 4.1 Render Pipeline Creation

```rust
// src/render/symbols.rs

impl SymbolRenderer {
    pub fn new(
        device: &wgpu::Device,
        config: &wgpu::SurfaceConfiguration,
        atlas: SymbolAtlas,
    ) -> Self {
        // Create shader module
        let shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
            label: Some("symbol_shader"),
            source: wgpu::ShaderSource::Wgsl(include_str!("../shaders/symbols.wgsl").into()),
        });

        // Bind group layout 0: Camera + Symbol metadata
        let camera_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
            label: Some("symbol_camera_layout"),
            entries: &[
                // Camera uniforms
                wgpu::BindGroupLayoutEntry {
                    binding: 0,
                    visibility: wgpu::ShaderStages::VERTEX,
                    ty: wgpu::BindingType::Buffer {
                        ty: wgpu::BufferBindingType::Uniform,
                        has_dynamic_offset: false,
                        min_binding_size: None,
                    },
                    count: None,
                },
                // Symbol metadata (storage buffer)
                wgpu::BindGroupLayoutEntry {
                    binding: 1,
                    visibility: wgpu::ShaderStages::VERTEX,
                    ty: wgpu::BindingType::Buffer {
                        ty: wgpu::BufferBindingType::Storage { read_only: true },
                        has_dynamic_offset: false,
                        min_binding_size: None,
                    },
                    count: None,
                },
            ],
        });

        // Bind group layout 1: Atlas texture + sampler
        let atlas_layout = device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
            label: Some("symbol_atlas_layout"),
            entries: &[
                wgpu::BindGroupLayoutEntry {
                    binding: 0,
                    visibility: wgpu::ShaderStages::FRAGMENT,
                    ty: wgpu::BindingType::Texture {
                        sample_type: wgpu::TextureSampleType::Float { filterable: true },
                        view_dimension: wgpu::TextureViewDimension::D2,
                        multisampled: false,
                    },
                    count: None,
                },
                wgpu::BindGroupLayoutEntry {
                    binding: 1,
                    visibility: wgpu::ShaderStages::FRAGMENT,
                    ty: wgpu::BindingType::Sampler(wgpu::SamplerBindingType::Filtering),
                    count: None,
                },
            ],
        });

        let pipeline_layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
            label: Some("symbol_pipeline_layout"),
            bind_group_layouts: &[&camera_layout, &atlas_layout],
            push_constant_ranges: &[],
        });

        let pipeline = device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
            label: Some("symbol_pipeline"),
            layout: Some(&pipeline_layout),
            vertex: wgpu::VertexState {
                module: &shader,
                entry_point: Some("vs_main"),
                buffers: &[SymbolInstance::desc()],  // Instance buffer only
                compilation_options: Default::default(),
            },
            fragment: Some(wgpu::FragmentState {
                module: &shader,
                entry_point: Some("fs_main"),
                targets: &[Some(wgpu::ColorTargetState {
                    format: config.format,
                    blend: Some(wgpu::BlendState::ALPHA_BLENDING),
                    write_mask: wgpu::ColorWrites::ALL,
                })],
                compilation_options: Default::default(),
            }),
            primitive: wgpu::PrimitiveState {
                topology: wgpu::PrimitiveTopology::TriangleList,
                strip_index_format: None,
                front_face: wgpu::FrontFace::Ccw,
                cull_mode: None,  // Symbols can face any direction
                polygon_mode: wgpu::PolygonMode::Fill,
                unclipped_depth: false,
                conservative: false,
            },
            depth_stencil: None,  // Render on top, no depth test
            multisample: wgpu::MultisampleState::default(),
            multiview: None,
            cache: None,
        });

        // Create buffers...

        Self {
            pipeline,
            camera_bind_group,
            atlas_bind_group,
            uniform_buffer,
            instance_buffer,
            instance_count: 0,
        }
    }
}
```

### 4.2 Draw Call Strategy

```rust
impl SymbolRenderer {
    pub fn render<'a>(&'a self, pass: &mut wgpu::RenderPass<'a>) {
        if self.instance_count == 0 {
            return;
        }

        pass.set_pipeline(&self.pipeline);
        pass.set_bind_group(0, &self.camera_bind_group, &[]);
        pass.set_bind_group(1, &self.atlas_bind_group, &[]);
        pass.set_vertex_buffer(0, self.instance_buffer.slice(..));

        // Single draw call for ALL symbols
        // 6 vertices per quad (procedurally generated), N instances
        pass.draw(0..6, 0..self.instance_count);
    }
}
```

**Key insight:** One draw call renders all symbol types. The `symbol_id` in each instance selects the UV rect from the metadata storage buffer.

---

## 5. Integration with Existing Renderer

### 5.1 State Structure Modification

```rust
// src/render/state.rs

pub struct State {
    // ... existing fields ...

    // Add symbol renderer
    pub symbol_renderer: SymbolRenderer,
}
```

### 5.2 Instance Update on Chart Load

```rust
impl State {
    pub fn load_chart(&mut self, senc_data: &SencData) {
        // ... existing area/line loading ...

        // Build symbol instances
        let mut instances: Vec<SymbolInstance> = Vec::new();

        for feature in &senc_data.features {
            if let Some(symbol_id) = map_feature_to_symbol(feature) {
                if let Some(position) = get_symbol_position(feature) {
                    instances.push(SymbolInstance {
                        position,
                        symbol_id: symbol_id as u32,
                        rotation: 0.0,
                    });
                }
            }
        }

        // Upload to GPU
        self.symbol_renderer.update_instances(&self.queue, &instances);
    }
}
```

### 5.3 Render Loop Order

```rust
impl State {
    pub fn render(&mut self) -> Result<(), wgpu::SurfaceError> {
        // ...
        {
            let mut pass = encoder.begin_render_pass(/* ... */);

            // 1. Draw area fills (DEPARE, LNDARE)
            self.area_renderer.render(&mut pass);

            // 2. Draw lines (coastlines, depth contours)
            self.line_renderer.render(&mut pass);

            // 3. Draw symbols LAST (on top of everything)
            self.symbol_renderer.render(&mut pass);
        }
        // ...
    }
}
```

---

## 6. Minimal Symbol Mapping from OpenCPN/QuteNav

### 6.1 Extracted Rules

Based on `chartsymbols.xml` and `s52functions.cpp`:

**BOYLAT (Lateral Buoy):**
```
if COLOUR contains 3 (red) or COLOUR contains "3,4,3" -> BOYLAT14 (red)
if COLOUR contains 4 (green) or COLOUR contains "4,3,4" -> BOYLAT13 (green)
if CATLAM = 1 (port) -> BOYLAT14 (red)
if CATLAM = 2 (starboard) -> BOYLAT13 (green)
default -> BOYLAT14
```

**BCNLAT (Lateral Beacon):**
```
if COLOUR contains 3 + BCNSHP = 1,2 -> BCNLAT21
if COLOUR contains 3 + BCNSHP = 3,4 -> BCNLAT15
if COLOUR contains 4 -> BCNLAT15 (green variant)
default -> BCNLAT21
```

**UWTROC (Underwater Rock) via OBSTRN04:**
```
if VALSOU is valid and depth <= 20:
  if WATLEV = 4 or 5 -> UWTROC04
  else -> DANGER51 with sounding
else if no VALSOU:
  if WATLEV = 2 -> LNDARE01 (dry)
  if WATLEV = 3 -> UWTROC03 (awash)
  else -> UWTROC04 (submerged)
```

### 6.2 Simplified Implementation Table

For Phase 1, we ignore:
- BCNSHP (beacon shape) - use single icon per color
- BOYSHP (buoy shape) - use conical as default
- VALSOU depth display - just show rock symbol
- IALA region differences (A vs B) - assume IALA-A

| S-57 Object | Simplified Rule | SymbolId |
|-------------|-----------------|----------|
| BOYLAT | COLOUR has 3 OR CATLAM=1 | BuoyLatPort |
| BOYLAT | COLOUR has 4 OR CATLAM=2 | BuoyLatStarboard |
| BOYLAT | (fallback) | BuoyLatPort |
| BCNLAT | COLOUR has 4 | BeaconLatStarboard |
| BCNLAT | (else) | BeaconLatPort |
| UWTROC | WATLEV = 3 | RockAwash |
| UWTROC | (else) | RockSubmerged |

---

## 7. Performance Considerations

### 7.1 Raspberry Pi 4 Safety Analysis

| Aspect | Design Choice | Pi4 Safe? |
|--------|---------------|-----------|
| Texture size | 1024x1024 RGBA8 = 4MB | Yes, well within VRAM |
| Draw calls | 1 per frame | Yes, minimal overhead |
| Instance buffer | ~1000 symbols x 16 bytes = 16KB | Yes, trivial |
| Shader complexity | Simple UV lookup + transform | Yes, no branching |
| Blending | Alpha blend (one state) | Yes, standard |
| Storage buffer | 256 x 16 bytes = 4KB | Yes, trivial |

### 7.2 Potential Bottlenecks

1. **Large chart with 10,000+ symbols:** Instance buffer at 160KB is still fine. Consider frustum culling if needed.
2. **Frequent zoom changes:** Camera uniform update is 80 bytes per frame - negligible.
3. **Atlas texture filtering:** Linear filtering may be slow on Pi4 for many samples. Mitigation: Symbol quads are small, few texels sampled per symbol.

### 7.3 Recommended Limits

```rust
const MAX_SYMBOLS: usize = 16384;      // 256KB instance buffer
const ATLAS_SIZE: u32 = 1024;           // 4MB texture
const SYMBOL_SIZE_PIXELS: f32 = 24.0;   // Fixed screen size
```

---

## 8. File Structure Summary

```
navcore/
├── assets/
│   └── symbols/
│       ├── atlas.png           # 1024x1024 RGBA texture
│       └── atlas.json          # Symbol metadata
├── shaders/
│   └── symbols.wgsl            # Vertex + fragment shader
└── src/
    ├── render/
    │   ├── mod.rs
    │   ├── state.rs            # Add SymbolRenderer field
    │   └── symbols.rs          # NEW: SymbolRenderer, SymbolAtlas, etc.
    └── senc/
        ├── mod.rs
        └── symbols.rs          # NEW: S-57 -> SymbolId mapping
```

---

## 9. Implementation Checklist

### Pass 1: Infrastructure
- [ ] Create `src/render/symbols.rs` with structs
- [ ] Create `shaders/symbols.wgsl`
- [ ] Create placeholder `assets/symbols/atlas.json`
- [ ] Create placeholder `assets/symbols/atlas.png` (1024x1024 with test patterns)

### Pass 2: Pipeline
- [ ] Implement `SymbolAtlas::load()`
- [ ] Implement `SymbolRenderer::new()`
- [ ] Implement `SymbolRenderer::render()`
- [ ] Add to State struct and render loop

### Pass 3: Feature Mapping
- [ ] Create `src/senc/symbols.rs`
- [ ] Implement `map_feature_to_symbol()`
- [ ] Implement `get_symbol_position()`
- [ ] Wire up in chart loading

### Pass 4: Assets
- [ ] Extract BOYLAT13, BOYLAT14 from OpenCPN/QuteNav resources
- [ ] Extract BCNLAT15, BCNLAT21
- [ ] Extract UWTROC03, UWTROC04
- [ ] Pack into atlas.png with correct layout
- [ ] Update atlas.json with correct pivots

### Pass 5: Testing
- [ ] Load test chart with buoys/beacons/rocks
- [ ] Verify symbols appear at correct positions
- [ ] Verify screen-space sizing works across zoom levels
- [ ] Test on Raspberry Pi 4

---

## References

- QuteNav source files analyzed:
  - `src/rastersymbolmanager.cpp` - Atlas loading and symbol management
  - `src/s52functions.cpp` - OBSTRN04 conditional symbology
  - `data/chartsymbols.xml` - Symbol definitions with graphics-location and pivot
  - `shaders/opengl-desktop/chartpainter-rastersymbol.vert` - Instanced vertex shader
