# Technology Architecture

## Overview

X3D Editor will use a hybrid architecture that separates the user interface layer from the 3D processing layer.

## Languages

### Java / Kotlin — UI

Java and/or Kotlin will be used for the user interface. The choice between using one language or combining both may be finalized during implementation.

Planned responsibilities:
- Material Design interface.
- Project management and listing.
- Menus, dialogs, settings, and controls.
- Editing and texturing workflows.
- Communication with the native backend.

### C++ — Backend

C++ will be used for 3D processing and data manipulation.

Planned responsibilities:
- Geometry and mesh processing.
- xatlas integration for UV unwrapping and island generation.
- Assimp integration for model import/export.
- Preparing model data for editing workflows.
- Heavy operations that must not block the UI.

## Libraries

### xatlas

Repository: https://github.com/jpcy/xatlas

Selected library for automatic UV map generation:
- UV unwrapping.
- UV island generation.
- Island packing/arrangement using the capabilities provided by the library.

### Assimp

Repository: https://github.com/assimp/assimp

Selected library to facilitate interoperability with 3D model formats:
- Import and export through formats and features supported by the integrated version.
- Conversion between scene and mesh structures.
- Each exporter must be validated individually, including materials, textures, hierarchy, and mesh attributes.

## Communication between UI and backend

The Java/Kotlin UI will call native C++ functions through **JNI (Java Native Interface)**. Since the project is Android-only, there is no need to abstract this integration for other platforms. The JNI API contract is still to be defined.

Principles:
- Do not run heavy operations on the UI main thread.
- Avoid unnecessary copies of large buffers between the JVM and C++.
- Clearly define memory ownership, lifetime, and release.
- Return progress, errors, and results in a controlled manner.
- Keep the native API independent of visual UI details whenever possible.

## Conceptual processing flow

1. The UI requests an operation.
2. The integration layer forwards the required data to the backend.
3. The C++ backend loads or processes the geometry.
4. xatlas generates the UV mapping when requested.
5. Assimp is used to import/export supported formats.
6. The backend returns the result and any error messages to the UI.

## Internal data format

The internal format will be binary from the first version, using the **`.bin`** extension. The structure is intentionally simple: a small header followed by identified sections, each containing data for a category (for example, scene, meshes, materials, and texture references).

The format will have an identifier and version so the backend can validate files and evolve the schema without unnecessary complexity. Numeric arrays will be stored directly in binary form, avoiding text serialization of large amounts of vertex, index, and UV data.

The initial specification is in [data-format.md](data-format.md).

## Decisions still pending

- Minimum Android version.
- Detailed JNI API contract.
- Whether Java, Kotlin, or both will be used in the UI code.
- Priority input and output formats.
- Detailed mesh, material, and scene data model.
- Attribute and material preservation policy during export.
- Cancellation and progress handling for long-running operations.
