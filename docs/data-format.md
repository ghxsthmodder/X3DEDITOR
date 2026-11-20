# Internal Binary Format (.bin)

## Purpose and scope

X3D Editor uses a custom binary format to store projects. The initial format version is **1**, and the file extension is `.bin`. The format prioritizes simple reading and writing, low memory overhead, and strict validation.

Version 1 does not include compression, encryption, checksums, global offsets, generic serialization, multiple UV channels, submeshes, or multiple materials per mesh.

## General conventions

- Integers and floats are written in little-endian order.
- Floats use 32-bit IEEE 754 (`float32`).
- There is no padding or alignment between fields.
- Strings are valid UTF-8, preceded by their byte length (`uint32`), with no NUL terminator.
- Arrays are written sequentially, without additional headers beyond the counts defined in each record.
- Collections are zero-indexed, in the order records appear.
- Counts and sizes must be validated before any allocation or access.
- NaN and infinity `float32` values are invalid.

Types used: `uint16`, `uint32`, `int32`, and `float32`, each with exactly the size indicated by its name.

## Header

| Field | Type | Description |
|---|---|---|
| `magic` | 4 bytes | ASCII bytes `X3D` followed by NUL |
| `version` | `uint16` | Format version; initial value `1` |
| `sectionCount` | `uint16` | Total number of sections present |

The format version is independent of the project version stored in `META`. A version 1 reader rejects format versions it does not support.

## Sections

Each section has this header:

| Field | Type | Description |
|---|---|---|
| `tag` | 4 bytes | ASCII identifier |
| `size` | `uint32` | Payload size in bytes |
| `payload` | `size` bytes | Section data |

The five known sections in version 1 are `META`, `NODE`, `MESH`, `MATL`, and `TEXR`. Each must appear exactly once. Therefore, a valid version 1 file contains at least these five sections. Additional unknown sections may be skipped using `size`; they count toward `sectionCount`.

Missing or duplicated known sections, truncated payloads, invalid tags, or data exceeding file boundaries cause the file to be rejected. The reader must consume exactly `size` bytes for each payload.

## META — project metadata

The section contains a single record, with no additional count.

C++ representation:

```cpp
struct Metadata {
    std::string name;
    std::string description;

    uint32_t versionMajor;
    uint32_t versionMinor;
    uint32_t versionPatch;
};
```

Payload, in this order:

| Order | Field | Type |
|---:|---|---|
| 1 | `name` | String |
| 2 | `description` | String |
| 3 | `versionMajor` | `uint32` |
| 4 | `versionMinor` | `uint32` |
| 5 | `versionPatch` | `uint32` |

There is no author field. The major.minor.patch version belongs to the project, not the binary format.

## NODE — nodes and hierarchy

The payload begins with `nodeCount` (`uint32`), followed by nodes in collection order.

```cpp
struct Node {
    std::string name;
    int32_t parentIndex;
    std::vector<uint32_t> meshIndices;

    float position[3];
    float rotation[4];
    float scale[3];
};
```

Each record:

| Order | Field | Type |
|---:|---|---|
| 1 | `name` | String |
| 2 | `parentIndex` | `int32` |
| 3 | `meshCount` | `uint32` |
| 4 | `meshIndices` | `meshCount × uint32` |
| 5 | `position` | 3 × `float32` (XYZ) |
| 6 | `rotation` | 4 × `float32` (quaternion XYZW) |
| 7 | `scale` | 3 × `float32` (XYZ) |

`parentIndex = -1` indicates a root node; non-negative values reference the node collection. `meshIndices` references the mesh collection. Transforms are local, relative to the parent. A node may contain zero or multiple meshes.

The reader rejects invalid references, self-references, and hierarchy cycles. The quaternion must have finite components and a non-zero length.

## MESH — geometry

The payload begins with `meshCount` (`uint32`).

```cpp
struct Mesh {
    std::string name;
    std::vector<float> positions;
    std::vector<float> normals;
    std::vector<float> uvs;
    std::vector<uint32_t> indices;
    int32_t materialIndex;
};
```

Each record:

| Order | Field | Type |
|---:|---|---|
| 1 | `name` | String |
| 2 | `vertexCount` | `uint32` |
| 3 | `indexCount` | `uint32` |
| 4 | `materialIndex` | `int32` |
| 5 | `attributeFlags` | `uint32` |
| 6 | `positions` | `vertexCount × 3 × float32` |
| 7 | `normals` | `vertexCount × 3 × float32`, if bit 0 is set |
| 8 | `uvs` | `vertexCount × 2 × float32`, if bit 1 is set |
| 9 | `indices` | `indexCount × uint32` |

Version 1 flags: bit 0 (`0x1`) indicates normals; bit 1 (`0x2`) indicates UV0. All other bits must be zero. Positions are required; normals and UV0 are optional. Version 1 has only one UV channel.

Geometry uses triangles. `indexCount` must be divisible by three; every index must be less than `vertexCount`. `vertexCount` must be greater than zero. `indexCount = 0` is allowed for a mesh without faces. Geometry imported through Assimp must be triangulated before it is stored.

`materialIndex = -1` means no material; otherwise it references `MATL`. Each mesh has at most one material. Submeshes and multiple materials per mesh are not part of version 1.

Arrays are contiguous. xatlas output may change vertex correspondence and must be applied consistently to positions, indices, and UVs.

## MATL — materials

The payload begins with `materialCount` (`uint32`).

```cpp
struct Material {
    std::string name;
    float albedo[4];
    float metallic;
    float roughness;
    float opacity;
    float emissive[3];

    int32_t albedoTexture;
    int32_t normalTexture;
    int32_t metallicTexture;
    int32_t roughnessTexture;
    int32_t occlusionTexture;
    int32_t emissiveTexture;
};
```

Each record:

| Order | Field | Type |
|---:|---|---|
| 1 | `name` | String |
| 2 | `albedo` | 4 × `float32` (RGBA) |
| 3 | `metallic` | `float32` |
| 4 | `roughness` | `float32` |
| 5 | `opacity` | `float32` |
| 6 | `emissive` | 3 × `float32` (RGB) |
| 7–12 | Texture references | 6 × `int32`, in the structure field order |

The metallic, roughness, and opacity factors must be between 0 and 1. Defaults for a new material: albedo (1,1,1,1), metallic 0, roughness 1, opacity 1, and emissive (0,0,0). Each texture index references `TEXR`; `-1` means none. Metallic and roughness use independent references. Factors and textures remain separate; combining them is the responsibility of the material/rendering system.

## TEXR — texture references

The payload begins with `textureCount` (`uint32`).

```cpp
struct Texture {
    std::string name;
    std::string path;
};
```

Each record contains `name` (String) and `path` (String), in that order. The path is relative to the project folder and encoded in UTF-8. Absolute paths are not accepted. Pixel data remains in external files and is not embedded in the `.bin`.

The path must be normalized and validated to prevent resolution outside the project folder, including through `..` segments or symbolic links.

## Common validation rules

- All five known sections are required and must appear exactly once.
- Unknown sections may be safely skipped using their declared size.
- Collections are zero-indexed, in record order.
- Empty collections use a count of zero; `META` always contains one record.
- The reader validates sizes, counts, and arithmetic operations before allocating memory.
- The reader validates `parentIndex`, `meshIndices`, `materialIndex`, and all texture references.
- All floats must be finite.
- The reader must finish exactly at the end of each payload and at the end of the file.
- Signed integers are interpreted using two's complement.
- Version 1 contains no compression, encryption, checksums, global offsets, or padding.

## Safe writing

Writing must preserve the previous valid file until the new file has been completely written.

1. Create a unique temporary file in the same directory as the project file.
2. Serialize the entire file to the temporary file; check write errors and ensure all bytes were written.
3. Flush and synchronize the temporary file data to persistent storage (`fsync` on Android/POSIX).
4. Close the temporary file. If any step fails, remove the temporary file when possible and keep the original file intact.
5. Replace the destination using an atomic rename on the same filesystem.
6. Synchronize the parent directory when supported, to persist the rename.

The temporary file must be in the same directory so replacement can be atomic. Never truncate the destination before the temporary write is complete. If rename fails, the previous file remains and the operation returns an error. The implementation must handle temporary files left by interrupted executions without confusing them with valid projects.

## Reference example and tests

The file `tests/fixtures/v1-minimal.hex` contains a byte-for-byte hexadecimal fixture of a valid minimal version 1 project. It can be converted to binary for use as a test input. Acceptance, rejection, round-trip, and safe-write cases are described in `docs/format-v1-validation.md`.

## Library integration

- **Assimp:** imports external formats into the internal representation and exports scenes to supported formats.
- **xatlas:** receives positions and indices to generate UVs; its output must be applied consistently to the mesh geometry.

The `.bin` format is internal to the project and does not replace the interchange formats supported by Assimp.

## Specification status

- [x] Technical review and consolidation of version 1 rules.
- [x] Binary layouts for `META`, `NODE`, `MESH`, `MATL`, and `TEXR`.
- [x] Define UV0 as the only channel and exclude submeshes from version 1.
- [x] Minimal binary example in a hexadecimal fixture.
- [x] Documented read, write, validation, and safe-write test matrix.
- [x] Safe-write strategy using a temporary file, synchronization, and atomic rename.
- [ ] Implement the reader and writer and run the automated C++ test suite alongside the backend.
