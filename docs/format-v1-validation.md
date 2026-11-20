# X3D v1 Format Conformance Tests

This document defines conformance tests for the binary format reader and writer. The reference fixture is `tests/fixtures/v1-minimal.hex`.

## Minimal fixture

The fixture is 169 bytes and represents:

- Header `X3D\\0`, format version 1, and five sections.
- `META`: name `Example`, description `Minimal X3D v1 fixture`, project version 1.0.0.
- `NODE`: one node named `Root`, with no parent and no meshes; position (0,0,0), identity quaternion (0,0,0,1), and scale (1,1,1).
- `MESH`, `MATL`, and `TEXR`: empty collections.

The fixture is a hexadecimal sequence without addresses or semantic separators. Whitespace and line breaks are ignored when converting it to bytes.

## Required positive tests

| ID | Scenario | Expected result |
|---|---|---|
| P-01 | Read the minimal fixture | Accept and reconstruct exactly the metadata and node described above |
| P-02 | Read empty collections | Accept MESH, MATL, and TEXR with a count of zero |
| P-03 | Fixture round-trip | Read and write again; produce the same 169 bytes |
| P-04 | Additional unknown section with a valid size | Skip the unknown payload and continue reading |
| P-05 | Node with a valid parent and valid references | Reconstruct the hierarchy and associations correctly |
| P-06 | Mesh with normals and UV0 enabled | Read arrays according to bits 0 and 1 of attributeFlags |
| P-07 | Material with valid texture references | Resolve each index to the correct TEXR record |

## Required negative tests

| ID | Invalid scenario | Expected result |
|---|---|---|
| N-01 | Magic value differs from `X3D\\0` | Reject |
| N-02 | Unsupported format version | Reject with a version error |
| N-03 | Truncated header | Reject |
| N-04 | sectionCount does not match available sections | Reject |
| N-05 | One of the five required sections is missing | Reject |
| N-06 | Known section appears more than once | Reject |
| N-07 | size exceeds the remaining bytes | Reject without reading out of bounds |
| N-08 | Known payload contains extra or missing bytes | Reject |
| N-09 | String length exceeds the payload | Reject |
| N-10 | Invalid UTF-8 string | Reject |
| N-11 | Count causes overflow or allocation beyond available limits | Reject before allocation |
| N-12 | parentIndex, meshIndices, materialIndex, or texture index is out of range | Reject |
| N-13 | Hierarchy contains a cycle or a node is its own parent | Reject |
| N-14 | attributeFlags contains unknown bits | Reject |
| N-15 | indexCount is not divisible by three, or an index is greater than or equal to vertexCount | Reject |
| N-16 | metallic, roughness, or opacity is outside [0,1] | Reject |
| N-17 | Float NaN or infinity in any attribute | Reject |
| N-18 | Absolute path or texture path escaping the project folder | Reject |
| N-19 | Extra bytes after the final section | Reject |

## Safe-save tests

| ID | Simulation | Expected result |
|---|---|---|
| S-01 | Save completes and rename succeeds | New file fully replaces the previous file |
| S-02 | Temporary file creation fails | Previous file remains intact |
| S-03 | Write or flush fails | Temporary file is discarded when possible; previous destination remains intact |
| S-04 | fsync fails before rename | Do not replace the destination |
| S-05 | Rename fails | Return an error and preserve the previous destination |
| S-06 | Process is interrupted before rename | Previous destination remains valid; orphaned temporary file may be cleaned up later |
| S-07 | Save to a directory without permission | Return an error without truncating the previous file |

## Pass criteria

- All positive tests are accepted and all negative tests are rejected.
- Round-trip P-03 must be byte-for-byte identical for the canonical fixture.
- Read errors must never cause out-of-bounds access or unbounded allocation.
- On save failures before rename, the previous file must remain intact.
- Automated reader/writer tests must be implemented in C++ alongside the backend. The hexadecimal fixture serves as an implementation-independent reference vector.
