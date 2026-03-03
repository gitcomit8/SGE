# Serix Game Engine (SGE)

## Introduction
The Serix Game Engine (SGE) is a high-performance, modular game engine developed in the Rust programming language. Designed with a focus on a voxel-based cyber-simulation, SGE prioritizes efficient data management, modularity, and modern rendering techniques.

## Technical Overview
SGE is built on a modular architecture using a Cargo Workspace to ensure a clean separation of concerns and maintainability.

### Core Architecture (Modular Crates)
*   **`sge-core`**: Manages the application lifecycle, including the main execution loop, timing logic, and windowing integration (via `winit`).
*   **`sge-renderer`**: Handles the GPU interface, shader management, pipeline state, and the implementation of a custom Render Graph.
*   **`sge-ecs`**: Facilitates state management through a custom-built entity-component-system, focused on efficient entity storage and querying.
*   **`sge-math`**: Provides custom linear algebra wrappers, leveraging `glam` for high-performance mathematical operations.

### Graphics & Rendering Strategy
SGE utilizes `wgpu` as its primary graphics abstraction layer, providing cross-platform support for Vulkan, DirectX 12, and Metal. The engine implements a **Bindless Renderer** and a custom **Render Graph** abstraction to optimize GPU utilization and simplify complex rendering pipelines.

## Feature Roadmap (MoSCoW)

| Priority | Feature | Description |
| :--- | :--- | :--- |
| **Must Have** | Sparse-Set ECS | Core state management using sparse-set architecture for O(1) access. |
| **Must Have** | Bindless Renderer | Modern rendering architecture to reduce CPU overhead and draw call limitations. |
| **Must Have** | Virtual File System | Unified interface for asset management and data abstraction. |
| **Should Have** | glTF 2.0 Asset Loader | Standardized 3D asset ingestion and processing. |
| **Should Have** | Job System | Multi-threaded task scheduling for optimized CPU utilization. |
| **Could Have** | Physically Based Rendering (PBR) | Advanced lighting and material simulation. |
| **Future** | Hot-Reloading Scripting | Dynamic logic updates without requiring full engine recompilation. |

## Target Application: Voxel-based Cyber-Simulation
The engine is specifically optimized for large-scale voxel environments, requiring:
*   **Dynamic Mesh Generation**: Real-time generation of geometry from voxel data.
*   **Massive Data Management**: Efficient handling of large-scale simulation states.
*   **Visibility Culling**: Advanced algorithms to ensure only relevant geometry is processed by the GPU.

## Development Blueprint

| Phase | Component | Internal Implementation (Custom) | External Dependencies |
| :--- | :--- | :--- | :--- |
| **I: Foundation** | The Heartbeat | Main Loop with fixed time-step logic. | `winit` (Windowing) |
| **II: Graphics** | Bindless Renderer | Render Graph design & Shader Pipeline. | `wgpu` (GPU Abstraction) |
| **III: Data** | Serix-ECS | Sparse-Set ECS implementation from scratch. | *None (Native Rust)* |
| **IV: Memory** | Arena Allocators | Custom memory management for frame-local data. | *None (Native Rust)* |
| **V: Geometry** | Voxel Mesher | Greedy Meshing algorithms implementation. | `glam` (SIMD Math) |

## Technology Stack
*   **Language**: Rust (utilizing Stable, with Nightly features for SIMD optimizations where necessary).
*   **Graphics Backend**: `wgpu` (targeting Vulkan and DX12).
*   **Mathematics**: `glam` (primary backend) with custom Coordinate Space wrappers.
*   **Structure**: Cargo Workspace for multi-crate management.
