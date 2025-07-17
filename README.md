# GRAPHITE Asset Graph Format

> **Graphite Recursive Asset Pipeline – Hypergraph Infinite Turtling Engine**

GRAPHITE is a production-ready binary format for game and application asset storage that unifies all asset types—textures, meshes, audio, scripts, dependencies, and transformations—under a single graph-based representation.

## Table of Contents

- [Quick Start](#quick-start)
- [Key Features](#key-features)
- [Documentation](#documentation)
- [Installation](#installation)
- [Usage](#usage)
- [Performance](#performance)
- [Contributing](#contributing)
- [License](#license)

## Quick Start

```bash
# Install GRAPHITE tools
make install

# Create a bundle from assets
graphite pack assets/ output.graphite

# Verify bundle integrity
graphite verify output.graphite

# Load in your application
#include <graphite/graphite.h>

graphite_bundle* bundle = graphite_open("output.graphite");
const graphite_graph* root = graphite_root(bundle);
// ... use assets ...
graphite_close(bundle);
```

## Key Features

### 🚀 Performance
- **Zero-Copy Loading**: Memory-mapped files with one-time pointer hydration
- **<200ms Load Times**: For 1GB bundles on 8-core systems
- **<3ms P99 Latency**: For individual asset operations
- **Parallel Processing**: Multi-threaded decompression and verification

### 🔧 Universal Graph Model
- **Everything is a Graph**: Assets, dependencies, transforms, and bundles
- **Recursive Composition**: Graphs can contain other graphs at arbitrary depth
- **Rich Relationships**: Edges themselves are graphs with semantic metadata

### 🔒 Security & Integrity
- **BLAKE3 Merkle Trees**: Cryptographic tamper detection
- **Per-Chunk CRC32**: Corruption detection
- **Optional Encryption**: AES-GCM support for sensitive assets

### 📦 Advanced Features
- **zstd Compression**: With dictionary training for optimal ratios
- **Hot Reload**: Live asset updates without restart
- **Cross-Platform**: Linux, macOS, Windows on x86_64, ARM64, and i386
- **Engine Integration**: Unity, Unreal, and Godot plugins available

## Documentation

### Core Documentation

1. **[Overview & Introduction](docs/01-overview.md)**
   - Executive summary
   - Core principles
   - Design philosophy
   - Use cases

2. **[System Architecture](docs/02-architecture.md)**
   - Component overview
   - Data flow
   - Memory management
   - Platform abstraction

3. **[Binary Format Specification](docs/03-binary-format.md)**
   - File structure
   - Header format
   - Chunk table
   - Data encoding

4. **[Graph Model](docs/04-graph-model.md)**
   - Graph theory foundation
   - Node and edge semantics
   - Property system
   - Special graph types

5. **[Security & Integrity](docs/05-integrity.md)**
   - BLAKE3 Merkle trees
   - Verification algorithms
   - Encryption support
   - Threat mitigation

6. **[Compression](docs/06-compression.md)**
   - zstd integration
   - Dictionary training
   - Compression strategies
   - Performance trade-offs

7. **[Performance Engineering](docs/07-performance.md)**
   - Memory optimization
   - Asynchronous I/O
   - SIMD acceleration
   - Benchmarking

### API Documentation

8. **[Core API](docs/08-api-core.md)**
   - Loading and access
   - Graph traversal
   - Property queries
   - Error handling

9. **[Tooling API & CLI](docs/09-api-tooling.md)**
   - Bundle creation
   - Graph building
   - Command-line tools
   - Analysis utilities

### Integration Guides

10. **[Engine Integration](docs/10-integration.md)**
    - Unity plugin
    - Unreal Engine
    - Godot
    - Custom engines

11. **[Implementation Guidelines](docs/11-implementation.md)**
    - Platform considerations
    - Build system integration
    - CI/CD pipelines
    - Best practices

### Reference

12. **[Appendices](docs/12-appendices.md)**
    - Reserved identifiers
    - Performance benchmarks
    - Compatibility matrix
    - Migration guides

## Installation

### Requirements

- C23 compiler with `_BitInt(40)` support (Clang 15+, GCC 13+, MSVC 2022 17.5+)
- zstd library (v1.5+)
- BLAKE3 implementation
- Platform: Linux, macOS, or Windows

### Building from Source

```bash
# Clone the repository
git clone https://github.com/yourusername/graphite.git
cd graphite

# Configure build
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release

# Build and test
make -j$(nproc)
make test

# Install
sudo make install
```

## Usage

### Creating Bundles

```bash
# Basic bundle creation
graphite pack assets/ bundle.graphite

# With compression
graphite pack --compress assets/ bundle.graphite

# With integrity verification
graphite pack --compress --verify assets/ bundle.graphite
```

### Loading in C

```c
#include <graphite/graphite.h>

// Open bundle with verification
graphite_bundle* bundle = graphite_open_with_flags(
    "assets.graphite",
    GRAPHITE_VERIFY_HASHES | GRAPHITE_DECOMPRESS
);

if (!bundle) {
    fprintf(stderr, "Failed to load: %s\n", 
            graphite_error_string(graphite_get_last_error()));
    return -1;
}

// Access root graph
const graphite_graph* root = graphite_root(bundle);

// Traverse graph
for (uint32_t i = 0; i < graphite_node_count(root); i++) {
    const graphite_graph* node = graphite_get_node(root, i);
    // Process node...
}

// Clean up
graphite_close(bundle);
```

### Unity Integration

```csharp
using Graphite;

public class AssetLoader : MonoBehaviour {
    void Start() {
        var bundle = GraphiteBundle.Load("assets.graphite");
        var texture = bundle.LoadAsset<Texture2D>("player_texture");
        // Use texture...
    }
}
```

## Performance

### Benchmark Results

| Bundle Size | Load Time | Memory Usage | Platform |
|-------------|-----------|--------------|----------|
| 10 MB | 12 ms | 12.8 MB | 8-core Zen 4 |
| 100 MB | 45 ms | 125 MB | 8-core Zen 4 |
| 1 GB | 182 ms | 1.2 GB | 8-core Zen 4 |
| 10 GB | 1.8 s | 11.6 GB | 8-core Zen 4 |

### Optimization Tips

1. **Use compression** for text-based assets (JSON, XML, scripts)
2. **Train dictionaries** for similar small files
3. **Enable prefetching** for predictable access patterns
4. **Use parallel groups** for independent assets

## Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### Development Setup

```bash
# Install development dependencies
./scripts/setup-dev.sh

# Run tests with sanitizers
./scripts/test-sanitizers.sh

# Run benchmarks
./scripts/benchmark.sh
```

## License

GRAPHITE is licensed under the MIT License. See [LICENSE](LICENSE) for details.

---

**Version:** 3.0.0  
**Status:** Production Ready  
**Specification:** [Full Technical Specification](docs/spec/spec.md)