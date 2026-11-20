# X3D Editor 1.0 — Master Technical Specification and Development Roadmap

> **Status:** Functional scope consolidated; implementation has not been started or is not verified by this document.  
> **Platform:** Android, landscape orientation.  
> **Native format:** X3D, `.bin` extension.  
> **Graphics API:** Vulkan 1.1.  
> **Last revised:** 2026-11-20.

This document consolidates the decisions approved for version 1.0 and defines a development sequence. It is the product scope reference; it does not replace detailed subsystem specifications or claim that any feature has been implemented.

## 1. Purpose and principles

X3D Editor is a 3D modeling editor exclusively for Android. Version 1.0 aims to provide a reliable workflow for creating, editing, importing, exporting, and storing scenes and meshes.

Guiding principles:

- Prioritize stability, performance, and predictable behavior on mobile devices.
- Keep the interface responsive; heavy operations must not run on the main thread.
- Separate the Android UI, editor logic, scene data, and rendering.
- Avoid unnecessary full copies and allocations of geometry data.
- Preserve existing projects during saves and failures.
- Build a solid modeling foundation first; texturing and animation features are outside the 1.0 scope.
- Explicitly handle limitations of external formats. Do not promise lossless conversion when the target format cannot represent the source data.

## 2. Version 1.0 functional scope

### 2.1 Platform and interface

- Android only, in landscape orientation.
- Material Design interface; the specific UI library version will be selected during implementation.
- Dark theme by default; light and system themes are also available.
- Languages: Portuguese (Brazil), English, and Spanish, switchable without restarting.
- Font sizes: Small, Medium (default), and Large.
- App-private storage as the default project location.
- File import and export through the Android Storage Access Framework (SAF), without requiring broad storage permissions.

### 2.2 Project manager

The home screen has a fixed sidebar and a project grid. Each card displays a thumbnail, project name, and last-modified date/time, plus a context menu for renaming or deleting. Opening a project is done by tapping its card.

- New Project opens a dialog with a required project-name field.
- Create opens the modeling screen directly.
- Delete requires confirmation and removes the project's associated files.
- Thumbnails are pre-generated images stored in the project folder and loaded on demand.
- Projects without thumbnails use a default image.
- The project list must not load geometry or deserialize X3D files just to display cards.

Settings include Appearance, Language, Storage, and About. Storage displays project count, occupied space, and cache/thumbnail usage; clearing the cache must not remove projects or original files. About displays app and format versions, libraries, licenses, and repository information.

### 2.3 Modeling screen

The viewport is the central area. The top bar contains navigation back to the project manager, Save, Undo, Redo, and Settings. Leaving with unsaved changes requires confirmation.

The left sidebar contains the Modeling and Add categories. Side panels are collapsible; the hierarchy panel opens as an overlay from the right. The bottom bar is contextual and contains Position, Rotation, and Scale. There is no visual history panel in version 1.0.

**Add:** primitives (including cube, sphere, and cylinder), Empty/position node, camera, light, and model import.

**Modeling:** exclusive modes Object (default), Vertex, Edge, and Face. Tools:
- Geometry: Extrude, Inset, and Bevel.
- Topology: Loop Cut, Merge, and Subdivide.
- Other: Delete and Move Pivot.

Tools expose contextual parameters and temporary previews where applicable. Confirm applies the operation; Cancel discards the preview without modifying the original geometry.

### 2.4 Selection

- Single selection and multi-selection through a dedicated control.
- Box selection: dragging left-to-right selects only fully enclosed elements; dragging right-to-left selects enclosed or intersected elements.
- Box selection respects the active mode and can be combined with multi-selection.
- Tapping empty space clears the selection.
- Hierarchy and viewport selection remain synchronized.
- Selected objects use an orange outline; hover/focus uses blue; components have their own highlight.
- The active object must be visually distinguishable from other items in a multi-selection.

### 2.5 Transformations and units

- Visual manipulators and numeric fields remain synchronized in real time.
- Position, Rotation, and Scale are in the bottom bar.
- Global and Local coordinates; Global is the default.
- Position uses world units, rotation is displayed in degrees, and scale is edited per axis.
- Uniform scale lock.
- Snap can be enabled or disabled. Initial increments: position 0.1 m, rotation 15°, and scale 0.1; increments are configurable.
- Axis constraints X/Y/Z, plane constraints XY/XZ/YZ, or free movement.
- Configurable display precision: 2, 3, 4, or 6 decimal places; default is 3.
- Internal position, rotation, and scale values use float32/quaternions according to the architecture. Changing units affects display only, not geometry.
- Display units: meters, centimeters, and millimeters.

Move Pivot is distinct from Empty: it changes the object's origin/pivot without visually moving its geometry. Modes: move freely, center on geometry, and world origin.

### 2.6 Mesh operations

All eight tools are included in version 1.0:

| Tool | Minimum behavior |
|---|---|
| Extrude | Extrude selected faces/regions; distance and extrusion mode |
| Inset | Create an inner boundary; thickness and individual inset option |
| Bevel | Chamfer edges/vertices; width, segments, and angle limit |
| Loop Cut | Insert cuts following topology; count and placement |
| Merge | Merge selected vertices; center, last selected, or distance |
| Subdivide | Split edges/faces; subdivision count |
| Delete | Remove objects or components according to selection mode |
| Move Pivot | Change the pivot without visually moving geometry |

Confirmed operations are integrated with Undo/Redo. The implementation must validate topology and prevent dangling references or invalid data.

### 2.7 Modifiers

The Modifier Stack is included in version 1.0, with **Mirror** as the only initial modifier.

Mirror:
- X, Y, and Z axes.
- Non-destructive preview.
- Updates when the source geometry changes.
- Enable/disable and apply permanently.
- Clipping at the center line and welding of vertices near the symmetry plane.
- Editable stack order, ready for future modifiers.

Bevel Modifier and Subdivision Surface are deferred to future versions. The destructive Bevel tool remains included in version 1.0.

### 2.8 Scene and hierarchy

The scene uses a hierarchical Scene Graph. Nodes have a unique identifier, name, local transform, parent, children, and optional components.

Planned components: Mesh, Camera, Light, and Empty. Empty has no geometry and can represent references such as attachment points. Global transforms are derived from the hierarchy.

- Persistent node IDs: uint64.
- Internal references use IDs, never names.
- Local transform: position float32×3, quaternion float32×4 (XYZW), and scale float32×3.
- Global matrices are calculated by the engine when needed.
- Transform Dirty Flags propagate invalidation to descendants and avoid unnecessary recomputation.
- Right-handed coordinate system, Y-up, negative Z forward.
- The UI displays angles in degrees; the engine uses radians where working with angles and quaternions internally.

### 2.9 Cameras and navigation

The viewport navigation camera is independent of scene cameras and uses an Orbit Camera model with Target, distance, horizontal/vertical rotation, zoom, and pan.

Default gestures:
- One finger: orbit.
- Two-finger pinch: zoom.
- Two fingers moving together: pan.

Quick controls: frame all, frame selected, and restore camera. Perspective and Orthographic projections. Rotation, zoom, and pan sensitivity are configurable; minimum/maximum distance and vertical rotation limits are configurable.

Scene cameras: multiple cameras; create, rename, delete, set active, view through camera, adjust FOV/perspective, and set Near/Far planes.

### 2.10 Lights, viewport, and presentation

Scene lights: Point, Directional, and Spot, with color, intensity, transform, and type-specific parameters. Gizmos remain visible even when a light is disabled. Viewport auxiliary lighting is independent of the scene, enabled by default in Solid mode, is not a scene object, and is not saved in the project.

Three-point auxiliary lighting: Key, Fill, and Rim. Viewport modes: Solid, Wireframe, Wireframe + Solid, and Flat Shading. Background options: uniform dark gray, dark gradient, or 3D grid with horizon/axes; grid and axes can be hidden independently.

Graphics features:
- Real-time shadows via Shadow Mapping. CSM for Directional lights; conventional shadow maps for Point and Spot lights.
- SSAO.
- MSAA: disabled, 2x, or 4x; 4x is the desired default, subject to device support.
- FXAA as a post-processing option.
- Tone Mapping and Gamma Correction.
- Low, Medium, High, and Custom profiles; effects can be configured individually.
- Initial Vulkan capability detection and profile recommendation. Users can change the profile later.

Shadow quality options: Disabled, Low, Medium, and High; Medium is the proposed default. Implementation must respect the limits and formats actually supported by the device and provide safe fallbacks.

### 2.11 Materials and textures

Version 1.0 includes basic materials for viewport display only:
- Base color per object/material.
- Default material.
- Basic lighting parameters.
- Basic opacity.
- Color display in Solid mode.

There is no advanced PBR editor, texturing workflow, or texture loading/editing. The data structures should allow future evolution without exposing unimplemented features.

### 2.12 Import and export

Priority formats: glTF/GLB, FBX, OBJ, and DAE (COLLADA), using Assimp as the primary library.

Workflow: **external model → importer/normalization → X3D scene → exporter → external model**.

Rules:
- Import the entire scene or selected elements where feasible.
- Preserve hierarchy, transforms, and geometry whenever the format allows it.
- Convert coordinates and units when necessary.
- Triangulate geometry when required by the internal representation.
- Warn about attributes, components, or materials that cannot be preserved.
- Validate each exporter and the capabilities of the integrated Assimp version individually.
- Show progress and allow safe cancellation of long-running operations.

Do not promise lossless round-tripping between formats with different capabilities.

### 2.13 Saving, projects, and recovery

Manual saving and configurable autosave. Autosave options: disabled, 1, 3, 5, 10, or 15 minutes. Show an unsaved-changes indicator.

Saving must preserve the previous file until the new file has been fully written: create a unique temporary file in the same directory, write all data, flush/synchronize, close, and atomically rename when supported. Failures must not truncate the previous valid project.

Local recovery after an unexpected shutdown; offer the user the option to recover the discovered version. The full Undo/Redo history does not need to be persisted in the main project file.

### 2.14 Performance

There is no artificial fixed vertex or face limit. Practical limits are determined by memory, device capability, and stability. The editor must:
- Monitor memory pressure and warn before risky operations.
- Update GPU buffers incrementally.
- Avoid unnecessary full copies.
- Run long operations outside the UI and render threads.
- Release resources when objects are removed or projects are closed.
- Implement Frustum Culling.
- Limit concurrent operations on the same mesh.
- Defer complex LOD systems and geometry streaming beyond version 1.0.

## 3. Technical architecture

### 3.1 Layers and modules

**Android / Java or Kotlin**
- UI, menus, dialogs, settings, lifecycle, and SAF.
- The choice between Java, Kotlin, or both remains open for the implementation phase; JNI is required for communication with the native backend.

**C++ backend**
- `Core`: lifecycle, shared services, and coordination.
- `Device`: GPU identification, Vulkan properties/limits, extensions, formats, device selection, and diagnostics.
- `Graphics`: Vulkan, rendering, resources, and shaders.
- `Geometry`: mesh data, Half-Edge Mesh, and modeling operations.
- `Scene`: Scene Graph, transforms, cameras, and lights.
- `ImportExport`: Assimp integration and conversion.
- `Project`: X3D reading, writing, versioning, and integrity checks.
- `Editor`: selection, tools, and commands.
- `Logger`: centralized logging.
- `Diagnostics`: diagnostics and local Crash Reports.

### 3.2 Vulkan and rendering pipeline

- Minimum API: Vulkan 1.1.
- Shader source: GLSL; compiled/distributed format: SPIR-V (`.spv`).
- Dedicated rendering thread.
- Logical render passes/stages: Geometry, Lighting, Grid, Gizmo, and Selection. Implementation may use subpasses or separate passes depending on compatibility and measured cost.
- Centralized Vulkan resource manager: buffers, images, shaders, pipelines, descriptor sets, and memory.
- Editable CPU geometry is separate from GPU rendering buffers.
- Data: position float32×3, normal float32×3, UV float32×2 (future-ready), and uint32 indices.
- Editable topology: Half-Edge Mesh. The rendering representation may be triangulated and derived from the topology.
- Right-handed coordinate system, Y-up, negative Z forward.

The ID Buffer uses 32-bit integer identifiers, preferably `VK_FORMAT_R32_UINT`, after support is verified. Persistent scene IDs remain uint64; picking IDs are temporary and mapped to persistent IDs. Object, vertex, edge, and face selection use GPU Picking. The picking pass must be independent of MSAA when needed to guarantee deterministic integer ID reads; readback must avoid blocking the render thread.

Depth: prefer `VK_FORMAT_D24_UNORM_S8_UINT`; use `VK_FORMAT_D32_SFLOAT` as an alternative if supported. MSAA 4x is the preferred default but must be negotiated against device capabilities. Do not assume every format, sample count, or optional feature is available on every Vulkan 1.1 GPU.

### 3.3 Rendering pipeline and effects

Logical stages:
1. Geometry Pass.
2. Lighting Pass.
3. Grid Pass.
4. Gizmo Pass.
5. Selection Pass.

Shadows: Shadow Mapping; CSM for directional lights and conventional maps for Point/Spot lights. SSAO, FXAA, Tone Mapping, and Gamma Correction are planned features. Effects must be individually switchable and grouped into Low/Medium/High/Custom profiles. The exact order and combination of MSAA and FXAA must be validated for visual quality and performance during prototyping.

### 3.4 Threads and synchronization

- Android Main Thread: UI and input events.
- Render Thread: graphics commands and Vulkan rendering.
- C++ Thread Pool: import/export, geometry operations, and heavy calculations.
- JNI: batched commands and result transfer; avoid excessively frequent calls.

The Thread Pool adapts its worker count to device resources; do not create a new thread for every operation. Operations should be cancellable when safe. A mesh must not be modified simultaneously by incompatible tasks. Results are published/synchronized with the Render Thread through an explicit queue/event mechanism.

The Android lifecycle must handle pause, resume, surface recreation, and Vulkan device loss.

### 3.5 Memory

- RAII for C++ resource ownership and release.
- Memory Pool for frequent allocations.
- Arena Allocator for temporary operation data.
- Dedicated GPU Allocator for Vulkan memory.

Strategies must be measured before premature optimization. Ownership, lifetime, and synchronization must be explicit.

### 3.6 Events, commands, and history

A centralized Event System, initially including: `ObjectSelected`, `ObjectCreated`, `ObjectDeleted`, `GeometryModified`, `TransformChanged`, `ProjectSaved`, and `ProjectLoaded`. It must be thread-safe and manage subscriber lifetimes.

The Command System represents operations such as `CreateObjectCommand`, `DeleteObjectCommand`, `TransformObjectCommand`, `ExtrudeCommand`, `BevelCommand`, and `MirrorCommand`. Commands should support execution and reversal where applicable.

Undo/Redo uses Command History rather than full mesh copies per operation. History is per project, has a configurable memory limit, and discards older commands when necessary. Commands may retain the geometry data required for exact reversal; do not rely only on mathematical inverse operations.

### 3.7 Logging, errors, and diagnostics

A shared Logger with TRACE, DEBUG, INFO, WARN, ERROR, and FATAL levels; timestamps, source module, local log files, rotation, and storage limits.

Vulkan error levels:
- Warning: a feature/resource is unavailable; continue with a fallback.
- Error: an operation failed; keep the project open when safe.
- Fatal: a critical failure requiring the graphics engine to stop.

Settings include a Diagnostics page showing GPU/vendor/model, Vulkan version, driver, reported memory information, active graphics profile, recent warnings/errors, and log export.

Local Crash Reports include the reason, responsible module when identifiable, native C++ stack trace, Android device data, GPU/driver/Vulkan information, and recent Logger events. Reports are not sent automatically.

## 4. Features outside version 1.0

Planned for future versions, with no release-date commitment:
- Texturing and UV editing.
- Advanced PBR material editor and texture loading.
- Bevel Modifier.
- Subdivision Surface.
- Animation and rigging.
- Physics/simulation.
- Sculpting.
- Automatic LOD and advanced geometry streaming.
- Collaborative editing.

The xatlas library has been considered for UV unwrapping, but UV generation/editing and the texturing workflow are not part of the version 1.0 functional scope. Integration may be revisited in a future version.

## 5. Compatibility and device requirements

- Minimum graphics requirement: Vulkan 1.1 support.
- Devices without Vulkan 1.1 are not compatible with the version 1.0 3D viewport.
- Detect GPU, driver, extensions, formats, limits, and supported sample counts.
- Recommend an initial graphics profile; allow manual changes.
- Test primarily on Adreno and Mali GPUs, as well as other available Vulkan devices.
- Do not assume optional features are available solely because of the API version.
- Minimum Android version, ABIs, NDK, SDK, and memory requirements must be defined before finalizing the build configuration.

## 6. X3D format compatibility and evolution

The repository already contains an independent specification for the `.bin` binary format in [data-format.md](data-format.md), defining version 1, the header, META/NODE/MESH/MATL/TEXR sections, and tests in [format-v1-validation.md](format-v1-validation.md).

**That existing specification remains the source of truth for version 1 bytes until a formal revision is approved.** This master document does not retroactively change the schema.

The following differences must be resolved before freezing the format used by the application:
- The current v1 schema represents node references using collection indices, while the approved architecture calls for persistent uint64 IDs.
- The current v1 schema does not explicitly describe Camera and Light components or the modifier stack.
- The persisted v1 geometry is triangulated (positions/normals/UV0/indices); Half-Edge is the editable in-memory topology and still needs a serialization decision.
- MATL/TEXR already define material data and texture references, although texturing is outside the version 1.0 functional scope.
- The v1 schema does not include submeshes or multiple materials per mesh.

**Required action during the format phase:** create a compatibility matrix between scene requirements and schema v1; decide whether the requirements can be represented through a compatible extension or require a new format version. Do not change the canonical fixture or v1 tests without explicit versioning and a documented decision. Readers must reject unsupported versions in a controlled manner and preserve the possibility of future migration.

## 7. Development stages

The stages below are ordered by dependency. Some tasks may run in parallel after module contracts are stable. Each stage ends with verifiable acceptance criteria; do not advance merely because the code compiles.

### Stage 0 — Build preparation and decisions

**Deliverables**
- Confirm minimum Android version, SDK/NDK, ABI(s), C++ toolchain, and UI language (Java, Kotlin, or both).
- Define directory structure and coding conventions.
- Configure the Android build and CMake/NDK integration.
- Define dependency policy and Assimp versions.
- Create basic CI and build documentation.

**Acceptance criteria**
- An empty Android app installs and starts on a compatible device.
- The native library builds for every selected ABI.
- Dependencies and versions are pinned and reproducible.

### Stage 1 — Android application skeleton and JNI

**Deliverables**
- Main Activity in landscape.
- Basic navigation and dark theme.
- C++ library loaded through JNI.
- Initial command/error contract between UI and backend.
- Android lifecycle handling.

**Acceptance criteria**
- The UI does not block when calling test native operations.
- JNI handles errors without crashing.
- Pause/resume does not leave orphaned threads or resources.

### Stage 2 — Core, Logger, Device, and initial diagnostics

**Deliverables**
- Core, Logger, and Device modules.
- Vulkan property collection and GPU/driver identification.
- Warning/Error/Fatal handling.
- Local logs with rotation.
- First diagnostics screen.

**Acceptance criteria**
- A device without Vulkan 1.1 receives a clear message and does not attempt to start an incompatible viewport.
- Device properties and limitations are recorded.
- Simulated errors are classified correctly.

### Stage 3 — Vulkan foundation and minimal viewport

**Deliverables**
- Instance, logical device, queues, swapchain, and synchronization.
- Dedicated Render Thread.
- Surface management and recreation across lifecycle events.
- SPIR-V loading.
- Minimal Geometry Pass and rendering of one primitive.
- Vulkan resource manager and Depth Buffer.

**Acceptance criteria**
- Continuous rendering without blocking the UI.
- Surface resize/recreation works.
- Resources are released without known leaks.
- Format/sample fallbacks are tested.

### Stage 4 — Scene Graph and transforms

**Deliverables**
- Nodes, IDs, components, and hierarchy.
- Local and global transforms using quaternions.
- Dirty Flags.
- Orbit camera, Perspective/Orthographic projection, and XYZ gizmo.
- Basic scene cameras and lights.

**Acceptance criteria**
- Deep hierarchies and parent changes update matrices correctly.
- No cycles or invalid references are allowed.
- Orbit, zoom, and pan gestures work on a touch screen.

### Stage 5 — Geometry and topology

**Deliverables**
- Separate CPU Mesh data and GPU buffers.
- Half-Edge Mesh.
- Conversion from topology to triangulated buffers.
- Normals, indices, and attributes.
- Incremental buffer updates.

**Acceptance criteria**
- Create, edit, and render test meshes.
- Validate connectivity and topology invariants after operations.
- Editing part of a mesh does not trigger a full rebuild unless necessary.

### Stage 6 — Selection, tools, and Command History

**Deliverables**
- GPU Picking for objects and components using an integer ID Buffer.
- Single, multi-, and box selection.
- Object/Vertex/Edge/Face modes.
- Eight modeling tools.
- Temporary preview and confirm/cancel behavior.
- Command System and per-project Undo/Redo.

**Acceptance criteria**
- Selection matches the element under the touch, including occlusion cases.
- Box selection follows the left-to-right/right-to-left rules.
- Every confirmed operation can be undone/redone correctly.
- Cancelling a preview does not modify the original mesh.

### Stage 7 — Modifiers

**Deliverables**
- Modifier Stack.
- Non-destructive Mirror on X/Y/Z.
- Clipping and plane welding.
- Enable, disable, reorder, and apply.

**Acceptance criteria**
- Editing the source mesh updates the mirrored result.
- Applying Mirror produces geometry equivalent to the preview.
- Undo/Redo covers modifier editing and application.

### Stage 8 — Projects and X3D format

**Deliverables**
- Requirements-versus-v1-schema compatibility matrix.
- Documented decision to retain/extend v1 or create a new version.
- Reader/writer and migration, if applicable.
- Structural and reference validation.
- Atomic saving and recovery.
- Thumbnails and project directory structure.

**Acceptance criteria**
- Fixtures and tests for the supported version pass.
- Round-tripping preserves data within the format rules.
- Save failures preserve the previous file.
- Interrupted projects can be recovered without silently overwriting the original.

### Stage 9 — Import and export

**Deliverables**
- Assimp integration.
- glTF/GLB, FBX, OBJ, and DAE.
- Scene, geometry, basic material, and coordinate mapping.
- Progress, cancellation, and loss warnings.

**Acceptance criteria**
- A test corpus imports and exports in the declared formats.
- Errors and unsupported attributes are reported.
- Round-trip tests document expected losses for each format.

### Stage 10 — Complete interface and workflow

**Deliverables**
- Project manager.
- Panels, hierarchy, tools, contextual bar, and settings.
- Units/precision, Snap, axes, and preferences.
- SAF import/export.
- Autosave, recovery, diagnostics, and Crash Reports.

**Acceptance criteria**
- Create → model → save → close → reopen workflow works.
- Preferences persist.
- No heavy operation blocks the main thread.

### Stage 11 — Graphics quality and optimization

**Deliverables**
- Solid, Wireframe, Wireframe + Solid, and Flat Shading.
- Three-point auxiliary lighting.
- Shadows, CSM, SSAO, MSAA/FXAA, Tone Mapping, and Gamma Correction.
- Graphics profiles.
- Frustum Culling and memory/transfer optimizations.

**Acceptance criteria**
- Every optional feature checks device support and has a fallback.
- Low/Medium/High/Custom profiles behave consistently.
- Performance and memory tests on different GPUs document results and limitations.

### Stage 12 — Integration testing, stability, and 1.0 release

**Deliverables**
- Unit, integration, regression, and device tests.
- Extended memory and rendering tests.
- Corrupt-file, interruption, and recovery tests.
- Accessibility, translation, license, and credits review.
- User documentation, release notes, and publication checklist.

**Acceptance criteria**
- No known blockers remain in primary workflows.
- No project loss occurs in the covered failure scenarios.
- Vulkan 1.1 compatibility is verified on target devices.
- All advertised formats pass the conformance corpus.
- Performance criteria and minimum supported devices are published before release.

## 8. Risks and remaining decisions

These items do not block the functional specification but must be resolved in the indicated stages:

1. **Android build:** minimum Android version, SDK/NDK, ABIs, and Java/Kotlin choice — Stage 0.
2. **X3D format:** reconcile schema v1 with uint64 IDs, components, modifiers, and topology — Stage 8.
3. **Vulkan:** optional features vary by GPU/driver; all must be queried and have fallbacks — Stages 2, 3, and 11.
4. **Half-Edge and export:** external formats may require triangulation/conversion; preserve editable internal topology and export a compatible representation.
5. **Autosave and SAF:** atomic saving may have different limitations for SAF URIs; define a safe workflow for private storage and SAF export.
6. **Threading:** geometry must not be modified simultaneously by tools and workers; ownership and synchronization contracts are mandatory.
7. **Material scope:** only basic materials in 1.0; do not expose texturing controls that are not implemented.

## 9. Version 1.0 completion criteria

Version 1.0 is ready when users can create a project, import a model, navigate and edit its geometry with the approved tools, organize the scene, use Mirror, undo/redo, save and recover the project, export to the advertised formats, and work in a stable Vulkan viewport on compatible devices.

Release depends on documented tests, not merely completed screens or a successful build.

## 10. Related documents

- [README](../README.md)
- [Current technology architecture](architecture.md)
- [X3D v1 binary format](data-format.md)
- [X3D v1 conformance tests](format-v1-validation.md)

---

**Governance note:** Scope changes must update this document. Binary layout changes must update the format specification, its version, and the corresponding fixtures/tests. Stage status must only be updated when implementation evidence exists and acceptance criteria have been met.
