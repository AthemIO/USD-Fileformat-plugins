# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

USD File Format Plugins enable interchange between Pixar's USD and various 3D file formats (FBX, glTF, OBJ, PLY, SPZ, STL, SBSAR). Each plugin implements USD's `SdfFileFormat` interface.

## Build Commands

```bash
# Configure (multi-config backend like MSVC)
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=bin -Dpxr_ROOT=<USD_INSTALL_PATH> <OPTIONS>

# Configure (single-config backend like Make)
cmake -S . -B build -DCMAKE_INSTALL_PREFIX=bin -DCMAKE_BUILD_TYPE=Release -Dpxr_ROOT=<USD_INSTALL_PATH> <OPTIONS>

# Build
cmake --build build --config release

# Install
cmake --install build --config release
```

Common options: `-DUSD_FILEFORMATS_ENABLE_<PLUGIN>=ON/OFF` (FBX, GLTF, OBJ, PLY, SPZ, STL, SBSAR)

Linux requires: `-DUSD_FILEFORMATS_ENABLE_CXX11_ABI=ON`

## Running Tests

```bash
# Install Python dependencies
pip install -r scripts/requirements.txt

# Run pytest tests (requires USD built with --openimageio)
pytest test/test.py

# Generate new baseline data
python test/test.py --generate_baseline
```

GTest-based sanity tests are in each plugin's `tests/` directory and run via CTest after building.

## Architecture

### Plugin Structure
Each plugin (`fbx/`, `gltf/`, `obj/`, `ply/`, `spz/`, `stl/`, `sbsar/`) follows the same pattern:
- `src/fileFormat.cpp` - Implements `SdfFileFormat::Read()` and `WriteToFile()`
- `src/<format>Import.cpp` - Converts native format → `UsdData`
- `src/<format>Export.cpp` - Converts `UsdData` → native format
- `src/<format>.cpp` - Native file reader/writer

### Shared Utilities (`utils/`)
The `fileformatUtils` library in `utils/` provides:
- **`usdData.h`** - Core data structures (`UsdData`, `Node`, `Mesh`, `Material`, etc.) that plugins populate during import
- **`layerRead.h`** - `readLayer()` reads USD layer into `UsdData`
- **`layerWriteSdfData.h`** - `writeLayer()` writes `UsdData` to USD layer
- **`geometry.h`** - Mesh processing utilities
- **`materials.h`** - Material/texture handling, Phong→PBR conversion
- **`images.h`** - Image I/O and caching
- **`resolver.h`** - Custom package resolver for embedded textures

### Import/Export Pattern
All plugins follow this flow:

**Import:**
```cpp
NativeFormat native;
UsdData usd;
readNative(native, path);           // Parse file with 3rd party lib
importNative(options, native, usd); // Convert to UsdData
writeLayer(options, usd, layer);    // Write to USD layer
```

**Export:**
```cpp
NativeFormat native;
UsdData usd;
readLayer(options, layer, usd);     // Read from USD layer
exportNative(options, usd, native); // Convert to native format
writeNative(native, filename);      // Write with 3rd party lib
```

## Key Dependencies

- **Pixar USD** (23.08+) - Core dependency
- **FBX SDK** (2020.3.7) - For usdfbx
- **TinyGLTF** - For usdgltf
- **Eigen** - For usdply, usdspz
- **fmt/FastFloat** - For usdobj

Many dependencies auto-fetch via CMake FetchContent (controlled by `USD_FILEFORMATS_FETCH_*` options).

## Environment Setup

After building, set `PXR_PLUGINPATH_NAME` to include `<INSTALL_PATH>/plugin/usd` for USD to discover plugins.
