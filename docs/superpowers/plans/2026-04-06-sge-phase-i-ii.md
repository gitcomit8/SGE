# SGE Phase I + II Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Cargo workspace with a windowed application loop (Phase I) and a DAG-based frame graph renderer that displays a spinning 3D cube (Phase II).

**Architecture:** Four crates — sge-core (winit loop, AppState trait), sge-renderer (wgpu + frame graph), sge-math (glam wrappers), sge-ecs (stub). The frame graph uses a build→compile→execute pattern per frame; draw calls are recorded as a `DrawCommand` enum (not closures) to sidestep Rust lifetime constraints on stored closures.

**Tech Stack:** Rust 2024 edition · winit 0.30 · wgpu 22 · glam 0.29 · bytemuck 1 · pollster 0.3

---

## File Map

```
SGE/
├── Cargo.toml
├── .gitignore
├── crates/
│   ├── sge-math/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── transform.rs
│   │       └── camera.rs
│   ├── sge-ecs/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── sge-core/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── lib.rs
│   │       ├── time.rs
│   │       ├── window.rs
│   │       └── app.rs
│   └── sge-renderer/
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           ├── context.rs
│           ├── buffer.rs
│           ├── shader.rs
│           ├── pipeline.rs
│           └── graph/
│               ├── mod.rs
│               ├── resource.rs
│               ├── builder.rs
│               ├── compiler.rs
│               └── executor.rs
├── examples/
│   └── spinning_cube.rs
└── assets/
    └── shaders/
        └── cube.wgsl
```

---

## Task 1: Workspace Scaffold

**Files:**
- Create: `Cargo.toml` (workspace root)
- Create: `.gitignore`
- Create: `crates/sge-math/Cargo.toml`
- Create: `crates/sge-ecs/Cargo.toml`
- Create: `crates/sge-core/Cargo.toml`
- Create: `crates/sge-renderer/Cargo.toml`

- [ ] **Step 1: Create workspace root `Cargo.toml`**

```toml
[workspace]
resolver = "2"
members = [
    "crates/sge-core",
    "crates/sge-renderer",
    "crates/sge-math",
    "crates/sge-ecs",
]

[workspace.dependencies]
sge-math = { path = "crates/sge-math" }
sge-ecs  = { path = "crates/sge-ecs" }
sge-core = { path = "crates/sge-core" }
sge-renderer = { path = "crates/sge-renderer" }

winit    = "0.30"
wgpu     = "22"
glam     = "0.29"
bytemuck = { version = "1", features = ["derive"] }
pollster = "0.3"
```

- [ ] **Step 2: Create `.gitignore`**

```
/target
Cargo.lock
```

> Keep `Cargo.lock` out of version control for a library/engine workspace. Flip this if you want reproducible CI builds.

- [ ] **Step 3: Create crate `Cargo.toml` files**

`crates/sge-math/Cargo.toml`:
```toml
[package]
name    = "sge-math"
version = "0.1.0"
edition = "2024"

[dependencies]
glam = { workspace = true }
```

`crates/sge-ecs/Cargo.toml`:
```toml
[package]
name    = "sge-ecs"
version = "0.1.0"
edition = "2024"
```

`crates/sge-core/Cargo.toml`:
```toml
[package]
name    = "sge-core"
version = "0.1.0"
edition = "2024"

[dependencies]
winit = { workspace = true }
```

`crates/sge-renderer/Cargo.toml`:
```toml
[package]
name    = "sge-renderer"
version = "0.1.0"
edition = "2024"

[dependencies]
wgpu     = { workspace = true }
bytemuck = { workspace = true }
```

> Note: sge-renderer does NOT depend on sge-math. The example will wire them together.

- [ ] **Step 4: Create stub `src/lib.rs` for every crate**

Each crate gets a minimal `lib.rs` so `cargo build --workspace` succeeds:

`crates/sge-math/src/lib.rs`:
```rust
pub mod camera;
pub mod transform;

pub use glam::{Mat4, Quat, Vec2, Vec3, Vec4};
```

`crates/sge-ecs/src/lib.rs`:
```rust
// Sparse-set ECS — Phase III implementation pending.
```

`crates/sge-core/src/lib.rs`:
```rust
pub mod app;
pub mod time;
pub mod window;

pub use app::{App, AppState, FrameContext};
pub use time::Time;
pub use window::WindowConfig;
```

`crates/sge-renderer/src/lib.rs`:
```rust
pub mod buffer;
pub mod context;
pub mod graph;
pub mod pipeline;
pub mod shader;

pub use context::GpuContext;
```

`crates/sge-renderer/src/graph/mod.rs`:
```rust
pub mod builder;
pub mod compiler;
pub mod executor;
pub mod resource;

pub use builder::{FrameGraph, PassBuilder};
pub use compiler::CompiledGraph;
pub use resource::{ResourceHandle, ResourcePool, TextureDesc};
```

Create placeholder files for each sub-module with just `// TODO`:
- `crates/sge-math/src/transform.rs`
- `crates/sge-math/src/camera.rs`
- `crates/sge-core/src/time.rs`
- `crates/sge-core/src/window.rs`
- `crates/sge-core/src/app.rs`
- `crates/sge-renderer/src/context.rs`
- `crates/sge-renderer/src/buffer.rs`
- `crates/sge-renderer/src/shader.rs`
- `crates/sge-renderer/src/pipeline.rs`
- `crates/sge-renderer/src/graph/resource.rs`
- `crates/sge-renderer/src/graph/builder.rs`
- `crates/sge-renderer/src/graph/compiler.rs`
- `crates/sge-renderer/src/graph/executor.rs`

- [ ] **Step 5: Verify workspace compiles**

```bash
cargo build --workspace
```

Expected: compiles with no errors (placeholder `// TODO` files compile fine).

- [ ] **Step 6: Commit**

```bash
git add Cargo.toml .gitignore crates/ assets/
git commit -m "chore: scaffold Cargo workspace with four crates"
```

---

## Task 2: sge-math

**Files:**
- Write: `crates/sge-math/src/transform.rs`
- Write: `crates/sge-math/src/camera.rs`

- [ ] **Step 1: Write failing tests**

Add to the bottom of `crates/sge-math/src/transform.rs` before the impl:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn identity_matrix_is_identity() {
        let t = Transform::identity();
        assert_eq!(t.matrix(), Mat4::IDENTITY);
    }

    #[test]
    fn translation_only() {
        let t = Transform { position: Vec3::new(1.0, 2.0, 3.0), ..Transform::identity() };
        let m = t.matrix();
        // w-column of translation matrix = [1,2,3,1]
        assert_eq!(m.w_axis, glam::Vec4::new(1.0, 2.0, 3.0, 1.0));
    }
}
```

Add to the bottom of `crates/sge-math/src/camera.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::f32::consts::FRAC_PI_2;

    #[test]
    fn projection_is_finite() {
        let cam = PerspectiveCamera { fov_y: FRAC_PI_2, aspect: 16.0 / 9.0, near: 0.1, far: 100.0 };
        let m = cam.projection();
        assert!(m.to_cols_array().iter().all(|v| v.is_finite()));
    }

    #[test]
    fn view_looks_along_negative_z() {
        // Camera at origin looking along -Z
        let v = PerspectiveCamera::view(Vec3::ZERO, Vec3::new(0.0, 0.0, -1.0), Vec3::Y);
        assert!(v.to_cols_array().iter().all(|v| v.is_finite()));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cargo test -p sge-math
```

Expected: compile error (types not yet defined).

- [ ] **Step 3: Implement `transform.rs`**

```rust
use glam::{Mat4, Quat, Vec3};

pub struct Transform {
    pub position: Vec3,
    pub rotation: Quat,
    pub scale: Vec3,
}

impl Transform {
    pub fn identity() -> Self {
        Self { position: Vec3::ZERO, rotation: Quat::IDENTITY, scale: Vec3::ONE }
    }

    pub fn matrix(&self) -> Mat4 {
        Mat4::from_scale_rotation_translation(self.scale, self.rotation, self.position)
    }
}
```

- [ ] **Step 4: Implement `camera.rs`**

```rust
use glam::{Mat4, Vec3};

pub struct PerspectiveCamera {
    pub fov_y: f32,   // radians
    pub aspect: f32,  // width / height
    pub near: f32,
    pub far: f32,
}

impl PerspectiveCamera {
    /// Right-handed perspective projection matching wgpu's NDC (depth 0..1).
    pub fn projection(&self) -> Mat4 {
        Mat4::perspective_rh(self.fov_y, self.aspect, self.near, self.far)
    }

    /// Right-handed look-at view matrix.
    pub fn view(eye: Vec3, target: Vec3, up: Vec3) -> Mat4 {
        Mat4::look_at_rh(eye, target, up)
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
cargo test -p sge-math
```

Expected: 3 tests pass.

- [ ] **Step 6: Commit**

```bash
git add crates/sge-math/
git commit -m "feat(sge-math): add Transform and PerspectiveCamera with glam backing"
```

---

## Task 3: sge-ecs Stub

**Files:**
- Write: `crates/sge-ecs/src/lib.rs` (already scaffolded — no further action needed)

- [ ] **Step 1: Verify the stub compiles**

```bash
cargo build -p sge-ecs
```

Expected: builds with no warnings.

- [ ] **Step 2: Commit**

```bash
git add crates/sge-ecs/
git commit -m "chore(sge-ecs): add Phase III stub"
```

---

## Task 4: sge-core Time

**Files:**
- Write: `crates/sge-core/src/time.rs`

- [ ] **Step 1: Write failing tests**

```rust
// At the bottom of crates/sge-core/src/time.rs

#[cfg(test)]
mod tests {
    use super::*;
    use std::time::Duration;

    #[test]
    fn accumulator_advances_correctly() {
        let mut acc = FixedTimestep::new(Duration::from_millis(16));
        acc.advance(Duration::from_millis(33)); // ~2 ticks
        assert_eq!(acc.tick_count(), 2);
    }

    #[test]
    fn alpha_is_remainder_fraction() {
        let dt = Duration::from_millis(16);
        let mut acc = FixedTimestep::new(dt);
        acc.advance(Duration::from_millis(24)); // 1 tick + 8ms remainder
        assert_eq!(acc.tick_count(), 1);
        let alpha = acc.alpha();
        assert!(alpha > 0.4 && alpha < 0.6, "alpha={alpha}");
    }

    #[test]
    fn alpha_clamps_to_zero_on_exact_tick() {
        let dt = Duration::from_millis(16);
        let mut acc = FixedTimestep::new(dt);
        acc.advance(Duration::from_millis(16));
        assert_eq!(acc.tick_count(), 1);
        assert_eq!(acc.alpha(), 0.0);
    }

    #[test]
    fn elapsed_accumulates_across_advances() {
        let mut acc = FixedTimestep::new(Duration::from_millis(16));
        acc.advance(Duration::from_millis(10));
        acc.advance(Duration::from_millis(10));
        assert_eq!(acc.elapsed(), Duration::from_millis(20));
    }
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
cargo test -p sge-core time
```

Expected: compile error.

- [ ] **Step 3: Implement `time.rs`**

```rust
use std::time::Duration;

/// Per-frame timing snapshot passed to AppState::on_frame.
pub struct Time {
    pub delta:      Duration, // wall time since last frame
    pub elapsed:    Duration, // total since app start
    pub tick_count: u64,      // cumulative fixed ticks
    pub alpha:      f32,      // render interpolation [0, 1)
}

/// Drives the semi-fixed-timestep accumulator.
pub struct FixedTimestep {
    dt:          Duration,
    accumulator: Duration,
    elapsed:     Duration,
    tick_count:  u64,
}

impl FixedTimestep {
    pub fn new(dt: Duration) -> Self {
        Self { dt, accumulator: Duration::ZERO, elapsed: Duration::ZERO, tick_count: 0 }
    }

    /// Call once per OS frame with the raw wall-clock delta.
    /// Advances the accumulator and fires ticks.
    pub fn advance(&mut self, delta: Duration) {
        self.elapsed += delta;
        self.accumulator += delta;
        while self.accumulator >= self.dt {
            self.accumulator -= self.dt;
            self.tick_count += 1;
        }
    }

    pub fn tick_count(&self) -> u64 { self.tick_count }
    pub fn elapsed(&self) -> Duration { self.elapsed }

    /// Interpolation factor for rendering between fixed ticks.
    pub fn alpha(&self) -> f32 {
        self.accumulator.as_secs_f32() / self.dt.as_secs_f32()
    }

    /// Snapshot into a Time for the current frame.
    pub fn snapshot(&self, delta: Duration) -> Time {
        Time {
            delta,
            elapsed: self.elapsed,
            tick_count: self.tick_count,
            alpha: self.alpha(),
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cargo test -p sge-core time
```

Expected: 4 tests pass.

- [ ] **Step 5: Commit**

```bash
git add crates/sge-core/src/time.rs
git commit -m "feat(sge-core): add fixed-timestep accumulator (Time + FixedTimestep)"
```

---

## Task 5: sge-core App, AppState trait, winit Integration

**Files:**
- Write: `crates/sge-core/src/window.rs`
- Write: `crates/sge-core/src/app.rs`

No GPU = no unit tests here. Verified via the spinning cube example (Task 13).

- [ ] **Step 1: Implement `window.rs`**

```rust
/// Configuration passed to App::with_window().
pub struct WindowConfig {
    pub title:  &'static str,
    pub width:  u32,
    pub height: u32,
    pub vsync:  bool,
}

impl Default for WindowConfig {
    fn default() -> Self {
        Self { title: "SGE", width: 1280, height: 720, vsync: true }
    }
}
```

- [ ] **Step 2: Implement `app.rs`**

```rust
use std::sync::Arc;
use std::time::{Duration, Instant};

use winit::application::ApplicationHandler;
use winit::event::WindowEvent;
use winit::event_loop::{ActiveEventLoop, EventLoop};
use winit::window::{Window, WindowId};

use crate::time::{FixedTimestep, Time};
use crate::window::WindowConfig;

/// Per-frame context passed to AppState::on_frame.
pub struct FrameContext<'a> {
    pub time:   &'a Time,
    pub window: &'a Arc<Window>,
}

/// Implement this trait to drive the SGE application loop.
pub trait AppState: Sized + 'static {
    /// Called once after the OS window is created and surfaces can be allocated.
    /// GPU context initialization belongs here (use pollster::block_on for async).
    fn init(window: Arc<Window>) -> Self;

    /// Called once per rendered frame.
    fn on_frame(&mut self, ctx: &FrameContext<'_>);

    /// Called when the window is resized. Re-configure surfaces here.
    fn on_resize(&mut self, _width: u32, _height: u32) {}

    /// Called just before the event loop exits.
    fn on_exit(&mut self) {}
}

/// Builder for the SGE application.
pub struct App {
    config:   WindowConfig,
    fixed_dt: Duration,
}

impl App {
    pub fn new() -> Self {
        Self { config: WindowConfig::default(), fixed_dt: Duration::from_secs_f64(1.0 / 60.0) }
    }

    pub fn with_window(mut self, config: WindowConfig) -> Self {
        self.config = config;
        self
    }

    pub fn with_fixed_timestep(mut self, dt: Duration) -> Self {
        self.fixed_dt = dt;
        self
    }

    /// Start the event loop. Blocks until the window is closed.
    pub fn run<S: AppState>(self) {
        let event_loop = EventLoop::new().expect("failed to create event loop");
        let mut app = SgeApp::<S>::new(self.config, self.fixed_dt);
        event_loop.run_app(&mut app).expect("event loop error");
    }
}

impl Default for App {
    fn default() -> Self { Self::new() }
}

// ── internal winit handler ────────────────────────────────────────────────────

struct SgeApp<S: AppState> {
    config:      WindowConfig,
    timestep:    FixedTimestep,
    last_frame:  Option<Instant>,
    window:      Option<Arc<Window>>,
    state:       Option<S>,
}

impl<S: AppState> SgeApp<S> {
    fn new(config: WindowConfig, fixed_dt: Duration) -> Self {
        Self {
            config,
            timestep: FixedTimestep::new(fixed_dt),
            last_frame: None,
            window: None,
            state: None,
        }
    }
}

impl<S: AppState> ApplicationHandler for SgeApp<S> {
    fn resumed(&mut self, event_loop: &ActiveEventLoop) {
        let attrs = Window::default_attributes()
            .with_title(self.config.title)
            .with_inner_size(winit::dpi::PhysicalSize::new(self.config.width, self.config.height));
        // Arc<Window> is required for wgpu surface creation ('static lifetime).
        let window = Arc::new(event_loop.create_window(attrs).expect("window creation failed"));
        self.window = Some(Arc::clone(&window));
        self.state = Some(S::init(window));
        self.last_frame = Some(Instant::now());
    }

    fn window_event(
        &mut self,
        event_loop: &ActiveEventLoop,
        _id: WindowId,
        event: WindowEvent,
    ) {
        match event {
            WindowEvent::CloseRequested => {
                if let Some(state) = &mut self.state {
                    state.on_exit();
                }
                event_loop.exit();
            }
            WindowEvent::Resized(size) => {
                if let Some(state) = &mut self.state {
                    state.on_resize(size.width, size.height);
                }
            }
            WindowEvent::RedrawRequested => {
                let now = Instant::now();
                let delta = self.last_frame.map_or(Duration::ZERO, |t| now - t);
                self.last_frame = Some(now);
                self.timestep.advance(delta);

                if let (Some(state), Some(window)) = (&mut self.state, &self.window) {
                    let time = self.timestep.snapshot(delta);
                    let ctx  = FrameContext { time: &time, window };
                    state.on_frame(&ctx);
                }
            }
            _ => {}
        }
    }

    fn about_to_wait(&mut self, _: &ActiveEventLoop) {
        if let Some(window) = &self.window {
            window.request_redraw();
        }
    }
}
```

- [ ] **Step 3: Verify the crate compiles**

```bash
cargo build -p sge-core
```

Expected: compiles with no errors.

- [ ] **Step 4: Commit**

```bash
git add crates/sge-core/
git commit -m "feat(sge-core): add App builder, AppState trait, winit 0.30 event loop"
```

---

## Task 6: sge-renderer GpuContext

**Files:**
- Write: `crates/sge-renderer/src/context.rs`

No unit tests (requires GPU). Verified via spinning_cube example.

- [ ] **Step 1: Implement `context.rs`**

```rust
use std::sync::Arc;
use winit::window::Window;

pub struct GpuContext {
    pub device:         wgpu::Device,
    pub queue:          wgpu::Queue,
    pub surface:        wgpu::Surface<'static>,
    pub surface_config: wgpu::SurfaceConfiguration,
    pub surface_format: wgpu::TextureFormat,
}

impl GpuContext {
    pub async fn new(window: Arc<Window>) -> Self {
        let size = window.inner_size();

        let instance = wgpu::Instance::new(&wgpu::InstanceDescriptor {
            backends: wgpu::Backends::PRIMARY,
            ..Default::default()
        });

        // Safety: surface lives as long as the window (Arc keeps it alive).
        let surface = instance.create_surface(Arc::clone(&window))
            .expect("failed to create surface");

        let adapter = instance
            .request_adapter(&wgpu::RequestAdapterOptions {
                power_preference:       wgpu::PowerPreference::HighPerformance,
                compatible_surface:     Some(&surface),
                force_fallback_adapter: false,
            })
            .await
            .expect("no suitable GPU adapter found");

        let (device, queue) = adapter
            .request_device(&wgpu::DeviceDescriptor {
                label:            Some("SGE Device"),
                required_features: wgpu::Features::empty(),
                required_limits:  wgpu::Limits::default(),
                ..Default::default()
            })
            .await
            .expect("failed to create device");

        let caps          = surface.get_capabilities(&adapter);
        // Prefer sRGB format so colour math is correct.
        let surface_format = caps.formats.iter()
            .find(|f| f.is_srgb())
            .copied()
            .unwrap_or(caps.formats[0]);

        let surface_config = wgpu::SurfaceConfiguration {
            usage:                         wgpu::TextureUsages::RENDER_ATTACHMENT,
            format:                        surface_format,
            width:                         size.width.max(1),
            height:                        size.height.max(1),
            present_mode:                  if caps.present_modes.contains(&wgpu::PresentMode::Mailbox) {
                                               wgpu::PresentMode::Mailbox
                                           } else {
                                               wgpu::PresentMode::AutoVsync
                                           },
            alpha_mode:                    caps.alpha_modes[0],
            view_formats:                  vec![surface_format.add_srgb_suffix()],
            desired_maximum_frame_latency: 2,
        };
        surface.configure(&device, &surface_config);

        Self { device, queue, surface, surface_config, surface_format }
    }

    /// Call from AppState::on_resize to keep the swapchain valid.
    pub fn resize(&mut self, width: u32, height: u32) {
        if width == 0 || height == 0 { return; }
        self.surface_config.width  = width;
        self.surface_config.height = height;
        self.surface.configure(&self.device, &self.surface_config);
    }

    /// Acquire the next frame. Reconfigures on stale/lost surface automatically.
    pub fn current_texture(&mut self) -> Option<wgpu::SurfaceTexture> {
        match self.surface.get_current_texture() {
            Ok(frame) => Some(frame),
            Err(wgpu::SurfaceError::Outdated | wgpu::SurfaceError::Lost) => {
                self.surface.configure(&self.device, &self.surface_config);
                self.surface.get_current_texture().ok()
            }
            Err(_) => None,
        }
    }
}
```

- [ ] **Step 2: Verify it compiles**

```bash
cargo build -p sge-renderer
```

Expected: compiles (context.rs is now fully implemented).

- [ ] **Step 3: Commit**

```bash
git add crates/sge-renderer/src/context.rs
git commit -m "feat(sge-renderer): add GpuContext with wgpu device/queue/surface init"
```

---

## Task 7: sge-renderer Buffer Types

**Files:**
- Write: `crates/sge-renderer/src/buffer.rs`

No unit tests (requires GPU). Verified via spinning_cube example.

- [ ] **Step 1: Implement `buffer.rs`**

```rust
use std::marker::PhantomData;
use std::sync::Arc;
use wgpu::util::DeviceExt as _;  // provides create_buffer_init

/// Vertex layout for Phase II. Position and per-vertex colour.
#[repr(C)]
#[derive(Copy, Clone, bytemuck::Pod, bytemuck::Zeroable)]
pub struct Vertex {
    pub position: [f32; 3],
    pub color:    [f32; 3],
}

impl Vertex {
    pub const ATTRIBS: [wgpu::VertexAttribute; 2] = wgpu::vertex_attr_array![
        0 => Float32x3,
        1 => Float32x3,
    ];

    pub fn layout() -> wgpu::VertexBufferLayout<'static> {
        wgpu::VertexBufferLayout {
            array_stride: std::mem::size_of::<Self>() as u64,
            step_mode:    wgpu::VertexStepMode::Vertex,
            attributes:   &Self::ATTRIBS,
        }
    }
}

pub struct VertexBuffer {
    pub buffer: Arc<wgpu::Buffer>,
    pub count:  u32,
}

impl VertexBuffer {
    pub fn new(device: &wgpu::Device, vertices: &[Vertex]) -> Self {
        let buffer = Arc::new(device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label:    Some("VertexBuffer"),
            contents: bytemuck::cast_slice(vertices),
            usage:    wgpu::BufferUsages::VERTEX,
        }));
        Self { buffer, count: vertices.len() as u32 }
    }
}

pub struct IndexBuffer {
    pub buffer: Arc<wgpu::Buffer>,
    pub count:  u32,
}

impl IndexBuffer {
    pub fn new(device: &wgpu::Device, indices: &[u16]) -> Self {
        let buffer = Arc::new(device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label:    Some("IndexBuffer"),
            contents: bytemuck::cast_slice(indices),
            usage:    wgpu::BufferUsages::INDEX,
        }));
        Self { buffer, count: indices.len() as u32 }
    }
}

/// A GPU-side uniform buffer for a single value of type T.
pub struct UniformBuffer<T: bytemuck::Pod> {
    pub buffer: Arc<wgpu::Buffer>,
    _marker:    PhantomData<T>,
}

impl<T: bytemuck::Pod> UniformBuffer<T> {
    pub fn new(device: &wgpu::Device, label: &str) -> Self {
        let buffer = Arc::new(device.create_buffer(&wgpu::BufferDescriptor {
            label:              Some(label),
            size:               std::mem::size_of::<T>() as u64,
            usage:              wgpu::BufferUsages::UNIFORM | wgpu::BufferUsages::COPY_DST,
            mapped_at_creation: false,
        }));
        Self { buffer, _marker: PhantomData }
    }

    pub fn write(&self, queue: &wgpu::Queue, data: &T) {
        queue.write_buffer(&self.buffer, 0, bytemuck::bytes_of(data));
    }
}
```

- [ ] **Step 2: Verify it compiles**

```bash
cargo build -p sge-renderer
```

- [ ] **Step 3: Commit**

```bash
git add crates/sge-renderer/src/buffer.rs
git commit -m "feat(sge-renderer): add Vertex, VertexBuffer, IndexBuffer, UniformBuffer"
```

---

## Task 8: sge-renderer Graph Resource Types

**Files:**
- Write: `crates/sge-renderer/src/graph/resource.rs`

- [ ] **Step 1: Write failing test**

```rust
// At the bottom of graph/resource.rs

#[cfg(test)]
mod tests {
    use super::*;
    use wgpu::{Extent3d, TextureFormat, TextureUsages};

    #[test]
    fn texture_desc_eq_ignores_label() {
        let a = TextureDesc::new(Extent3d { width: 1280, height: 720, depth_or_array_layers: 1 },
                                 TextureFormat::Depth32Float,
                                 TextureUsages::RENDER_ATTACHMENT)
                             .with_label("depth-pass-a");
        let b = TextureDesc::new(Extent3d { width: 1280, height: 720, depth_or_array_layers: 1 },
                                 TextureFormat::Depth32Float,
                                 TextureUsages::RENDER_ATTACHMENT)
                             .with_label("depth-pass-b");
        assert_eq!(a, b);
        // Same hash despite different labels.
        use std::collections::hash_map::DefaultHasher;
        use std::hash::{Hash, Hasher};
        let hash = |x: &TextureDesc| {
            let mut h = DefaultHasher::new();
            x.hash(&mut h);
            h.finish()
        };
        assert_eq!(hash(&a), hash(&b));
    }
}
```

- [ ] **Step 2: Run to verify it fails**

```bash
cargo test -p sge-renderer graph::resource
```

- [ ] **Step 3: Implement `resource.rs`**

```rust
use std::collections::HashMap;
use std::hash::{Hash, Hasher};
use std::sync::Arc;
use wgpu::{Extent3d, TextureFormat, TextureUsages};

/// Opaque handle to a resource in the frame graph.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct ResourceHandle(pub(super) u32);

/// GPU-format descriptor for a transient texture.
/// The `label` field is excluded from Eq/Hash so the pool can reuse textures
/// with matching GPU properties regardless of debug names.
#[derive(Clone, Debug)]
pub struct TextureDesc {
    pub size:   Extent3d,
    pub format: TextureFormat,
    pub usage:  TextureUsages,
    pub label:  Option<&'static str>,
}

impl TextureDesc {
    pub fn new(size: Extent3d, format: TextureFormat, usage: TextureUsages) -> Self {
        Self { size, format, usage, label: None }
    }
    pub fn with_label(mut self, label: &'static str) -> Self {
        self.label = Some(label);
        self
    }
}

// Manual impls that skip `label`.
impl PartialEq for TextureDesc {
    fn eq(&self, other: &Self) -> bool {
        self.size.width  == other.size.width  &&
        self.size.height == other.size.height &&
        self.size.depth_or_array_layers == other.size.depth_or_array_layers &&
        self.format == other.format &&
        self.usage  == other.usage
    }
}
impl Eq for TextureDesc {}

impl Hash for TextureDesc {
    fn hash<H: Hasher>(&self, state: &mut H) {
        self.size.width.hash(state);
        self.size.height.hash(state);
        self.size.depth_or_array_layers.hash(state);
        self.format.hash(state);
        self.usage.hash(state);
    }
}

/// Reuses wgpu::Texture objects with matching descriptors across frames.
/// This is object-level reuse (not raw memory aliasing — wgpu manages that).
pub struct ResourcePool {
    free: HashMap<TextureDesc, Vec<wgpu::Texture>>,
}

impl ResourcePool {
    pub fn new() -> Self {
        Self { free: HashMap::new() }
    }

    /// Returns a matching free texture or creates a new one.
    pub fn acquire(&mut self, device: &wgpu::Device, desc: &TextureDesc) -> wgpu::Texture {
        if let Some(list) = self.free.get_mut(desc) {
            if let Some(tex) = list.pop() {
                return tex;
            }
        }
        device.create_texture(&wgpu::TextureDescriptor {
            label:             desc.label,
            size:              desc.size,
            mip_level_count:   1,
            sample_count:      1,
            dimension:         wgpu::TextureDimension::D2,
            format:            desc.format,
            usage:             desc.usage,
            view_formats:      &[],
        })
    }

    /// Returns a texture to the free list for future reuse.
    pub fn release(&mut self, desc: TextureDesc, texture: wgpu::Texture) {
        self.free.entry(desc).or_default().push(texture);
    }
}

impl Default for ResourcePool {
    fn default() -> Self { Self::new() }
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cargo test -p sge-renderer graph::resource
```

- [ ] **Step 5: Commit**

```bash
git add crates/sge-renderer/src/graph/resource.rs
git commit -m "feat(sge-renderer): add ResourceHandle, TextureDesc, ResourcePool"
```

---

## Task 9: sge-renderer Graph Builder

**Files:**
- Write: `crates/sge-renderer/src/graph/builder.rs`

- [ ] **Step 1: Implement `builder.rs`**

```rust
use std::sync::Arc;
use wgpu::IndexFormat;

use super::resource::{ResourceHandle, TextureDesc};

// ── draw commands ─────────────────────────────────────────────────────────────

/// A recorded GPU command. Stored in PassDecl and replayed during execution.
/// Uses Arc wrappers because wgpu handles are not Clone.
pub enum DrawCommand {
    SetPipeline    { pipeline:    Arc<wgpu::RenderPipeline> },
    SetVertexBuffer{ slot: u32,   buffer: Arc<wgpu::Buffer> },
    SetIndexBuffer { buffer:      Arc<wgpu::Buffer>, format: IndexFormat },
    SetBindGroup   { index: u32,  bind_group: Arc<wgpu::BindGroup>, offsets: Vec<u32> },
    DrawIndexed    { indices: std::ops::Range<u32>, base_vertex: i32, instances: std::ops::Range<u32> },
}

// ── attachments ───────────────────────────────────────────────────────────────

pub struct ColorAttachment {
    pub handle: ResourceHandle,
    pub clear:  Option<wgpu::Color>,
}

pub struct DepthAttachment {
    pub handle:       ResourceHandle,
    pub clear_depth:  Option<f32>,
    pub clear_stencil: Option<u32>,
}

// ── pass declaration ──────────────────────────────────────────────────────────

pub struct PassDecl {
    pub name:             &'static str,
    pub color_attachments: Vec<ColorAttachment>,
    pub depth_attachment:  Option<DepthAttachment>,
    pub reads:            Vec<ResourceHandle>,
    pub commands:         Vec<DrawCommand>,
}

impl PassDecl {
    /// All resource handles this pass writes to (for dependency graph construction).
    pub fn writes(&self) -> impl Iterator<Item = ResourceHandle> + '_ {
        self.color_attachments.iter().map(|a| a.handle)
            .chain(self.depth_attachment.iter().map(|a| a.handle))
    }
}

// ── pass builder ─────────────────────────────────────────────────────────────

/// Passed to the setup closure in FrameGraph::add_pass.
pub struct PassBuilder {
    pub(super) decl: PassDecl,
}

impl PassBuilder {
    fn new(name: &'static str) -> Self {
        Self { decl: PassDecl {
            name,
            color_attachments: vec![],
            depth_attachment:  None,
            reads:             vec![],
            commands:          vec![],
        }}
    }

    /// Declare a color attachment to write (and optionally clear) each frame.
    pub fn write_color(&mut self, handle: ResourceHandle, clear: wgpu::Color) -> &mut Self {
        self.decl.color_attachments.push(ColorAttachment { handle, clear: Some(clear) });
        self
    }

    /// Declare a depth attachment to write (and optionally clear) each frame.
    pub fn write_depth(&mut self, handle: ResourceHandle, clear: f32) -> &mut Self {
        self.decl.depth_attachment = Some(DepthAttachment {
            handle, clear_depth: Some(clear), clear_stencil: None,
        });
        self
    }

    /// Declare a texture this pass reads (creates a dependency edge in the graph).
    pub fn read_texture(&mut self, handle: ResourceHandle) -> &mut Self {
        self.decl.reads.push(handle);
        self
    }

    pub fn set_pipeline(&mut self, pipeline: Arc<wgpu::RenderPipeline>) -> &mut Self {
        self.decl.commands.push(DrawCommand::SetPipeline { pipeline });
        self
    }

    pub fn set_vertex_buffer(&mut self, slot: u32, buffer: Arc<wgpu::Buffer>) -> &mut Self {
        self.decl.commands.push(DrawCommand::SetVertexBuffer { slot, buffer });
        self
    }

    pub fn set_index_buffer(&mut self, buffer: Arc<wgpu::Buffer>, format: IndexFormat) -> &mut Self {
        self.decl.commands.push(DrawCommand::SetIndexBuffer { buffer, format });
        self
    }

    pub fn set_bind_group(&mut self, index: u32, bind_group: Arc<wgpu::BindGroup>) -> &mut Self {
        self.decl.commands.push(DrawCommand::SetBindGroup { index, bind_group, offsets: vec![] });
        self
    }

    pub fn draw_indexed(
        &mut self,
        indices: std::ops::Range<u32>,
        base_vertex: i32,
        instances: std::ops::Range<u32>,
    ) -> &mut Self {
        self.decl.commands.push(DrawCommand::DrawIndexed { indices, base_vertex, instances });
        self
    }
}

// ── frame graph ───────────────────────────────────────────────────────────────

pub(super) enum ResourceEntry {
    Transient { desc: TextureDesc },
    Imported  { view: Arc<wgpu::TextureView> },
}

pub struct FrameGraph {
    pub(super) resources: Vec<ResourceEntry>,
    pub(super) passes:    Vec<PassDecl>,
}

impl FrameGraph {
    pub fn new() -> Self {
        Self { resources: vec![], passes: vec![] }
    }

    /// Register a transient texture (allocated from ResourcePool during execution).
    pub fn create_texture(&mut self, name: &'static str, desc: TextureDesc) -> ResourceHandle {
        let handle = ResourceHandle(self.resources.len() as u32);
        self.resources.push(ResourceEntry::Transient { desc: desc.with_label(name) });
        handle
    }

    /// Register an imported texture view (e.g. the swapchain backbuffer).
    /// Imported resources are always treated as "used" — they anchor culling.
    pub fn import_texture(&mut self, view: Arc<wgpu::TextureView>) -> ResourceHandle {
        let handle = ResourceHandle(self.resources.len() as u32);
        self.resources.push(ResourceEntry::Imported { view });
        handle
    }

    /// Add a render pass. The setup closure runs immediately to record attachments
    /// and draw commands into the PassDecl.
    pub fn add_pass(&mut self, name: &'static str, setup: impl FnOnce(&mut PassBuilder)) {
        let mut builder = PassBuilder::new(name);
        setup(&mut builder);
        self.passes.push(builder.decl);
    }

    /// Compile the frame graph into an executable form.
    pub fn compile(self) -> super::compiler::CompiledGraph {
        super::compiler::compile(self)
    }
}

impl Default for FrameGraph {
    fn default() -> Self { Self::new() }
}
```

- [ ] **Step 2: Verify it compiles**

```bash
cargo build -p sge-renderer
```

- [ ] **Step 3: Commit**

```bash
git add crates/sge-renderer/src/graph/builder.rs
git commit -m "feat(sge-renderer): add FrameGraph builder with DrawCommand recording"
```

---

## Task 10: sge-renderer Graph Compiler

**Files:**
- Write: `crates/sge-renderer/src/graph/compiler.rs`

This is the most complex file. Pure logic, fully unit-testable without a GPU.

- [ ] **Step 1: Write failing tests first**

```rust
// Place at the bottom of compiler.rs

#[cfg(test)]
mod tests {
    use super::*;
    use crate::graph::builder::FrameGraph;
    use crate::graph::resource::TextureDesc;
    use wgpu::{Extent3d, TextureFormat, TextureUsages};

    fn depth_desc() -> TextureDesc {
        TextureDesc::new(
            Extent3d { width: 1280, height: 720, depth_or_array_layers: 1 },
            TextureFormat::Depth32Float,
            TextureUsages::RENDER_ATTACHMENT,
        )
    }

    fn dummy_view() -> std::sync::Arc<wgpu::TextureView> {
        // We can't create a real TextureView without a device.
        // Use a placeholder via transmute — only for compile testing.
        // In real tests this is fine because we never call execute().
        // SAFETY: we only test the compiler's graph logic, not GPU calls.
        unsafe {
            let raw = 1usize as *const ();
            std::sync::Arc::from_raw(raw as *const wgpu::TextureView)
        }
    }

    // Helper: build a simple graph with just a geometry pass → backbuffer.
    fn simple_graph() -> (FrameGraph, crate::graph::resource::ResourceHandle) {
        let mut g = FrameGraph::new();
        let bb = g.import_texture(dummy_view());
        g.add_pass("geometry", |b| {
            b.write_color(bb, wgpu::Color::BLACK);
        });
        (g, bb)
    }

    #[test]
    fn single_pass_appears_in_order() {
        let (g, _) = simple_graph();
        let compiled = g.compile();
        assert_eq!(compiled.pass_names(), vec!["geometry"]);
    }

    #[test]
    fn dead_pass_is_culled() {
        let mut g = FrameGraph::new();
        let bb        = g.import_texture(dummy_view());
        let transient = g.create_texture("t", depth_desc());

        // "dead" only writes to a transient texture nobody reads.
        g.add_pass("dead", |b| { b.write_depth(transient, 1.0); });
        g.add_pass("geometry", |b| { b.write_color(bb, wgpu::Color::BLACK); });

        let compiled = g.compile();
        assert_eq!(compiled.pass_names(), vec!["geometry"]);
    }

    #[test]
    fn dependency_ordering_respected() {
        let mut g    = FrameGraph::new();
        let bb        = g.import_texture(dummy_view());
        let shadow    = g.create_texture("shadow", depth_desc());

        // "shadows" must run before "geometry" because geometry reads shadow.
        g.add_pass("geometry", |b| {
            b.read_texture(shadow);
            b.write_color(bb, wgpu::Color::BLACK);
        });
        g.add_pass("shadows", |b| { b.write_depth(shadow, 1.0); });

        let compiled = g.compile();
        let names = compiled.pass_names();
        let s_idx = names.iter().position(|n| *n == "shadows").unwrap();
        let g_idx = names.iter().position(|n| *n == "geometry").unwrap();
        assert!(s_idx < g_idx, "shadows must precede geometry");
    }

    #[test]
    fn diamond_dependency_all_live() {
        let mut g    = FrameGraph::new();
        let bb       = g.import_texture(dummy_view());
        let r1       = g.create_texture("r1", depth_desc());
        let r2       = g.create_texture("r2", depth_desc());

        // "merge" reads r1 and r2, both of which are written by separate passes.
        g.add_pass("merge", |b| {
            b.read_texture(r1);
            b.read_texture(r2);
            b.write_color(bb, wgpu::Color::BLACK);
        });
        g.add_pass("pass_a", |b| { b.write_depth(r1, 1.0); });
        g.add_pass("pass_b", |b| { b.write_depth(r2, 1.0); });

        let compiled = g.compile();
        let names = compiled.pass_names();
        assert!(names.contains(&"pass_a"));
        assert!(names.contains(&"pass_b"));
        assert!(names.contains(&"merge"));
        let m = names.iter().position(|n| *n == "merge").unwrap();
        let a = names.iter().position(|n| *n == "pass_a").unwrap();
        let b = names.iter().position(|n| *n == "pass_b").unwrap();
        assert!(a < m && b < m);
    }

    #[test]
    fn transitive_culling() {
        // If the anchor pass is removed, both A and B must be culled.
        let mut g = FrameGraph::new();
        let bb    = g.import_texture(dummy_view());
        let r1    = g.create_texture("r1", depth_desc());
        let r2    = g.create_texture("r2", depth_desc());

        // A writes r1, B reads r1 and writes r2, C reads r2 and writes bb.
        // We do NOT add pass C — so A and B should both be culled.
        g.add_pass("a", |b| { b.write_depth(r1, 1.0); });
        g.add_pass("b", |b| { b.read_texture(r1); b.write_depth(r2, 1.0); });
        // (no pass that writes to bb)
        let _ = bb; // silence unused-variable warning

        let compiled = g.compile();
        assert!(compiled.pass_names().is_empty(), "expected all passes culled");
    }

    #[test]
    fn resource_lifetime_is_correct() {
        // depth is written by pass 0 ("geometry"), never read again.
        // Its lifetime should be [0, 0].
        let mut g  = FrameGraph::new();
        let bb     = g.import_texture(dummy_view());
        let depth  = g.create_texture("depth", depth_desc());

        g.add_pass("geometry", |b| {
            b.write_color(bb, wgpu::Color::BLACK);
            b.write_depth(depth, 1.0);
        });

        let compiled = g.compile();
        let lt = compiled.resource_lifetime(depth).expect("depth must have a lifetime");
        assert_eq!(lt.first_write, 0);
        assert_eq!(lt.last_read,   0);
    }

    #[test]
    fn cycle_causes_panic() {
        // Pass A writes R1, pass B reads R1 and writes R2, pass A also reads R2.
        // This forms a cycle; compile() should panic with a clear message.
        let result = std::panic::catch_unwind(|| {
            let mut g = FrameGraph::new();
            let bb = g.import_texture(dummy_view());
            let r1 = g.create_texture("r1", depth_desc());
            let r2 = g.create_texture("r2", depth_desc());

            g.add_pass("a", |b| { b.write_color(bb, wgpu::Color::BLACK); b.read_texture(r2); b.write_depth(r1, 1.0); });
            g.add_pass("b", |b| { b.read_texture(r1); b.write_depth(r2, 1.0); });
            g.compile()
        });
        assert!(result.is_err(), "cyclic graph must panic");
    }
}
```

- [ ] **Step 2: Run to verify they fail**

```bash
cargo test -p sge-renderer graph::compiler
```

Expected: compile error (CompiledGraph not yet defined).

- [ ] **Step 3: Implement `compiler.rs`**

```rust
use std::collections::HashMap;
use std::collections::VecDeque;

use super::builder::{FrameGraph, PassDecl};
use super::resource::{ResourceHandle, TextureDesc};
use super::builder::ResourceEntry;

#[derive(Debug, Clone, Copy)]
pub struct ResourceLifetime {
    pub first_write: usize, // index in sorted pass order
    pub last_read:   usize,
}

/// The output of FrameGraph::compile(). Ready to hand to the executor.
pub struct CompiledGraph {
    /// Passes in dependency-resolved execution order (dead passes removed).
    pub passes:             Vec<PassDecl>,
    pub resource_lifetimes: HashMap<ResourceHandle, ResourceLifetime>,
    pub resources:          Vec<ResourceEntry>,
}

impl CompiledGraph {
    /// Pass names in execution order — used primarily in tests.
    pub fn pass_names(&self) -> Vec<&str> {
        self.passes.iter().map(|p| p.name).collect()
    }

    pub fn resource_lifetime(&self, handle: ResourceHandle) -> Option<ResourceLifetime> {
        self.resource_lifetimes.get(&handle).copied()
    }
}

/// Core compilation function: topological sort + dead-pass culling + lifetimes.
pub(super) fn compile(graph: FrameGraph) -> CompiledGraph {
    let FrameGraph { resources, passes } = graph;
    let n = passes.len();

    // ── Step 1: Identify which resources are "imported" (anchors for culling) ──
    let imported: Vec<bool> = resources.iter().map(|r| matches!(r, ResourceEntry::Imported { .. })).collect();

    // ── Step 2: Build write-map: resource → pass index that writes it ──────────
    // (A resource may only be written by one pass in our model.)
    let mut writer_of: HashMap<ResourceHandle, usize> = HashMap::new();
    for (pi, pass) in passes.iter().enumerate() {
        for handle in pass.writes() {
            writer_of.insert(handle, pi);
        }
    }

    // ── Step 3: Build adjacency + in-degree for Kahn's algorithm ─────────────
    // Edge: writer_of[r] → pass that reads r.
    let mut adj:     Vec<Vec<usize>> = vec![vec![]; n]; // adj[from] = [to]
    let mut in_deg:  Vec<usize>      = vec![0; n];

    for (pi, pass) in passes.iter().enumerate() {
        for &dep_handle in &pass.reads {
            if let Some(&src) = writer_of.get(&dep_handle) {
                adj[src].push(pi);
                in_deg[pi] += 1;
            }
        }
    }

    // ── Step 4: Kahn's topological sort ────────────────────────────────────────
    let mut queue: VecDeque<usize> = (0..n).filter(|&i| in_deg[i] == 0).collect();
    let mut sorted: Vec<usize>     = Vec::with_capacity(n);

    while let Some(pi) = queue.pop_front() {
        sorted.push(pi);
        for &next in &adj[pi] {
            in_deg[next] -= 1;
            if in_deg[next] == 0 {
                queue.push_back(next);
            }
        }
    }

    assert_eq!(sorted.len(), n, "frame graph contains a cycle — this is not a valid DAG");

    // ── Step 5: Dead-pass culling ───────────────────────────────────────────────
    // Walk backwards from passes that write to imported (anchored) resources.
    // A pass is live iff it transitively produces output consumed by an imported resource.
    let mut live = vec![false; n];

    // Seed: any pass that writes to an imported resource is live.
    for (pi, pass) in passes.iter().enumerate() {
        if pass.writes().any(|h| imported.get(h.0 as usize).copied().unwrap_or(false)) {
            live[pi] = true;
        }
    }

    // Propagate liveness backwards (in reverse topological order).
    for &pi in sorted.iter().rev() {
        if live[pi] {
            // Every pass that writes a resource this pass reads is also live.
            for &dep_handle in &passes[pi].reads {
                if let Some(&src) = writer_of.get(&dep_handle) {
                    live[src] = true;
                }
            }
        }
    }

    // Build the final ordered list of live passes.
    let live_passes: Vec<PassDecl> = sorted.into_iter()
        .filter(|&pi| live[pi])
        .map(|pi| {
            // SAFETY: we consume `passes` by index. Build an index-to-PassDecl map first.
            pi  // placeholder — see below
        })
        .collect::<Vec<_>>()
        // We need to consume `passes`, so we do this differently:
        // convert passes into a Vec<Option<PassDecl>>, take by index.
        ;

    // Rebuild properly (Rust doesn't let us index-consume a Vec directly):
    let mut pass_slots: Vec<Option<PassDecl>> = passes.into_iter().map(Some).collect();
    let live_passes: Vec<PassDecl> = sorted_live_indices(n, &live, &adj_original(n, &pass_slots, &writer_of))
        .into_iter()
        .map(|pi| pass_slots[pi].take().unwrap())
        .collect();

    // ── Step 6: Resource lifetime analysis ────────────────────────────────────
    let mut lifetimes: HashMap<ResourceHandle, ResourceLifetime> = HashMap::new();

    for (order_idx, pass) in live_passes.iter().enumerate() {
        for handle in pass.writes() {
            lifetimes.entry(handle).and_modify(|lt| lt.last_read = order_idx).or_insert(ResourceLifetime {
                first_write: order_idx,
                last_read:   order_idx,
            });
        }
        for &handle in &pass.reads {
            lifetimes.entry(handle).and_modify(|lt| lt.last_read = order_idx).or_insert(ResourceLifetime {
                first_write: order_idx,
                last_read:   order_idx,
            });
        }
    }

    CompiledGraph { passes: live_passes, resource_lifetimes: lifetimes, resources }
}
```

> **Note:** The code above has a structural issue — the `sorted_live_indices` helper and `adj_original` are referenced before definition, and the intermediate `live_passes` computation is split awkwardly. Below is the clean, complete, working version to replace the above entirely:

```rust
use std::collections::HashMap;
use std::collections::VecDeque;

use super::builder::{FrameGraph, PassDecl, ResourceEntry};
use super::resource::{ResourceHandle, TextureDesc};

#[derive(Debug, Clone, Copy)]
pub struct ResourceLifetime {
    pub first_write: usize,
    pub last_read:   usize,
}

pub struct CompiledGraph {
    pub passes:             Vec<PassDecl>,
    pub resource_lifetimes: HashMap<ResourceHandle, ResourceLifetime>,
    pub resources:          Vec<ResourceEntry>,
}

impl CompiledGraph {
    pub fn pass_names(&self) -> Vec<&str> {
        self.passes.iter().map(|p| p.name).collect()
    }
    pub fn resource_lifetime(&self, handle: ResourceHandle) -> Option<ResourceLifetime> {
        self.resource_lifetimes.get(&handle).copied()
    }
}

pub(super) fn compile(graph: FrameGraph) -> CompiledGraph {
    let FrameGraph { resources, passes } = graph;
    let n = passes.len();

    // Identify imported (anchor) resources.
    let imported: Vec<bool> = resources.iter()
        .map(|r| matches!(r, ResourceEntry::Imported { .. }))
        .collect();

    // Build write-map: resource handle → index of the pass that writes it.
    let mut writer_of: HashMap<ResourceHandle, usize> = HashMap::new();
    for (pi, pass) in passes.iter().enumerate() {
        for handle in pass.writes() {
            writer_of.insert(handle, pi);
        }
    }

    // Build adjacency list and in-degree for Kahn's algorithm.
    // Edge: pi → qi means qi depends on pi (qi reads something pi writes).
    let mut adj    = vec![vec![]; n];
    let mut in_deg = vec![0usize; n];
    for (qi, pass) in passes.iter().enumerate() {
        for &dep in &pass.reads {
            if let Some(&pi) = writer_of.get(&dep) {
                if pi != qi {
                    adj[pi].push(qi);
                    in_deg[qi] += 1;
                }
            }
        }
    }

    // Kahn's topological sort.
    let mut queue: VecDeque<usize> = (0..n).filter(|&i| in_deg[i] == 0).collect();
    let mut sorted = Vec::with_capacity(n);
    while let Some(pi) = queue.pop_front() {
        sorted.push(pi);
        for &next in &adj[pi] {
            in_deg[next] -= 1;
            if in_deg[next] == 0 {
                queue.push_back(next);
            }
        }
    }
    assert_eq!(sorted.len(), n, "frame graph contains a cycle");

    // Dead-pass culling: backward liveness from imported anchors.
    let mut live = vec![false; n];
    for (pi, pass) in passes.iter().enumerate() {
        if pass.writes().any(|h| imported.get(h.0 as usize).copied().unwrap_or(false)) {
            live[pi] = true;
        }
    }
    for &pi in sorted.iter().rev() {
        if live[pi] {
            for &dep in &passes[pi].reads {
                if let Some(&src) = writer_of.get(&dep) {
                    live[src] = true;
                }
            }
        }
    }

    // Consume passes in topological order, keeping only live ones.
    let mut slots: Vec<Option<PassDecl>> = passes.into_iter().map(Some).collect();
    let live_passes: Vec<PassDecl> = sorted.iter()
        .filter(|&&pi| live[pi])
        .map(|&pi| slots[pi].take().unwrap())
        .collect();

    // Resource lifetime analysis over the live ordered passes.
    let mut lifetimes: HashMap<ResourceHandle, ResourceLifetime> = HashMap::new();
    let update = |lt: &mut ResourceLifetime, idx: usize| lt.last_read = lt.last_read.max(idx);

    for (idx, pass) in live_passes.iter().enumerate() {
        for handle in pass.writes() {
            lifetimes.entry(handle)
                .and_modify(|lt| update(lt, idx))
                .or_insert(ResourceLifetime { first_write: idx, last_read: idx });
        }
        for &handle in &pass.reads {
            lifetimes.entry(handle)
                .and_modify(|lt| update(lt, idx))
                .or_insert(ResourceLifetime { first_write: idx, last_read: idx });
        }
    }

    CompiledGraph { passes: live_passes, resource_lifetimes: lifetimes, resources }
}
```

> **Compiler note on `dummy_view()` in tests:** The test creates a fake `Arc<wgpu::TextureView>` via pointer transmutation to avoid needing a GPU in unit tests. The test file must mark the module `#[allow(invalid_reference_casting)]` or the tests must use `ManuallyDrop` to avoid the destructor running. A cleaner alternative: wrap `Option<Arc<wgpu::TextureView>>` in the `Imported` variant and use `None` in tests. Choose whichever approach compiles cleanly.

- [ ] **Step 4: Run tests to verify they pass**

```bash
cargo test -p sge-renderer graph::compiler
```

Expected: 7 tests pass.

- [ ] **Step 5: Commit**

```bash
git add crates/sge-renderer/src/graph/compiler.rs
git commit -m "feat(sge-renderer): add frame graph compiler (topo sort, culling, lifetimes)"
```

---

## Task 11: sge-renderer Graph Executor

**Files:**
- Write: `crates/sge-renderer/src/graph/executor.rs`

No unit tests (requires GPU). Verified via spinning_cube example.

- [ ] **Step 1: Implement `executor.rs`**

```rust
use std::collections::HashMap;
use std::sync::Arc;

use super::builder::{DrawCommand, ResourceEntry};
use super::compiler::CompiledGraph;
use super::resource::{ResourceHandle, ResourcePool};

impl CompiledGraph {
    /// Execute all live passes.
    ///
    /// `imported_views`: maps each imported ResourceHandle → its TextureView.
    ///   The caller is responsible for providing the backbuffer view here.
    ///
    /// One CommandEncoder is created for the entire frame — more efficient than
    /// one per pass. The encoder is finished and submitted at the end.
    pub fn execute(
        self,
        device:         &wgpu::Device,
        queue:          &wgpu::Queue,
        pool:           &mut ResourcePool,
        imported_views: &HashMap<ResourceHandle, Arc<wgpu::TextureView>>,
    ) {
        // Resolve all resource handles to TextureViews.
        // Transient resources are acquired from the pool; imported ones come from the caller.
        let mut resolved: HashMap<ResourceHandle, (Option<wgpu::Texture>, wgpu::TextureView)> =
            HashMap::new();

        for (idx, entry) in self.resources.iter().enumerate() {
            let handle = ResourceHandle(idx as u32);
            match entry {
                ResourceEntry::Imported { view } => {
                    // We store None for the texture — no cleanup needed for imported.
                    resolved.insert(handle, (None, Arc::clone(view).as_ref().clone()));
                    // Note: wgpu::TextureView is not Clone. Instead store Arc and deref when needed.
                    // See below — we keep Arc<TextureView> for imported and create owned TextureView for transient.
                }
                ResourceEntry::Transient { desc } => {
                    if self.resource_lifetimes.contains_key(&handle) {
                        let tex  = pool.acquire(device, desc);
                        let view = tex.create_view(&wgpu::TextureViewDescriptor::default());
                        resolved.insert(handle, (Some(tex), view));
                    }
                }
            }
        }

        // Build and record all passes into a single command encoder.
        let mut encoder = device.create_command_encoder(
            &wgpu::CommandEncoderDescriptor { label: Some("SGE Frame") }
        );

        // We need TextureView references that outlive each render pass scope.
        // Re-derive views from Arc for imported, keep owned TextureView for transient.
        // This section resolves handles to &TextureView.
        //
        // Implementation note: because wgpu::TextureView is not Clone, we store
        // imported views separately as Arc<TextureView> and deref them.
        // Transient views are owned in `resolved`.

        // Separate storage so borrows work cleanly:
        let mut transient_views: HashMap<ResourceHandle, wgpu::TextureView> = HashMap::new();
        let mut transient_textures: Vec<(ResourceHandle, super::resource::TextureDesc, wgpu::Texture)> = vec![];

        for (idx, entry) in self.resources.iter().enumerate() {
            let handle = ResourceHandle(idx as u32);
            if let ResourceEntry::Transient { desc } = entry {
                if self.resource_lifetimes.contains_key(&handle) {
                    let tex  = pool.acquire(device, desc);
                    let view = tex.create_view(&wgpu::TextureViewDescriptor::default());
                    transient_views.insert(handle, view);
                    transient_textures.push((handle, desc.clone(), tex));
                }
            }
        }

        for (pass_idx, pass) in self.passes.iter().enumerate() {
            // Build color attachment descriptors.
            let color_attachments: Vec<Option<wgpu::RenderPassColorAttachment<'_>>> = pass
                .color_attachments
                .iter()
                .map(|ca| {
                    let view: &wgpu::TextureView = if let Some(arc) =
                        imported_views.get(&ca.handle)
                    {
                        arc.as_ref()
                    } else {
                        transient_views.get(&ca.handle).expect("unresolved color attachment")
                    };
                    Some(wgpu::RenderPassColorAttachment {
                        view,
                        resolve_target: None,
                        ops: wgpu::Operations {
                            load: ca.clear.map_or(
                                wgpu::LoadOp::Load,
                                |c| wgpu::LoadOp::Clear(c),
                            ),
                            store: wgpu::StoreOp::Store,
                        },
                        depth_slice: None, // required in wgpu 22
                    })
                })
                .collect();

            // Build depth attachment descriptor.
            let depth_stencil_attachment =
                pass.depth_attachment.as_ref().map(|da| {
                    let view: &wgpu::TextureView = if let Some(arc) =
                        imported_views.get(&da.handle)
                    {
                        arc.as_ref()
                    } else {
                        transient_views.get(&da.handle).expect("unresolved depth attachment")
                    };
                    wgpu::RenderPassDepthStencilAttachment {
                        view,
                        depth_ops: da.clear_depth.map(|d| wgpu::Operations {
                            load:  wgpu::LoadOp::Clear(d),
                            store: wgpu::StoreOp::Store,
                        }),
                        stencil_ops: None,
                    }
                });

            let mut render_pass = encoder.begin_render_pass(&wgpu::RenderPassDescriptor {
                label:                    Some(pass.name),
                color_attachments:        &color_attachments,
                depth_stencil_attachment: depth_stencil_attachment.as_ref(),
                timestamp_writes:         None,
                occlusion_query_set:      None,
            });

            // Replay draw commands.
            // IMPORTANT: use explicit `let` bindings before passing Arc-derefs into
            // render_pass methods — the borrow from Deref must outlive the render pass scope.
            for cmd in &pass.commands {
                match cmd {
                    DrawCommand::SetPipeline { pipeline } => {
                        let p = pipeline.as_ref();
                        render_pass.set_pipeline(p);
                    }
                    DrawCommand::SetVertexBuffer { slot, buffer } => {
                        let b = buffer.as_ref();          // keeps borrow alive
                        render_pass.set_vertex_buffer(*slot, b.slice(..));
                    }
                    DrawCommand::SetIndexBuffer { buffer, format } => {
                        let b = buffer.as_ref();
                        render_pass.set_index_buffer(b.slice(..), *format);
                    }
                    DrawCommand::SetBindGroup { index, bind_group, offsets } => {
                        let bg = bind_group.as_ref();
                        render_pass.set_bind_group(*index, bg, offsets);
                    }
                    DrawCommand::DrawIndexed { indices, base_vertex, instances } => {
                        render_pass.draw_indexed(
                            indices.clone(),
                            *base_vertex,
                            instances.clone(),
                        );
                    }
                }
            }
        }

        queue.submit(std::iter::once(encoder.finish()));

        // Return transient resources to the pool.
        for (handle, desc, tex) in transient_textures {
            let lt = self.resource_lifetimes.get(&handle);
            if lt.map_or(true, |lt| lt.last_read < self.passes.len()) {
                pool.release(desc, tex);
            }
        }
    }
}
```

- [ ] **Step 2: Verify it compiles**

```bash
cargo build -p sge-renderer
```

- [ ] **Step 3: Commit**

```bash
git add crates/sge-renderer/src/graph/executor.rs
git commit -m "feat(sge-renderer): add frame graph executor with DrawCommand replay"
```

---

## Task 12: sge-renderer Pipeline, Shader, and cube.wgsl

**Files:**
- Write: `crates/sge-renderer/src/shader.rs`
- Write: `crates/sge-renderer/src/pipeline.rs`
- Create: `assets/shaders/cube.wgsl`

- [ ] **Step 1: Implement `shader.rs`**

```rust
use std::path::Path;

/// Load a WGSL shader from disk and compile it.
/// Panics with the file path on read failure.
pub fn load_wgsl(device: &wgpu::Device, path: &Path) -> wgpu::ShaderModule {
    let source = std::fs::read_to_string(path)
        .unwrap_or_else(|e| panic!("failed to read shader {}: {e}", path.display()));
    device.create_shader_module(wgpu::ShaderModuleDescriptor {
        label:  path.file_name().and_then(|n| n.to_str()),
        source: wgpu::ShaderSource::Wgsl(source.into()),
    })
}
```

- [ ] **Step 2: Implement `pipeline.rs`**

```rust
use std::collections::HashMap;
use std::hash::{DefaultHasher, Hash, Hasher};
use std::sync::Arc;

use crate::buffer::Vertex;

/// Everything needed to create a wgpu RenderPipeline, in hashable form.
pub struct RenderPipelineDesc<'a> {
    pub label:          &'static str,
    pub shader:         &'a wgpu::ShaderModule,
    pub vs_entry:       &'static str,
    pub fs_entry:       &'static str,
    pub bind_layouts:   &'a [&'a wgpu::BindGroupLayout],
    pub color_format:   wgpu::TextureFormat,
    pub depth_format:   Option<wgpu::TextureFormat>,
}

/// Creates render pipelines on first use and caches them by a hash of the descriptor.
/// Use this to avoid redundant pipeline compilation.
pub struct PipelineCache {
    cache: HashMap<u64, Arc<wgpu::RenderPipeline>>,
}

impl PipelineCache {
    pub fn new() -> Self {
        Self { cache: HashMap::new() }
    }

    pub fn get_or_create(
        &mut self,
        device: &wgpu::Device,
        desc:   &RenderPipelineDesc<'_>,
    ) -> Arc<wgpu::RenderPipeline> {
        let key = hash_desc(desc);
        if let Some(p) = self.cache.get(&key) {
            return Arc::clone(p);
        }

        let layout = device.create_pipeline_layout(&wgpu::PipelineLayoutDescriptor {
            label:                Some(desc.label),
            bind_group_layouts:   desc.bind_layouts,
            push_constant_ranges: &[],
        });

        let pipeline = Arc::new(device.create_render_pipeline(&wgpu::RenderPipelineDescriptor {
            label:         Some(desc.label),
            layout:        Some(&layout),
            vertex:        wgpu::VertexState {
                module:      desc.shader,
                entry_point: Some(desc.vs_entry),
                buffers:     &[Vertex::layout()],
                compilation_options: Default::default(),
            },
            fragment:      Some(wgpu::FragmentState {
                module:      desc.shader,
                entry_point: Some(desc.fs_entry),
                targets:     &[Some(wgpu::ColorTargetState {
                    format:     desc.color_format,
                    blend:      Some(wgpu::BlendState::REPLACE),
                    write_mask: wgpu::ColorWrites::ALL,
                })],
                compilation_options: Default::default(),
            }),
            primitive:     wgpu::PrimitiveState {
                topology:  wgpu::PrimitiveTopology::TriangleList,
                cull_mode: Some(wgpu::Face::Back),
                ..Default::default()
            },
            depth_stencil: desc.depth_format.map(|fmt| wgpu::DepthStencilState {
                format:              fmt,
                depth_write_enabled: true,
                depth_compare:       wgpu::CompareFunction::Less,
                stencil:             wgpu::StencilState::default(),
                bias:                wgpu::DepthBiasState::default(),
            }),
            multisample:   wgpu::MultisampleState::default(),
            multiview:     None,
            cache:         None,
        }));

        self.cache.insert(key, Arc::clone(&pipeline));
        pipeline
    }
}

impl Default for PipelineCache {
    fn default() -> Self { Self::new() }
}

fn hash_desc(desc: &RenderPipelineDesc<'_>) -> u64 {
    let mut h = DefaultHasher::new();
    desc.label.hash(&mut h);
    desc.vs_entry.hash(&mut h);
    desc.fs_entry.hash(&mut h);
    desc.color_format.hash(&mut h);
    desc.depth_format.hash(&mut h);
    h.finish()
}
```

- [ ] **Step 3: Create `assets/shaders/cube.wgsl`**

```wgsl
// Uniforms — MVP matrix passed as a 4×4 column-major float array.
struct Uniforms {
    mvp: mat4x4<f32>,
}
@group(0) @binding(0) var<uniform> u: Uniforms;

// Vertex input / output.
struct VIn {
    @location(0) pos: vec3<f32>,
    @location(1) col: vec3<f32>,
}
struct VOut {
    @builtin(position) pos: vec4<f32>,
    @location(0)       col: vec3<f32>,
}

@vertex
fn vs_main(in: VIn) -> VOut {
    return VOut(u.mvp * vec4<f32>(in.pos, 1.0), in.col);
}

@fragment
fn fs_main(in: VOut) -> @location(0) vec4<f32> {
    return vec4<f32>(in.col, 1.0);
}
```

- [ ] **Step 4: Verify renderer compiles**

```bash
cargo build -p sge-renderer
```

- [ ] **Step 5: Commit**

```bash
git add crates/sge-renderer/src/shader.rs crates/sge-renderer/src/pipeline.rs assets/shaders/cube.wgsl
git commit -m "feat(sge-renderer): add PipelineCache, shader loader, and cube.wgsl"
```

---

## Task 13: Spinning Cube Example

**Files:**
- Create: `examples/spinning_cube.rs`
- Modify: `Cargo.toml` (workspace root — add example entry with deps)

- [ ] **Step 1: Add example to workspace `Cargo.toml`**

Append to the workspace root `Cargo.toml`:

```toml
[[example]]
name = "spinning_cube"
path = "examples/spinning_cube.rs"

[dev-dependencies]
sge-core     = { workspace = true }
sge-renderer = { workspace = true }
sge-math     = { workspace = true }
pollster     = { workspace = true }
```

- [ ] **Step 2: Define cube geometry constants**

The cube has 24 vertices (4 per face, unique normals per face) and 36 indices (2 triangles × 6 faces).

```rust
// Place near the top of spinning_cube.rs (after imports).

use sge_renderer::buffer::Vertex;

/// 24 vertices: 4 per face (front, back, left, right, top, bottom).
/// Each face has a solid colour.
const VERTICES: &[Vertex] = &[
    // Front (red)
    Vertex { position: [-0.5, -0.5,  0.5], color: [1.0, 0.2, 0.2] },
    Vertex { position: [ 0.5, -0.5,  0.5], color: [1.0, 0.2, 0.2] },
    Vertex { position: [ 0.5,  0.5,  0.5], color: [1.0, 0.2, 0.2] },
    Vertex { position: [-0.5,  0.5,  0.5], color: [1.0, 0.2, 0.2] },
    // Back (green)
    Vertex { position: [ 0.5, -0.5, -0.5], color: [0.2, 1.0, 0.2] },
    Vertex { position: [-0.5, -0.5, -0.5], color: [0.2, 1.0, 0.2] },
    Vertex { position: [-0.5,  0.5, -0.5], color: [0.2, 1.0, 0.2] },
    Vertex { position: [ 0.5,  0.5, -0.5], color: [0.2, 1.0, 0.2] },
    // Left (blue)
    Vertex { position: [-0.5, -0.5, -0.5], color: [0.2, 0.2, 1.0] },
    Vertex { position: [-0.5, -0.5,  0.5], color: [0.2, 0.2, 1.0] },
    Vertex { position: [-0.5,  0.5,  0.5], color: [0.2, 0.2, 1.0] },
    Vertex { position: [-0.5,  0.5, -0.5], color: [0.2, 0.2, 1.0] },
    // Right (yellow)
    Vertex { position: [ 0.5, -0.5,  0.5], color: [1.0, 1.0, 0.2] },
    Vertex { position: [ 0.5, -0.5, -0.5], color: [1.0, 1.0, 0.2] },
    Vertex { position: [ 0.5,  0.5, -0.5], color: [1.0, 1.0, 0.2] },
    Vertex { position: [ 0.5,  0.5,  0.5], color: [1.0, 1.0, 0.2] },
    // Top (cyan)
    Vertex { position: [-0.5,  0.5,  0.5], color: [0.2, 1.0, 1.0] },
    Vertex { position: [ 0.5,  0.5,  0.5], color: [0.2, 1.0, 1.0] },
    Vertex { position: [ 0.5,  0.5, -0.5], color: [0.2, 1.0, 1.0] },
    Vertex { position: [-0.5,  0.5, -0.5], color: [0.2, 1.0, 1.0] },
    // Bottom (magenta)
    Vertex { position: [-0.5, -0.5, -0.5], color: [1.0, 0.2, 1.0] },
    Vertex { position: [ 0.5, -0.5, -0.5], color: [1.0, 0.2, 1.0] },
    Vertex { position: [ 0.5, -0.5,  0.5], color: [1.0, 0.2, 1.0] },
    Vertex { position: [-0.5, -0.5,  0.5], color: [1.0, 0.2, 1.0] },
];

/// 36 indices — 2 triangles per face, CCW winding.
const INDICES: &[u16] = &[
     0,  1,  2,   0,  2,  3,  // front
     4,  5,  6,   4,  6,  7,  // back
     8,  9, 10,   8, 10, 11,  // left
    12, 13, 14,  12, 14, 15,  // right
    16, 17, 18,  16, 18, 19,  // top
    20, 21, 22,  20, 22, 23,  // bottom
];
```

- [ ] **Step 3: Write the full example**

```rust
// examples/spinning_cube.rs

use std::f32::consts::TAU;
use std::sync::Arc;

use sge_core::{App, AppState, FrameContext, WindowConfig};
use sge_math::camera::PerspectiveCamera;
use sge_renderer::{
    GpuContext,
    buffer::{IndexBuffer, UniformBuffer, Vertex, VertexBuffer},
    graph::{FrameGraph, ResourcePool},
    graph::resource::TextureDesc,
    pipeline::{PipelineCache, RenderPipelineDesc},
    shader::load_wgsl,
};
use wgpu::{Extent3d, TextureFormat, TextureUsages};

// … paste VERTICES and INDICES constants here …

struct CubeDemo {
    gpu:         GpuContext,
    vertex_buf:  VertexBuffer,
    index_buf:   IndexBuffer,
    mvp_uniform: UniformBuffer<[f32; 16]>,
    pipeline:    Arc<wgpu::RenderPipeline>,
    bind_group:  Arc<wgpu::BindGroup>,
    pool:        ResourcePool,
    camera:      PerspectiveCamera,
    depth_desc:  TextureDesc,
}

impl AppState for CubeDemo {
    fn init(window: Arc<winit::window::Window>) -> Self {
        let gpu = pollster::block_on(GpuContext::new(Arc::clone(&window)));

        let vertex_buf  = VertexBuffer::new(&gpu.device, VERTICES);
        let index_buf   = IndexBuffer::new(&gpu.device, INDICES);
        let mvp_uniform = UniformBuffer::<[f32; 16]>::new(&gpu.device, "MVP");

        // Bind group layout: single uniform buffer at binding 0.
        let bgl = gpu.device.create_bind_group_layout(&wgpu::BindGroupLayoutDescriptor {
            label:   Some("mvp_bgl"),
            entries: &[wgpu::BindGroupLayoutEntry {
                binding:    0,
                visibility: wgpu::ShaderStages::VERTEX,
                ty:         wgpu::BindingType::Buffer {
                    ty:                 wgpu::BufferBindingType::Uniform,
                    has_dynamic_offset: false,
                    min_binding_size:   None,
                },
                count: None,
            }],
        });

        let bind_group = Arc::new(gpu.device.create_bind_group(&wgpu::BindGroupDescriptor {
            label:   Some("mvp_bg"),
            layout:  &bgl,
            entries: &[wgpu::BindGroupEntry {
                binding:  0,
                resource: mvp_uniform.buffer.as_entire_binding(),
            }],
        }));

        let shader = load_wgsl(
            &gpu.device,
            std::path::Path::new("assets/shaders/cube.wgsl"),
        );

        let size    = window.inner_size();
        let aspect  = size.width as f32 / size.height as f32;
        let camera  = PerspectiveCamera {
            fov_y:  45_f32.to_radians(),
            aspect,
            near:   0.1,
            far:    100.0,
        };

        let depth_format = TextureFormat::Depth32Float;
        let depth_desc = TextureDesc::new(
            Extent3d { width: size.width, height: size.height, depth_or_array_layers: 1 },
            depth_format,
            TextureUsages::RENDER_ATTACHMENT,
        ).with_label("depth");

        let mut pipeline_cache = PipelineCache::new();
        let pipeline = pipeline_cache.get_or_create(&gpu.device, &RenderPipelineDesc {
            label:        "cube",
            shader:       &shader,
            vs_entry:     "vs_main",
            fs_entry:     "fs_main",
            bind_layouts: &[&bgl],
            color_format: gpu.surface_format,
            depth_format: Some(depth_format),
        });

        Self {
            gpu, vertex_buf, index_buf, mvp_uniform, pipeline, bind_group,
            pool: ResourcePool::new(), camera, depth_desc,
        }
    }

    fn on_frame(&mut self, ctx: &FrameContext<'_>) {
        // Compute rotation from elapsed time (90°/s).
        let angle = ctx.time.elapsed.as_secs_f32() * TAU / 4.0;
        let model = glam::Mat4::from_rotation_y(angle);
        let eye   = glam::Vec3::new(0.0, 1.5, 3.0);
        let view  = PerspectiveCamera::view(eye, glam::Vec3::ZERO, glam::Vec3::Y);
        let proj  = self.camera.projection();
        let mvp   = proj * view * model;
        self.mvp_uniform.write(&self.gpu.queue, &mvp.to_cols_array());

        let Some(frame) = self.gpu.current_texture() else { return };
        let surface_view = Arc::new(
            frame.texture.create_view(&wgpu::TextureViewDescriptor::default())
        );

        // Build frame graph.
        let mut graph  = FrameGraph::new();
        let backbuffer = graph.import_texture(Arc::clone(&surface_view));
        let depth      = graph.create_texture("depth", self.depth_desc.clone());

        let pipeline   = Arc::clone(&self.pipeline);
        let vb         = Arc::clone(&self.vertex_buf.buffer);
        let ib         = Arc::clone(&self.index_buf.buffer);
        let bg         = Arc::clone(&self.bind_group);
        let idx_count  = self.index_buf.count;

        graph.add_pass("geometry", |b| {
            b.write_color(backbuffer, wgpu::Color { r: 0.08, g: 0.08, b: 0.10, a: 1.0 });
            b.write_depth(depth, 1.0);
            b.set_pipeline(pipeline);
            b.set_vertex_buffer(0, vb);
            b.set_index_buffer(ib, wgpu::IndexFormat::Uint16);
            b.set_bind_group(0, bg);
            b.draw_indexed(0..idx_count, 0, 0..1);
        });

        // Compile + execute.
        let mut imported = std::collections::HashMap::new();
        imported.insert(backbuffer, surface_view);

        graph.compile().execute(
            &self.gpu.device,
            &self.gpu.queue,
            &mut self.pool,
            &imported,
        );

        // Pre-present notification (frame pacing hint on some platforms).
        ctx.window.pre_present_notify();
        frame.present();
    }

    fn on_resize(&mut self, width: u32, height: u32) {
        self.gpu.resize(width, height);
        self.camera.aspect = width as f32 / height as f32;
        self.depth_desc.size = Extent3d { width, height, depth_or_array_layers: 1 };
    }
}

fn main() {
    App::new()
        .with_window(WindowConfig { title: "SGE — Spinning Cube", width: 1280, height: 720, vsync: true })
        .with_fixed_timestep(std::time::Duration::from_secs_f64(1.0 / 60.0))
        .run::<CubeDemo>();
}
```

- [ ] **Step 4: Add `glam` and `winit` to dev-dependencies in workspace `Cargo.toml`**

```toml
[dev-dependencies]
sge-core     = { workspace = true }
sge-renderer = { workspace = true }
sge-math     = { workspace = true }
pollster     = { workspace = true }
glam         = { workspace = true }
winit        = { workspace = true }
```

- [ ] **Step 5: Build the example**

```bash
cargo build --example spinning_cube
```

Expected: compiles with no errors.

- [ ] **Step 6: Commit**

```bash
git add examples/spinning_cube.rs Cargo.toml
git commit -m "feat: add spinning_cube example wiring sge-core + sge-renderer + sge-math"
```

---

## Task 14: Final Verification

- [ ] **Step 1: Full workspace build, no warnings**

```bash
cargo build --workspace --examples
```

Expected: zero errors, zero warnings.

- [ ] **Step 2: Clippy clean**

```bash
cargo clippy --workspace --examples -- -D warnings
```

Expected: zero lints.

- [ ] **Step 3: All unit tests pass**

```bash
cargo test --workspace
```

Expected: all tests pass (sge-math: 3, sge-core: 4, sge-renderer: 8+).

- [ ] **Step 4: Run the example**

```bash
cargo run --example spinning_cube
```

Expected:
- A 1280×720 window opens
- A coloured cube rotates smoothly around the Y axis
- Each face is a distinct solid colour (red, green, blue, yellow, cyan, magenta)
- No flickering or visual artefacts

- [ ] **Step 5: Verify resize**

While the window is open, drag to resize it. Expected: cube continues rendering correctly at the new dimensions, no panic.

- [ ] **Step 6: Verify clean exit**

Close the window. Expected: process exits with code 0, no panic output.

- [ ] **Step 7: Final commit**

```bash
git add -A
git commit -m "feat: Phase I + II complete — spinning cube via frame graph renderer"
```

---

## Spec Coverage Summary

| Spec Requirement | Covered In |
|---|---|
| Cargo workspace, 4 crates | Task 1 |
| sge-math: Vec3/Mat4/Quat, Transform, PerspectiveCamera | Task 2 |
| sge-ecs stub | Task 3 |
| Fixed-timestep main loop | Task 4 |
| AppState trait, winit 0.30 ApplicationHandler | Task 5 |
| GpuContext (wgpu device/queue/surface, sRGB, resize, lost surface) | Task 6 |
| Vertex, VertexBuffer, IndexBuffer, UniformBuffer | Task 7 |
| ResourceHandle, TextureDesc (label excluded from hash), ResourcePool | Task 8 |
| FrameGraph, PassBuilder, DrawCommand (Arc-based, no closures) | Task 9 |
| Frame graph compiler: topo sort, dead-pass culling, lifetimes | Task 10 |
| Frame graph executor: single encoder per frame, Arc deref pattern | Task 11 |
| PipelineCache, load_wgsl, cube.wgsl shader | Task 12 |
| Spinning cube example: MVP transform, depth buffer, resize, present | Task 13 |
| Verification: build + clippy + tests + runtime checks | Task 14 |
