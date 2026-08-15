#  Distribution & Release Infrastructure

This repository contains automated reconstruction tools, distribution manifests, and release assets organized across **5 isolated encoding/compression strategies**.

---

## Release Strategies Matrix

Each release represents an isolated engineering pipeline. The internal file contents and relative paths are identical across all strategies, differing only in the per-file transport transform:

| Strategy | Release Tag | Internal Transform | Transformation Tool | Purpose & Recommended Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Strategy A** | [`v1.0.0-strategy-a`](../../releases/tag/v1.0.0-strategy-a) | `<file>.gz` | `gzip -9` / `gunzip` | Standard UNIX compression; optimal decompression speed on POSIX systems. |
| **Strategy B** | [`v1.0.0-strategy-b`](../../releases/tag/v1.0.0-strategy-b) | `<file>.xz` | `xz -9` / `unxz` | High-efficiency LZMA2 compression; minimum network bandwidth usage. |
| **Strategy C** | [`v1.0.0-strategy-c`](../../releases/tag/v1.0.0-strategy-c) | `<file>.b64` | `base64` / `base64 -d` | 7-bit ASCII representation; transparent text inspection and egress gateways. |
| **Strategy D** | [`v1.0.0-strategy-d`](../../releases/tag/v1.0.0-strategy-d) | `<file>.tar` | `tar cf` / `tar xf` | Uncompressed POSIX single-file archives; preservation of raw streaming boundaries. |
| **Strategy E** | [`v1.0.0-strategy-e`](../../releases/tag/v1.0.0-strategy-e) | `<file>.dat` | `cp` / `mv` | Bit-for-bit raw byte streaming; zero decompression compute overhead. |

---

## Semantic Module Partitioning (< 75 MB per Container)

Every strategy batch is partitioned into **7 logical modules** to adhere strictly to transport boundaries and provide modular component extraction:

1. **Part 1 (`runtime-and-physics`)**: Core executables (`bin/zapdos-opt`), simulation decks (`sim/`), startup scripts (`run_airgapped_zapdos.sh`), and Zapdos/Crane/WASP physics modules.
2. **Part 2 (`system-dependencies`)**: System dynamic runtime shared libraries (crypto, ssl, boost, hdf5, xml, gfortran, openblas, zlib, X11, xcb).
3. **Part 3 (`numerical-solvers`)**: High-performance scientific solvers (PETSc, LAPACK, ScaLAPACK, HYPRE, MUMPS, BLAS).
4. **Part 4 (`netgen-mesh-engine`)**: Netgen 3D tetrahedral mesh engine shared libraries (`libnglib`, `libngcore`).
5. **Part 5 (`libmesh-fem-engine`)**: LibMesh adaptive mesh refinement and parallel finite element engine (`libmesh_opt`).
6. **Part 6 (`moose-framework-core`)**: MOOSE multiphysics framework runtime shared libraries (`libmoose-opt.so`, `libmoose-opt.so.0`).
7. **Part 7 (`moose-framework-symbols`)**: MOOSE framework versioned shared library components (`libmoose-opt.so.0.0.0`).

---

## One-Click Automated Reconstruction

The provided [`reconstruct.sh`](reconstruct.sh) script automatically detects the downloaded strategy, unpacks containers, reverses file transformations, configures execute permissions, and verifies SHA-256 checksums against the manifest.

### 1. Download Release Assets (Example for Strategy A)
```bash
# Using GitHub CLI:
gh release download v1.0.0-strategy-a --pattern "*.tar.gz" --pattern "MANIFEST.md"

# Or download via curl / browser into your working directory
```

### 2. Run Reconstruction
```bash
# Make script executable
chmod +x reconstruct.sh

# Run automatic reconstruction
./reconstruct.sh
```

### Optional Flags:
```bash
# Force a specific strategy:
./reconstruct.sh --strategy strategy-b

# Clean up part packages after successful verification:
./reconstruct.sh --clean

# Target a different directory:
./reconstruct.sh --dir /path/to/downloads
```

---

## Manual Step-by-Step Reconstruction

If you prefer manual restoration without `reconstruct.sh`:

```bash
# 1. Extract all 7 container parts into workspace
for f in zapdos-airgap-v1.0.0-strategy-a-part*.tar.gz; do
    tar -xzf "$f"
done

# 2. Decompress/Decode all internal files (depending on strategy):
# For Strategy A (.gz):
find zapdos_airgapped_standalone -name "*.gz" -exec gzip -d {} +

# For Strategy B (.xz):
find zapdos_airgapped_standalone -name "*.xz" -exec xz -d {} +

# For Strategy C (.b64):
find zapdos_airgapped_standalone -name "*.b64" | while read f; do
    base64 -d "$f" > "${f%.b64}" && rm "$f"
done

# For Strategy D (.tar):
find zapdos_airgapped_standalone -name "*.tar" | while read f; do
    tar -xf "$f" -C "$(dirname "$f")" && rm "$f"
done

# For Strategy E (.dat):
find zapdos_airgapped_standalone -name "*.dat" | while read f; do
    mv "$f" "${f%.dat}"
done

# 3. Set binary permissions
chmod +x zapdos_airgapped_standalone/zapdos_airgapped_bundle/bin/*
chmod +x zapdos_airgapped_standalone/zapdos_airgapped_bundle/run_airgapped_zapdos.sh

# 4. Launch Zapdos Simulation
cd zapdos_airgapped_standalone/zapdos_airgapped_bundle
./run_airgapped_zapdos.sh sim/native_zapdos_pulsed_2d.i
```

---
