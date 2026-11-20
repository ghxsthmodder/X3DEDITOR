# X3D Editor

**3D modeling editor for Android, with a C++ backend and Vulkan rendering.**

X3D Editor is a project in the specification and planning phase for version 1.0. Its goal is to provide polygon modeling tools, scene organization, model import/export, and a native binary format, prioritizing performance, stability, and efficient memory usage on mobile devices.

## Main specification

The master technical specification consolidates the functional scope, architecture, rendering requirements, compatibility, risks, and development stages:

- **[X3D Editor 1.0 — Master Technical Specification and Development Roadmap](docs/specification-v1.0.md)**

This document is the scope reference for version 1.0. It does not mean that the listed features have already been implemented.

## Platform and architecture

- **Platform:** Android only, landscape orientation.
- **UI:** Java and/or Kotlin; the final choice will be made during build preparation.
- **Backend:** C++.
- **Integration:** JNI.
- **Rendering:** Vulkan 1.1.
- **Shaders:** GLSL source and compiled SPIR-V.
- **Import/export:** Assimp.
- **Editable topology:** Half-Edge Mesh.
- **Native format:** X3D binary, `.bin` extension.

## Version 1.0 scope at a glance

- Project manager and 3D viewport.
- Polygon modeling: Extrude, Inset, Bevel, Loop Cut, Merge, Subdivide, Delete, and Move Pivot.
- Object, vertex, edge, and face selection.
- Transforms, Snap, and axis constraints.
- Scene Graph with meshes, Empty nodes, cameras, and lights.
- Modifier Stack with Mirror.
- Undo/Redo, autosave, and recovery.
- glTF/GLB, FBX, OBJ, and DAE import/export.
- Diagnostics, logging, and local Crash Reports.

Texturing, UV editing, advanced PBR materials, animation, rigging, Bevel Modifier, and Subdivision Surface are deferred to future versions. See the master specification for the complete scope and exclusions.

## Technical documentation

- [Master technical specification and 1.0 roadmap](docs/specification-v1.0.md)
- [Technology architecture](docs/architecture.en.md)
- [X3D v1 internal binary format](docs/data-format.md)
- [X3D v1 format conformance tests](docs/format-v1-validation.md)

**Format note:** the existing v1 binary specification remains the source of truth for the current byte layout. The master specification records differences between the current schema and the approved functional requirements; any evolution must be versioned and accompanied by fixtures and tests.

## Development stages

0. Build preparation and decisions.
1. Android application skeleton and JNI.
2. Core, Logger, Device, and initial diagnostics.
3. Vulkan foundation and minimal viewport.
4. Scene Graph and transforms.
5. Geometry and topology.
6. Selection, tools, and Command History.
7. Modifier Stack and Mirror.
8. Projects and X3D format.
9. Import and export.
10. Complete interface and workflow.
11. Graphics quality and optimization.
12. Integration testing, stability, and 1.0 release.

Deliverables and acceptance criteria for each stage are described in the [master specification](docs/specification-v1.0.md).

## Status

**Planning — version 1.0 functional scope consolidated; implementation and technical validation pending.**

Implementation status must only be updated when code and tests provide evidence that the acceptance criteria have been met.
