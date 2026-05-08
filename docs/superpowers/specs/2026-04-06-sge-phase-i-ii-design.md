# SGE Phase I + II Design Spec
**Date:** 2026-04-06  
**Scope:** Phase I (Foundation) + Phase II (Graphics)  
**Target:** Spinning 3D cube rendered through a full DAG-based frame graph  
**Rust edition:** 2024, latest stable toolchain

---

## 1. Goals & Success Criteria

- Cargo workspace compiles cleanly with all four crates
- An `App` runs a window with a fixed-timestep main loop via `winit`
- The renderer initializes `wgpu` (Vulkan/DX12 backend) and presents to the window surface
- A full Frame Graph executes each frame: build → compile (topological sort, dead-pass culling, resource lifetime analysis, transient allocation) → execute
- A spinning 3D cube renders through the frame graph with depth testing and per-vertex color
- `sge-math` provides glam-backed Vec3/Mat4/Quat wrappers and a perspective camera

---

## 2. Workspace Structure

```
SGE/
├── Cargo.toml                       # Workspace root, [workspace.dependencies]
├── README.md
├── crates/
│   ├── sge-core/                    # Phase I: app lifecycle, main loop, windowing
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── app.rs               # App builder + run() entry point
│   │       ├── time.rs              # Fixed-timestep accumulator, Time struct
│   │       └── window.rs            # WindowConfig, winit integration
│   ├── sge-renderer/                # Phase II: wgpu, frame graph, pipelines
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── context.rs           # GpuContext: device, queue, surface
│   │       ├── graph/
│   │       │   ├── mod.rs
│   │       │   ├── builder.rs       # FrameGraph, PassBuilder, ResourceHandle
│   │       │   ├── compiler.rs      # Topo sort, dead-pass culling, lifetime analysis
│   │       │   ├── executor.rs      # Command encoding + queue submission
│   │       │   └── resource.rs      # TextureDesc, BufferDesc, ResourcePool
│   │       ├── pipeline.rs          # PipelineCache, RenderPipelineDesc
│   │       ├── shader.rs            # ShaderModule, .wgsl file loading
│   │       └── buffer.rs            # VertexBuffer, IndexBuffer, UniformBuffer<T>
│   ├── sge-math/                    # glam wrappers + camera
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── transform.rs         # Transform struct → Mat4
│   │       └── camera.rs            # PerspectiveCamera, view/projection matrices
│   └── sge-ecs/                     # Stub (Phase III)
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs
├── examples/
│   └── spinning_cube.rs             # Integration demo
└── assets/
    └── shaders/
        └── cube.wgsl
```

**Crate dependencies:**
- `sge-core` → `winit`
- `sge-renderer` → `wgpu`, `sge-math`
- `sge-math` → `glam`
- `sge-ecs` → *(none)*
- `spinning_cube` example → `sge-core`, `sge-renderer`, `sge-math`

---

## 3. Phase I: sge-core

### 3.1 App Builder

The `App` uses a **trait-based state pattern** to integrate cleanly with winit 0.30+'s `ApplicationHandler`, where window creation and GPU surface creation are deferred until the event loop starts:

```rust
pub trait AppState: Sized + 'static {
    fn init(window: Arc<winit::window::Window>) -> Self;  // called once, after window is ready
    fn on_frame(&mut self, ctx: &FrameContext);
    fn on_resize(&mut self, width: u32, height: u32) {}
    fn on_exit(&mut self) {}
}

App::new()
    .with_window(WindowConfig { title: "SGE", width: 1280, height: 720 })
    .with_fixed_timestep(Duration::from_secs_f64(1.0 / 60.0))
    .run::<MyState>();
```

`App::run::<S>()` creates the winit `EventLoop`, enters it, calls `S::init()` in `ApplicationHandler::resumed()` (which is when the window handle is valid and surfaces can be created), then drives the main loop calling `S::on_frame()` each frame.

This pattern solves the async wgpu initialization problem: `S::init()` can call `pollster::block_on(GpuContext::new(window))` to synchronously await GPU initialization on the first `resumed` event.

### 3.2 Fixed Timestep Loop

Semi-fixed accumulator pattern:
1. Each iteration of the event loop tick measures elapsed wall time
2. Elapsed time is added to an accumulator
3. While accumulator ≥ fixed_dt: run a logic tick, subtract fixed_dt
4. After ticks: render once, with interpolation alpha = accumulator / fixed_dt (passed in `FrameContext`)

This decouples simulation update rate from frame rate.

### 3.3 Time

```rust
pub struct Time {
    pub delta: Duration,        // wall time since last frame
    pub elapsed: Duration,      // total elapsed since start
    pub tick_count: u64,        // number of fixed ticks so far
    pub alpha: f32,             // render interpolation factor [0, 1)
}
```

### 3.4 Window

```rust
pub struct WindowConfig {
    pub title: &'static str,
    pub width: u32,
    pub height: u32,
    pub vsync: bool,
}
```

Wraps `winit 0.30+` using the `ApplicationHandler` trait. Exposes the raw `Arc<winit::window::Window>` for surface creation. Handles resize events by propagating new dimensions to the renderer.

### 3.5 FrameContext

Passed to the user callback each frame:
```rust
pub struct FrameContext<'a> {
    pub time: &'a Time,
    pub window: &'a Window,      // sge-core Window wrapper
}
```

---

## 4. Phase II: sge-renderer

### 4.1 GpuContext

Initialized once from a window handle:
```rust
pub struct GpuContext {
    pub device: wgpu::Device,
    pub queue: wgpu::Queue,
    pub surface: wgpu::Surface<'static>,
    pub surface_config: wgpu::SurfaceConfiguration,
}

impl GpuContext {
    pub async fn new(window: Arc<winit::window::Window>) -> Self { ... }
    pub fn resize(&mut self, width: u32, height: u32) { ... }
    pub fn current_texture(&self) -> wgpu::SurfaceTexture { ... }
}
```

### 4.2 Frame Graph — Design

The frame graph follows the **Frostbite FrameGraph pattern** (GDC 2017). Each frame has three sequential phases:

#### Build
User code calls `FrameGraph::new()`, declares passes via `add_pass()`, and describes each pass's resource reads and writes using a `PassBuilder`. Resources are identified by opaque `ResourceHandle`s (not actual GPU allocations yet).

```rust
let mut graph = FrameGraph::new();

let depth_handle = graph.create_texture(TextureDesc { ... });
let backbuffer = graph.import_texture(surface_view);

graph.add_pass("geometry", |builder: &mut PassBuilder| {
    builder.write_color(backbuffer, ClearColor::rgba(0.1, 0.1, 0.15, 1.0));
    builder.write_depth(depth_handle, ClearDepth(1.0));
    builder.set_render(move |ctx: &mut RenderContext| {
        // draw calls here
    });
});
```

Imported resources (backbuffer) are always treated as "used" — they anchor the culling algorithm.

#### Compile

`graph.compile()` → `CompiledGraph`

Steps:
1. **Build dependency graph**: For each pass, collect all resources it reads (edges from resource → pass) and writes (edges from pass → resource).
2. **Topological sort**: Kahn's algorithm over passes ordered by resource dependencies.
3. **Dead-pass culling**: Walk backwards from passes that write to imported resources; any pass not reachable is culled.
4. **Resource lifetime analysis**: For each transient resource, record its `first_write_pass` and `last_read_pass` indices in the sorted order.
5. **Transient resource allocation**: Assign actual wgpu textures/buffers from a `ResourcePool`, reusing previously-allocated objects whose descriptors match and whose lifetime ended before this resource's lifetime begins. (wgpu abstracts the underlying allocator — this is object reuse, not raw memory aliasing.)

#### Execute

`compiled_graph.execute(&device, &queue, &mut pool)`:
- For each pass in compiled order:
  - Resolve `ResourceHandle`s to actual wgpu views from the pool
  - Create a `wgpu::CommandEncoder`
  - For render passes: construct `RenderPassDescriptor` from the declared attachments
  - Call the pass's render closure with a `RenderContext`
  - Finish and submit the encoder to the queue
- Release transient resources back to pool after last use (by returning the `wgpu::Texture`/`wgpu::Buffer` to the pool's free list, keyed by descriptor)

### 4.3 Key Frame Graph Types

```rust
// Opaque handle — no wgpu allocation until Execute
pub struct ResourceHandle(u32);

// Descriptor for a transient texture
pub struct TextureDesc {
    pub size: wgpu::Extent3d,
    pub format: wgpu::TextureFormat,
    pub usage: wgpu::TextureUsages,
    pub label: Option<&'static str>,
}

// Caches and reuses wgpu textures/buffers across frames
pub struct ResourcePool {
    textures: HashMap<TextureDesc, Vec<wgpu::Texture>>,
    // ...
}

// Thin wrapper passed to render closures
pub struct RenderContext<'pass> {
    pub pass: wgpu::RenderPass<'pass>,
}
```

### 4.4 Pipeline & Shader Management

**PipelineCache**: Stores `wgpu::RenderPipeline` keyed by a hash of `RenderPipelineDesc`. First call creates; subsequent calls return cached.

**ShaderModule**: Loads `.wgsl` from a path, compiles via `device.create_shader_module()`. Stored by path for reuse.

**No hot-reloading in Phase II** — that's a future roadmap item.

### 4.5 Buffer Types

```rust
// Vertex struct for Phase II demo
#[repr(C)]
pub struct Vertex {
    pub position: [f32; 3],
    pub color: [f32; 3],
}

pub struct VertexBuffer { /* wgpu::Buffer, len */ }
pub struct IndexBuffer  { /* wgpu::Buffer, len */ }
pub struct UniformBuffer<T: bytemuck::Pod> {
    pub buffer: wgpu::Buffer,
    _marker: PhantomData<T>,
}

impl<T: bytemuck::Pod> UniformBuffer<T> {
    pub fn write(&self, queue: &wgpu::Queue, data: &T) { ... }
}
```

---

## 5. sge-math

### 5.1 Primitives
Re-export `glam::{Vec2, Vec3, Vec4, Mat4, Quat}` with no wrapping — glam's API is already ergonomic.

### 5.2 Transform

```rust
pub struct Transform {
    pub position: Vec3,
    pub rotation: Quat,
    pub scale: Vec3,
}

impl Transform {
    pub fn matrix(&self) -> Mat4 {
        Mat4::from_scale_rotation_translation(self.scale, self.rotation, self.position)
    }
}
```

### 5.3 PerspectiveCamera

```rust
pub struct PerspectiveCamera {
    pub fov_y: f32,       // radians
    pub aspect: f32,
    pub near: f32,
    pub far: f32,
}

impl PerspectiveCamera {
    pub fn projection(&self) -> Mat4 { /* glam::Mat4::perspective_rh */ }
    pub fn view(eye: Vec3, target: Vec3, up: Vec3) -> Mat4 { /* look_at_rh */ }
}
```

Coordinate system: right-handed (wgpu convention).

---

## 6. sge-ecs (Stub)

Phase III stub — just:
```rust
// sge-ecs/src/lib.rs
// Sparse-set ECS — Phase III implementation pending.
```

---

## 7. Spinning Cube Example

`examples/spinning_cube.rs` demonstrates Phase I + II integration using the trait-based `AppState` pattern:

```rust
struct CubeDemo {
    gpu: GpuContext,
    vertex_buf: VertexBuffer,
    index_buf: IndexBuffer,
    mvp_uniform: UniformBuffer<[f32; 16]>,
    pipeline: wgpu::RenderPipeline,
    bind_group: wgpu::BindGroup,
    camera: PerspectiveCamera,
}

impl AppState for CubeDemo {
    fn init(window: Arc<winit::window::Window>) -> Self {
        let gpu = pollster::block_on(GpuContext::new(window));
        // ... create buffers, pipeline, bind groups ...
    }
    fn on_frame(&mut self, ctx: &FrameContext) { /* see below */ }
    fn on_resize(&mut self, w: u32, h: u32) { self.gpu.resize(w, h); }
}
```

**init() steps:**
1. `GpuContext::new(window)` — initialize wgpu
2. Define 24 cube vertices (4 per face, per-face solid color) and 36 indices
3. Upload via `VertexBuffer::new` and `IndexBuffer::new`
4. Create `UniformBuffer<[f32; 16]>` for MVP
5. Load `cube.wgsl`, create `RenderPipeline` via `PipelineCache`
6. Create bind group layout + bind group for the MVP uniform

**on_frame() steps:**
   - Compute rotation from `time.elapsed` (degrees/sec)
   - `model = Mat4::from_rotation_y(angle)`
   - `view = PerspectiveCamera::view(eye, target, Vec3::Y)`
   - `proj = camera.projection()`
   - `mvp_uniform.write(&queue, (proj * view * model).to_cols_array())`
   - Build frame graph: one `"geometry"` pass writing to backbuffer + depth
   - Compile → execute
   - Present surface texture

### cube.wgsl

```wgsl
struct Uniforms { mvp: mat4x4<f32> }
@group(0) @binding(0) var<uniform> u: Uniforms;

struct VIn  { @location(0) pos: vec3<f32>, @location(1) col: vec3<f32> }
struct VOut { @builtin(position) pos: vec4<f32>, @location(0) col: vec3<f32> }

@vertex
fn vs_main(in: VIn) -> VOut {
    return VOut(u.mvp * vec4(in.pos, 1.0), in.col);
}

@fragment
fn fs_main(in: VOut) -> @location(0) vec4<f32> {
    return vec4(in.col, 1.0);
}
```

---

## 8. Key Dependencies (Cargo)

| Crate | Dependency | Version |
|---|---|---|
| `sge-core` | `winit` | `0.30` |
| `sge-renderer` | `wgpu` | `22` (latest) |
| `sge-renderer` | `bytemuck` | `1` (with `derive` feature) |
| `sge-math` | `glam` | `0.29` |
| `examples` | `pollster` | `0.3` (for blocking on async GPU init) |

---

## 9. Verification

1. `cargo build --workspace` — clean build, no warnings
2. `cargo clippy --workspace -- -D warnings` — passes
3. `cargo run --example spinning_cube` — opens window, colored cube rotates smoothly
4. Resize window — cube re-renders correctly at new dimensions
5. Close window — process exits cleanly
