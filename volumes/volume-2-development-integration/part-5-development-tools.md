# Volume 2: Development & Integration
## Part 5: Development Tools

### Table of Contents
- [Chapter 12: Build System](#chapter-12-build-system)
  - [Build Pipeline Integration](#build-pipeline-integration)
  - [CMake Integration](#cmake-integration)
  - [Other Build Systems](#other-build-systems)
  - [CI/CD Integration](#cicd-integration)
  - [Incremental Builds](#incremental-builds)
- [Chapter 13: Command Line Tools](#chapter-13-command-line-tools)
  - [Core CLI Tools](#core-cli-tools)
  - [Bundle Operations](#bundle-operations)
  - [Analysis Tools](#analysis-tools)
  - [Development Utilities](#development-utilities)
- [Chapter 14: Testing Framework](#chapter-14-testing-framework)
  - [Unit Testing](#unit-testing)
  - [Integration Testing](#integration-testing)
  - [Property-Based Testing](#property-based-testing)
  - [Performance Testing](#performance-testing)
  - [Fuzz Testing](#fuzz-testing)
- [Cross-References](#cross-references)
- [Navigation](#navigation)

### Overview

This part covers TurtlGraph's comprehensive development toolchain, including build system integration, command-line utilities, and testing frameworks. Chapter 12 details seamless integration with modern build systems. Chapter 13 presents the full suite of command-line tools. Chapter 14 covers the comprehensive testing strategy from unit tests to fuzz testing.

---

## Chapter 12: Build System

TurtlGraph integrates seamlessly with modern build systems, providing first-class support for automated bundle generation, dependency tracking, and incremental builds across all major platforms.

### Build Pipeline Integration

TurtlGraph's build integration follows a declarative approach:

```mermaid
graph LR
    subgraph "Input Assets"
        TEX["Textures<br/>PNG/JPG/TGA"]
        MESH["Meshes<br/>FBX/GLTF/OBJ"]
        AUDIO["Audio<br/>WAV/OGG/MP3"]
        SCRIPTS["Scripts<br/>JSON/YAML"]
    end
    
    subgraph "Processing Pipeline"
        PROC["Asset Processors"]
        DEPS["Dependency Analysis"]
        OPT["Optimization"]
        PACK["Bundle Packing"]
    end
    
    subgraph "Output"
        BUNDLE["HyperDAG Bundle<br/>.turtlgraph"]
        MANIFEST["Manifest<br/>Dependencies"]
        REPORT["Build Report<br/>Statistics"]
    end
    
    TEX --> PROC
    MESH --> PROC
    AUDIO --> PROC
    SCRIPTS --> PROC
    
    PROC --> DEPS
    DEPS --> OPT
    OPT --> PACK
    
    PACK --> BUNDLE
    PACK --> MANIFEST
    PACK --> REPORT
```

### CMake Integration

TurtlGraph provides comprehensive CMake modules for seamless integration:

```cmake
# FindTurtlGraph.cmake module
find_package(TurtlGraph REQUIRED)

# Create bundle target with full configuration
hyperdag_add_bundle(MyGame_Assets
    SOURCES 
        ${ASSET_FILES}
        assets/textures/*.png
        assets/models/*.gltf
        assets/sounds/*.wav
    OUTPUT ${CMAKE_BINARY_DIR}/assets/game.turtlgraph
    COMPRESSION zstd
    COMPRESSION_LEVEL 19
    INTEGRITY blake3
    HOT_RELOAD $<CONFIG:Debug>
    DICTIONARY ${CMAKE_SOURCE_DIR}/assets/dict.zstd
)

# Advanced dependency tracking
hyperdag_add_dependencies(MyGame_Assets
    DEPENDS 
        texture_processor 
        mesh_optimizer
        audio_encoder
    TRIGGERS 
        ${SHADER_FILES} 
        ${TEXTURE_FILES}
        ${CONFIG_FILES}
)

# Custom transformation rules
hyperdag_add_transform(texture_transform
    INPUT_PATTERN "*.png;*.jpg;*.tga"
    OUTPUT_PATTERN "*.g_tex"
    COMMAND texture_processor 
        --format $<IF:$<CONFIG:Release>,bc7,rgba8>
        --mips 
        --quality $<IF:$<CONFIG:Release>,production,preview>
        --output $output 
        $input
    CACHE_KEY content_hash
    CACHE_DIR ${CMAKE_BINARY_DIR}/.hyperdag_cache
)

# Platform-specific bundle generation
if(WIN32)
    hyperdag_add_platform_bundle(MyGame_Assets_Win
        BASE_BUNDLE MyGame_Assets
        PLATFORM_ASSETS assets/platform/windows/*
        COMPRESSION_ALGORITHM lz4  # Faster for runtime
    )
elseif(APPLE)
    hyperdag_add_platform_bundle(MyGame_Assets_Mac
        BASE_BUNDLE MyGame_Assets
        PLATFORM_ASSETS assets/platform/macos/*
        CODE_SIGN_IDENTITY ${APPLE_DEVELOPER_ID}
    )
endif()

# Integration with install targets
install(FILES 
    $<TARGET_PROPERTY:MyGame_Assets,OUTPUT_FILE>
    DESTINATION ${CMAKE_INSTALL_PREFIX}/data
    COMPONENT GameAssets
)

# Generate asset manifest for runtime
hyperdag_generate_manifest(MyGame_Assets
    OUTPUT ${CMAKE_BINARY_DIR}/asset_manifest.h
    NAMESPACE MyGame
    LANGUAGE CXX
)
```

#### Advanced CMake Features

```cmake
# Multi-bundle dependencies
hyperdag_create_bundle_group(AllGameAssets
    BUNDLES
        CoreAssets
        LevelAssets
        CharacterAssets
        UIAssets
    MERGE_DUPLICATES ON
    CROSS_REFERENCE ON
)

# Conditional asset inclusion
hyperdag_add_bundle(DemoAssets
    SOURCES assets/demo/*
    CONDITION $<OR:$<CONFIG:Debug>,$<BOOL:${BUILD_DEMO}>>
)

# Asset validation rules
hyperdag_add_validation(MyGame_Assets
    RULES
        MAX_BUNDLE_SIZE 2GB
        MAX_TEXTURE_SIZE 4096
        REQUIRED_MIPMAPS ON
        COMPRESSION_RATIO_MIN 0.5
    FAIL_ON_WARNING $<CONFIG:Release>
)

# Performance profiling integration
hyperdag_enable_profiling(MyGame_Assets
    REPORT_FILE ${CMAKE_BINARY_DIR}/asset_build_profile.json
    TRACE_LEVEL detailed
    MEASURE_COMPRESSION_TIME ON
    MEASURE_IO_TIME ON
)
```

### Other Build Systems

#### Ninja Integration

Direct integration with Ninja for blazing-fast builds:

```ninja
# TurtlGraph build rules
rule turtlgraph_pack
  command = turtlgraph pack --config $config --output $out $in
  description = Packing HyperDAG bundle $out
  deps = gcc
  depfile = $out.d
  restat = 1

rule turtlgraph_incremental
  command = turtlgraph update --bundle $bundle --changed $in --output $out
  description = Incrementally updating $bundle
  deps = gcc
  depfile = $out.d

# Build targets
build assets/game.turtlgraph: turtlgraph_pack $
    assets/textures/*.png $
    assets/models/*.gltf $
    assets/config/*.json $
    | hyperdag_config.json
  config = release

# Incremental updates
build assets/game_updated.turtlgraph: turtlgraph_incremental $
    assets/textures/player.png
  bundle = assets/game.turtlgraph

# Asset processing pipeline
rule texture_process
  command = texture_processor --format bc7 --output $out $in

build processed/player.g_tex: texture_process assets/textures/player.png

# Bundle with processed assets
build final/game.turtlgraph: turtlgraph_pack processed/*.g_tex
  config = final
```

#### Bazel Integration

Hermetic builds with Bazel:

```python
# //tools/turtlgraph:defs.bzl
load("@bazel_skylib//lib:paths.bzl", "paths")

def hyperdag_bundle(name, srcs, compression="zstd", integrity="blake3", 
                   dictionary=None, platform=None, **kwargs):
    """Create a HyperDAG bundle from source assets."""
    
    cmd_parts = [
        "$(location //tools/turtlgraph:pack)",
        "--compression=%s" % compression,
        "--integrity=%s" % integrity,
    ]
    
    if dictionary:
        cmd_parts.append("--dictionary=$(location %s)" % dictionary)
    
    if platform:
        cmd_parts.append("--platform=%s" % platform)
    
    cmd_parts.extend([
        "--output $@",
        "$(SRCS)"
    ])
    
    native.genrule(
        name = name,
        srcs = srcs,
        outs = [name + ".turtlgraph"],
        cmd = " ".join(cmd_parts),
        tools = ["//tools/turtlgraph:pack"],
        **kwargs
    )

def hyperdag_test_bundle(name, bundle, **kwargs):
    """Test a HyperDAG bundle for integrity and performance."""
    native.sh_test(
        name = name + "_test",
        srcs = ["//tools/turtlgraph:test_runner.sh"],
        data = [bundle],
        args = ["$(location %s)" % bundle],
        **kwargs
    )

# BUILD file usage
load("//tools/turtlgraph:defs.bzl", "hyperdag_bundle", "hyperdag_test_bundle")

hyperdag_bundle(
    name = "game_assets",
    srcs = glob(["assets/**/*"]),
    compression = "zstd",
    compression_level = "19",
    integrity = "blake3",
    dictionary = "//assets:compression_dict",
    platform = select({
        "//platforms:windows": "win64",
        "//platforms:linux": "linux64",
        "//platforms:macos": "darwin64",
    }),
)

hyperdag_test_bundle(
    name = "game_assets_test",
    bundle = ":game_assets",
    size = "small",
)
```

#### Make Integration

Traditional Make with modern features:

```makefile
# TurtlGraph Makefile integration
TURTLGRAPH := turtlgraph
TURTLGRAPH_FLAGS := --compression zstd --integrity blake3

# Directories
ASSET_DIR := assets
BUILD_DIR := build
CACHE_DIR := .hyperdag_cache

# Find all assets
TEXTURES := $(wildcard $(ASSET_DIR)/textures/*.png)
MODELS := $(wildcard $(ASSET_DIR)/models/*.gltf)
CONFIGS := $(wildcard $(ASSET_DIR)/config/*.json)

ALL_ASSETS := $(TEXTURES) $(MODELS) $(CONFIGS)

# Main bundle target
$(BUILD_DIR)/game.turtlgraph: $(ALL_ASSETS) turtlgraph.config
	@mkdir -p $(BUILD_DIR)
	$(TURTLGRAPH) pack $(TURTLGRAPH_FLAGS) \
		--config turtlgraph.config \
		--output $@ \
		--depfile $@.d \
		--cache-dir $(CACHE_DIR) \
		$(ALL_ASSETS)

# Include dependency files
-include $(BUILD_DIR)/*.d

# Incremental updates
.PHONY: incremental
incremental: $(BUILD_DIR)/game.turtlgraph
	@echo "Checking for asset updates..."
	@$(TURTLGRAPH) update \
		--bundle $< \
		--verify \
		--incremental

# Platform-specific bundles
$(BUILD_DIR)/game_%.turtlgraph: $(BUILD_DIR)/game.turtlgraph
	$(TURTLGRAPH) repack \
		--input $< \
		--platform $* \
		--output $@

# Clean targets
.PHONY: clean clean-cache
clean:
	rm -rf $(BUILD_DIR)

clean-cache:
	rm -rf $(CACHE_DIR)

# Performance analysis
.PHONY: analyze
analyze: $(BUILD_DIR)/game.turtlgraph
	$(TURTLGRAPH) analyze $< \
		--report $(BUILD_DIR)/analysis.html \
		--format html
```

### CI/CD Integration

#### GitHub Actions

Comprehensive CI/CD workflow:

```yaml
name: Build and Deploy Assets

on:
  push:
    branches: [main, develop]
  pull_request:
    paths:
      - 'assets/**'
      - 'turtlgraph.config'

env:
  TURTLGRAPH_VERSION: 1.0.0
  CACHE_KEY_PREFIX: turtlgraph-cache-v1

jobs:
  build-assets:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        config: [debug, release]
    
    steps:
    - uses: actions/checkout@v3
      with:
        lfs: true  # For large asset files
    
    - name: Setup TurtlGraph
      uses: turtlgraph-tools/setup-turtlgraph@v1
      with:
        version: ${{ env.TURTLGRAPH_VERSION }}
    
    - name: Cache processed assets
      uses: actions/cache@v3
      with:
        path: |
          .hyperdag_cache
          build/cache
        key: ${{ env.CACHE_KEY_PREFIX }}-${{ matrix.os }}-${{ matrix.config }}-${{ hashFiles('assets/**') }}
        restore-keys: |
          ${{ env.CACHE_KEY_PREFIX }}-${{ matrix.os }}-${{ matrix.config }}-
          ${{ env.CACHE_KEY_PREFIX }}-${{ matrix.os }}-
    
    - name: Validate assets
      run: |
        turtlgraph validate assets/ \
          --config validation-rules.json \
          --report validation-report.json
    
    - name: Build asset bundles
      run: |
        turtlgraph pack \
          --config ${{ matrix.config }}.json \
          --platform ${{ runner.os }} \
          --output dist/${{ matrix.os }}-${{ matrix.config }}.turtlgraph \
          --stats build-stats.json \
          assets/
    
    - name: Test bundle integrity
      run: |
        turtlgraph verify dist/${{ matrix.os }}-${{ matrix.config }}.turtlgraph \
          --full \
          --report integrity-report.json
    
    - name: Measure performance
      run: |
        turtlgraph benchmark dist/${{ matrix.os }}-${{ matrix.config }}.turtlgraph \
          --iterations 10 \
          --report performance-report.json
    
    - name: Generate size report
      run: |
        turtlgraph size-report dist/${{ matrix.os }}-${{ matrix.config }}.turtlgraph \
          --format markdown \
          --output size-report.md
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: asset-bundles-${{ matrix.os }}-${{ matrix.config }}
        path: |
          dist/*.turtlgraph
          *-report.*
        retention-days: 30
    
    - name: Comment PR with reports
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v6
      with:
        script: |
          const fs = require('fs');
          const sizeReport = fs.readFileSync('size-report.md', 'utf8');
          
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.name,
            body: `## Asset Build Report\n\n${sizeReport}`
          });

  deploy-assets:
    needs: build-assets
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    
    steps:
    - name: Download all artifacts
      uses: actions/download-artifact@v3
    
    - name: Deploy to CDN
      env:
        CDN_BUCKET: ${{ secrets.CDN_BUCKET }}
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      run: |
        for bundle in asset-bundles-*/*.turtlgraph; do
          aws s3 cp $bundle s3://$CDN_BUCKET/assets/ \
            --content-type application/octet-stream \
            --cache-control "public, max-age=3600"
        done
    
    - name: Invalidate CDN cache
      run: |
        aws cloudfront create-invalidation \
          --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
          --paths "/assets/*"
```

### Incremental Builds

TurtlGraph's incremental build system minimizes rebuild times:

```c
// Incremental build configuration
typedef struct {
    const char* cache_directory;
    const char* dependency_file;
    bool use_content_hash;
    bool use_timestamps;
    size_t max_cache_size;
    uint32_t cache_ttl_days;
} hyperdag_incremental_config;

// Dependency tracking
typedef struct {
    char* asset_path;
    uint64_t last_modified;
    uint8_t content_hash[32];
    char** dependencies;
    size_t dep_count;
} asset_dependency;

// Incremental build context
typedef struct {
    hyperdag_incremental_config config;
    hash_table* asset_cache;
    hash_table* dependency_graph;
    dynamic_array* changed_assets;
    dynamic_array* affected_bundles;
} incremental_build_context;

// Check if rebuild needed
bool needs_rebuild(incremental_build_context* ctx, const char* asset_path) {
    asset_dependency* dep = hash_table_get(ctx->dependency_graph, asset_path);
    if (!dep) {
        return true; // New asset
    }
    
    // Check timestamp
    struct stat st;
    if (stat(asset_path, &st) != 0) {
        return true; // File missing
    }
    
    if (ctx->config.use_timestamps && st.st_mtime > dep->last_modified) {
        return true; // Modified
    }
    
    // Check content hash
    if (ctx->config.use_content_hash) {
        uint8_t current_hash[32];
        if (compute_file_hash(asset_path, current_hash) != 0) {
            return true; // Hash computation failed
        }
        
        if (memcmp(current_hash, dep->content_hash, 32) != 0) {
            return true; // Content changed
        }
    }
    
    // Check dependencies recursively
    for (size_t i = 0; i < dep->dep_count; i++) {
        if (needs_rebuild(ctx, dep->dependencies[i])) {
            return true; // Dependency changed
        }
    }
    
    return false; // No rebuild needed
}

// Incremental bundle update
hyperdag_result update_bundle_incremental(
    const char* bundle_path,
    incremental_build_context* ctx
) {
    // Load existing bundle
    hyperdag_bundle* bundle = hyperdag_open(bundle_path, HYPERDAG_OPEN_WRITE);
    if (!bundle) {
        return HYPERDAG_ERROR_BUNDLE_OPEN;
    }
    
    // Find changed assets
    dynamic_array_clear(ctx->changed_assets);
    
    hash_table_iterator iter;
    hash_table_iter_init(&iter, ctx->dependency_graph);
    
    while (hash_table_iter_next(&iter)) {
        const char* asset_path = iter.key;
        if (needs_rebuild(ctx, asset_path)) {
            dynamic_array_push(ctx->changed_assets, &asset_path);
        }
    }
    
    if (ctx->changed_assets->size == 0) {
        hyperdag_close(bundle);
        return HYPERDAG_SUCCESS; // Nothing to update
    }
    
    // Process changed assets
    for (size_t i = 0; i < ctx->changed_assets->size; i++) {
        const char* asset_path = *(const char**)dynamic_array_get(
            ctx->changed_assets, i);
        
        // Process asset
        processed_asset* processed = process_asset(asset_path);
        if (!processed) {
            hyperdag_close(bundle);
            return HYPERDAG_ERROR_ASSET_PROCESSING;
        }
        
        // Update in bundle
        hyperdag_result result = hyperdag_bundle_update_chunk(
            bundle, 
            processed->chunk_id, 
            processed->data, 
            processed->size
        );
        
        if (result != HYPERDAG_SUCCESS) {
            free_processed_asset(processed);
            hyperdag_close(bundle);
            return result;
        }
        
        // Update cache
        update_asset_cache(ctx, asset_path, processed);
        
        free_processed_asset(processed);
    }
    
    // Update bundle metadata
    hyperdag_bundle_update_timestamp(bundle);
    hyperdag_bundle_recalculate_hashes(bundle);
    
    hyperdag_close(bundle);
    
    // Save dependency graph
    save_dependency_graph(ctx);
    
    return HYPERDAG_SUCCESS;
}
```

---

## Chapter 13: Command Line Tools

TurtlGraph provides a comprehensive suite of command-line tools for asset management, debugging, and optimization.

### Core CLI Tools

The main `turtlgraph` command provides all essential operations:

```bash
# Basic information and inspection
turtlgraph info bundle.turtlgraph           # Show bundle statistics
turtlgraph ls bundle.turtlgraph             # List bundle contents
turtlgraph ls bundle.turtlgraph /textures   # List specific path
turtlgraph tree bundle.turtlgraph           # Show hierarchy tree

# Content extraction
turtlgraph cat bundle.turtlgraph /strings/42      # Extract string by ID
turtlgraph extract bundle.turtlgraph /meshes/player.mesh -o player.mesh
turtlgraph extract-all bundle.turtlgraph -o extracted/  # Extract everything

# Integrity verification  
turtlgraph verify bundle.turtlgraph         # Full integrity check
turtlgraph verify --quick bundle.turtlgraph # CRC only
turtlgraph verify --deep bundle.turtlgraph  # Full hash verification

# Performance testing
turtlgraph bench bundle.turtlgraph          # Performance measurement
turtlgraph bench --detailed bundle.turtlgraph    # Per-stage timing
turtlgraph bench --compare old.turtlgraph new.turtlgraph  # A/B comparison
```

### Bundle Operations

Creating and manipulating bundles:

```bash
# Bundle creation
turtlgraph pack assets/ output.turtlgraph   # Basic packing
turtlgraph pack --compress assets/ output.turtlgraph # With compression
turtlgraph pack --compress=zstd:19 assets/ output.turtlgraph # Max compression
turtlgraph pack --dict=dict.zstd assets/ output.turtlgraph # Dictionary compression
turtlgraph pack --encrypt --key=secret.key assets/ output.turtlgraph # Encrypted

# Advanced packing options
turtlgraph pack \
  --config production.json \
  --platform windows \
  --exclude "*.psd" \
  --exclude "source/" \
  --manifest manifest.json \
  --sign cert.pem \
  --timestamp \
  assets/ game.turtlgraph

# Bundle manipulation
turtlgraph merge bundle1.turtlgraph bundle2.turtlgraph -o merged.turtlgraph
turtlgraph split large.turtlgraph --max-size 100MB -o chunks/chunk_
turtlgraph repack old.turtlgraph --compress=lz4 -o new.turtlgraph
turtlgraph optimize bloated.turtlgraph -o optimized.turtlgraph

# Differential updates
turtlgraph diff old.turtlgraph new.turtlgraph # Show differences
turtlgraph patch old.turtlgraph changes.delta -o updated.turtlgraph
turtlgraph delta old.turtlgraph new.turtlgraph -o changes.delta
```

### Analysis Tools

Deep inspection and analysis capabilities:

```bash
# Size analysis
turtlgraph size bundle.turtlgraph           # Summary
turtlgraph size --detailed bundle.turtlgraph # Per-chunk breakdown
turtlgraph size --format=json bundle.turtlgraph > sizes.json

# Compression analysis
turtlgraph compress-stats bundle.turtlgraph
turtlgraph compress-test bundle.turtlgraph --algorithms all
turtlgraph train-dict samples/*.json -o optimized.zstd

# Dependency analysis
turtlgraph deps bundle.turtlgraph           # Show dependencies
turtlgraph deps --graph bundle.turtlgraph   # Graphviz output
turtlgraph deps --missing bundle.turtlgraph # Find missing deps

# Performance analysis
turtlgraph analyze bundle.turtlgraph        # Optimization suggestions
turtlgraph analyze --profile=mobile bundle.turtlgraph # Platform-specific
turtlgraph hotspots bundle.turtlgraph      # Find performance issues

# Content analysis
turtlgraph stats bundle.turtlgraph         # Statistical analysis
turtlgraph duplicates bundle.turtlgraph    # Find duplicate data
turtlgraph unused bundle.turtlgraph        # Find unreferenced chunks
```

### Development Utilities

Tools for development and debugging:

```bash
# Test data generation
turtlgraph generate --size=1GB --nodes=1M test.turtlgraph
turtlgraph generate --pattern=game-like --assets=10000 realistic.turtlgraph
turtlgraph generate --fuzzer --seed=42 fuzz.turtlgraph

# Debugging tools
turtlgraph dump bundle.turtlgraph           # Human-readable dump
turtlgraph dump --hex bundle.turtlgraph     # Hex dump
turtlgraph dump --chunk=42 bundle.turtlgraph # Specific chunk
turtlgraph trace bundle.turtlgraph          # Load operation trace
turtlgraph validate bundle.turtlgraph       # Validate structure

# Format conversion
turtlgraph convert old-format.dat -o new.turtlgraph
turtlgraph import unity.assets -o unity.turtlgraph
turtlgraph export bundle.turtlgraph --format=gltf -o exported/

# Development server
turtlgraph serve bundle.turtlgraph --port=8080  # HTTP server
turtlgraph watch assets/ --auto-pack          # File watcher
turtlgraph repl bundle.turtlgraph              # Interactive REPL
```

#### Command Implementation Examples

```c
// CLI command structure
typedef struct {
    const char* name;
    const char* description;
    int (*handler)(int argc, char** argv);
    const char* usage;
    hyperdag_cmd_flags flags;
} hyperdag_command;

// Info command implementation
int cmd_info(int argc, char** argv) {
    if (argc < 1) {
        fprintf(stderr, "Usage: turtlgraph info <bundle>\n");
        return 1;
    }
    
    const char* bundle_path = argv[0];
    hyperdag_bundle* bundle = hyperdag_open(bundle_path, HYPERDAG_OPEN_READ);
    if (!bundle) {
        fprintf(stderr, "Error: Failed to open bundle '%s'\n", bundle_path);
        return 1;
    }
    
    // Get bundle statistics
    hyperdag_stats stats;
    hyperdag_get_stats(bundle, &stats);
    
    // Print information
    printf("HyperDAG Bundle Information\n");
    printf("===========================\n");
    printf("File: %s\n", bundle_path);
    printf("Version: %d.%d\n", 
           bundle->header.version_major,
           bundle->header.version_minor);
    printf("Size: %.2f MB\n", stats.total_size / (1024.0 * 1024.0));
    printf("Chunks: %u\n", stats.chunk_count);
    printf("Graphs: %u\n", stats.graph_count);
    printf("Nodes: %u\n", stats.total_nodes);
    printf("Edges: %u\n", stats.total_edges);
    printf("Compression: %s\n", stats.compression_type);
    printf("Compression Ratio: %.2f%%\n", 
           100.0 * (1.0 - (double)stats.compressed_size / stats.uncompressed_size));
    printf("Integrity: %s\n", stats.has_integrity ? "Yes" : "No");
    
    if (stats.has_integrity) {
        printf("Hash Algorithm: %s\n", stats.hash_algorithm);
    }
    
    // Platform-specific info
    if (stats.platform[0]) {
        printf("Platform: %s\n", stats.platform);
    }
    
    if (stats.build_timestamp) {
        char timestamp[64];
        format_timestamp(stats.build_timestamp, timestamp, sizeof(timestamp));
        printf("Built: %s\n", timestamp);
    }
    
    hyperdag_close(bundle);
    return 0;
}

// Tree command with visualization
int cmd_tree(int argc, char** argv) {
    if (argc < 1) {
        fprintf(stderr, "Usage: turtlgraph tree <bundle> [path]\n");
        return 1;
    }
    
    hyperdag_bundle* bundle = hyperdag_open(argv[0], HYPERDAG_OPEN_READ);
    if (!bundle) {
        fprintf(stderr, "Error: Failed to open bundle\n");
        return 1;
    }
    
    const char* start_path = argc > 1 ? argv[1] : "/";
    const hyperdag_graph* root = hyperdag_get_graph_by_path(bundle, start_path);
    
    if (!root) {
        fprintf(stderr, "Error: Path '%s' not found\n", start_path);
        hyperdag_close(bundle);
        return 1;
    }
    
    // Print tree visualization
    print_tree_recursive(root, 0, true, "");
    
    hyperdag_close(bundle);
    return 0;
}

void print_tree_recursive(const hyperdag_graph* node, int depth, 
                         bool is_last, const char* prefix) {
    // Print current node
    printf("%s", prefix);
    if (depth > 0) {
        printf("%s── ", is_last ? "└" : "├");
    }
    
    const char* name = hyperdag_get_node_name(node);
    const char* type = hyperdag_get_node_type(node);
    uint32_t size = hyperdag_get_node_size(node);
    
    printf("%s", name ? name : "<unnamed>");
    if (type) {
        printf(" [%s]", type);
    }
    if (size > 0) {
        char size_str[32];
        format_size(size, size_str, sizeof(size_str));
        printf(" (%s)", size_str);
    }
    printf("\n");
    
    // Update prefix for children
    char child_prefix[1024];
    snprintf(child_prefix, sizeof(child_prefix), "%s%s", 
             prefix, depth > 0 ? (is_last ? "    " : "│   ") : "");
    
    // Print children
    uint32_t child_count = hyperdag_node_count(node);
    for (uint32_t i = 0; i < child_count; i++) {
        const hyperdag_graph* child = hyperdag_get_node(node, i);
        bool child_is_last = (i == child_count - 1);
        print_tree_recursive(child, depth + 1, child_is_last, child_prefix);
    }
}
```

---

## Chapter 14: Testing Framework

TurtlGraph's comprehensive testing framework ensures reliability and performance across all components.

### Unit Testing

Lightweight, fast unit testing framework:

```c
// Test registration and execution
typedef enum {
    TURTLGRAPH_TEST_PASS,
    TURTLGRAPH_TEST_FAIL,
    TURTLGRAPH_TEST_SKIP,
    TURTLGRAPH_TEST_TIMEOUT
} hyperdag_test_result;

typedef struct {
    const char* name;
    const char* suite;
    hyperdag_test_result (*test_func)(void);
    uint64_t execution_time_ns;
    const char* failure_message;
    size_t memory_allocated;
    size_t memory_freed;
} hyperdag_test_case;

// Test fixtures for setup/teardown
typedef struct {
    void* (*setup)(void);
    void (*teardown)(void* context);
    const char* name;
} hyperdag_test_fixture;

// Assertion macros with detailed output
#define TURTLGRAPH_ASSERT_EQ(expected, actual) \
    do { \
        if ((expected) != (actual)) { \
            turtlgraph_test_fail(__FILE__, __LINE__, \
                "Expected: %lld, Actual: %lld", \
                (long long)(expected), (long long)(actual)); \
            return TURTLGRAPH_TEST_FAIL; \
        } \
    } while(0)

#define TURTLGRAPH_ASSERT_STR_EQ(expected, actual) \
    do { \
        if (strcmp((expected), (actual)) != 0) { \
            turtlgraph_test_fail(__FILE__, __LINE__, \
                "Expected: \"%s\", Actual: \"%s\"", \
                (expected), (actual)); \
            return TURTLGRAPH_TEST_FAIL; \
        } \
    } while(0)

#define TURTLGRAPH_ASSERT_NEAR(expected, actual, epsilon) \
    do { \
        double diff = fabs((double)(expected) - (double)(actual)); \
        if (diff > (epsilon)) { \
            turtlgraph_test_fail(__FILE__, __LINE__, \
                "Expected: %f ± %f, Actual: %f (diff: %f)", \
                (double)(expected), (double)(epsilon), \
                (double)(actual), diff); \
            return TURTLGRAPH_TEST_FAIL; \
        } \
    } while(0)

// Example unit tests
hyperdag_test_result test_bundle_creation(void) {
    // Arrange
    hyperdag_builder* builder = hyperdag_builder_create();
    TURTLGRAPH_ASSERT_NOT_NULL(builder);
    
    // Act
    const char* test_string = "Hello, HyperDAG!";
    uint32_t string_id = hyperdag_builder_add_string(builder, test_string);
    
    hyperdag_bundle* bundle = hyperdag_builder_finalize(builder);
    TURTLGRAPH_ASSERT_NOT_NULL(bundle);
    
    // Assert
    const char* retrieved = hyperdag_get_string(bundle, string_id);
    TURTLGRAPH_ASSERT_STR_EQ(test_string, retrieved);
    
    // Verify bundle structure
    TURTLGRAPH_ASSERT_EQ(1, bundle->header.string_count);
    TURTLGRAPH_ASSERT_GT(bundle->header.file_size, 0);
    
    // Cleanup
    hyperdag_close(bundle);
    hyperdag_builder_destroy(builder);
    
    return TURTLGRAPH_TEST_PASS;
}

hyperdag_test_result test_graph_operations(void) {
    hyperdag_test_fixture fixture = {
        .setup = setup_test_bundle,
        .teardown = teardown_test_bundle,
        .name = "GraphOps"
    };
    
    void* context = fixture.setup();
    TURTLGRAPH_ASSERT_NOT_NULL(context);
    
    test_bundle_context* ctx = (test_bundle_context*)context;
    
    // Test node creation
    hyperdag_node_id root = hyperdag_create_node(ctx->bundle);
    TURTLGRAPH_ASSERT_NE(INVALID_NODE_ID, root);
    
    // Test property operations
    hyperdag_set_property(ctx->bundle, root, "name", "root_node");
    const char* name = hyperdag_get_property_string(ctx->bundle, root, "name");
    TURTLGRAPH_ASSERT_STR_EQ("root_node", name);
    
    // Test edge creation
    hyperdag_node_id child = hyperdag_create_node(ctx->bundle);
    hyperdag_edge_id edge = hyperdag_create_edge(ctx->bundle, root, child);
    TURTLGRAPH_ASSERT_NE(INVALID_EDGE_ID, edge);
    
    // Test traversal
    size_t child_count = hyperdag_get_child_count(ctx->bundle, root);
    TURTLGRAPH_ASSERT_EQ(1, child_count);
    
    hyperdag_node_id retrieved_child = hyperdag_get_child(ctx->bundle, root, 0);
    TURTLGRAPH_ASSERT_EQ(child, retrieved_child);
    
    fixture.teardown(context);
    return TURTLGRAPH_TEST_PASS;
}
```

### Integration Testing

End-to-end testing of complete workflows:

```c
// Integration test framework
typedef struct {
    const char* temp_directory;
    char** created_files;
    size_t file_count;
    size_t file_capacity;
    hyperdag_test_fixture fixture;
} hyperdag_integration_context;

// Test complete asset pipeline
hyperdag_test_result test_asset_pipeline_integration(void) {
    hyperdag_integration_context ctx = {0};
    
    // Create temporary directory
    ctx.temp_directory = create_temp_directory("hyperdag_test");
    TURTLGRAPH_ASSERT_NOT_NULL(ctx.temp_directory);
    
    // Create test assets
    create_test_texture(&ctx, "player.png", 1024, 1024);
    create_test_mesh(&ctx, "player.gltf");
    create_test_config(&ctx, "config.json");
    
    // Run asset pipeline
    char command[1024];
    snprintf(command, sizeof(command),
        "turtlgraph pack --compress --verify %s/assets output.turtlgraph",
        ctx.temp_directory);
    
    int result = system(command);
    TURTLGRAPH_ASSERT_EQ(0, result);
    
    // Verify output bundle
    hyperdag_bundle* bundle = hyperdag_open("output.turtlgraph", HYPERDAG_OPEN_READ);
    TURTLGRAPH_ASSERT_NOT_NULL(bundle);
    
    // Check contents
    TURTLGRAPH_ASSERT_TRUE(bundle_contains_asset(bundle, "player.png"));
    TURTLGRAPH_ASSERT_TRUE(bundle_contains_asset(bundle, "player.gltf"));
    TURTLGRAPH_ASSERT_TRUE(bundle_contains_asset(bundle, "config.json"));
    
    // Verify compression
    hyperdag_stats stats;
    hyperdag_get_stats(bundle, &stats);
    TURTLGRAPH_ASSERT_GT(stats.compression_ratio, 0.3); // At least 30% compression
    
    // Test loading performance
    uint64_t start_time = hyperdag_get_time_ns();
    
    for (int i = 0; i < 100; i++) {
        const void* data;
        size_t size;
        hyperdag_result res = hyperdag_get_asset(bundle, "player.png", &data, &size);
        TURTLGRAPH_ASSERT_EQ(HYPERDAG_SUCCESS, res);
        TURTLGRAPH_ASSERT_NOT_NULL(data);
        TURTLGRAPH_ASSERT_GT(size, 0);
    }
    
    uint64_t elapsed = hyperdag_get_time_ns() - start_time;
    double ms_per_load = (elapsed / 100.0) / 1e6;
    TURTLGRAPH_ASSERT_LT(ms_per_load, 1.0); // Less than 1ms per load
    
    // Cleanup
    hyperdag_close(bundle);
    cleanup_integration_context(&ctx);
    
    return TURTLGRAPH_TEST_PASS;
}

// Test hot reload functionality
hyperdag_test_result test_hot_reload_integration(void) {
    // Create initial bundle
    hyperdag_bundle* bundle = create_test_bundle_with_assets();
    TURTLGRAPH_ASSERT_NOT_NULL(bundle);
    
    // Set up file watcher
    hot_reload_context* reload_ctx = hyperdag_hot_reload_create();
    hyperdag_hot_reload_watch(reload_ctx, bundle);
    
    // Modify an asset
    modify_test_asset("assets/texture.png");
    
    // Wait for hot reload
    sleep_ms(100);
    
    // Verify reload occurred
    TURTLGRAPH_ASSERT_TRUE(reload_ctx->reload_count > 0);
    
    // Verify new content is loaded
    const void* texture_data;
    size_t texture_size;
    hyperdag_get_asset(bundle, "texture.png", &texture_data, &texture_size);
    
    uint32_t checksum = compute_checksum(texture_data, texture_size);
    TURTLGRAPH_ASSERT_NE(original_checksum, checksum);
    
    // Cleanup
    hyperdag_hot_reload_destroy(reload_ctx);
    hyperdag_close(bundle);
    
    return TURTLGRAPH_TEST_PASS;
}
```

### Property-Based Testing

Automated testing with generated inputs:

```c
// Property-based testing framework
typedef struct {
    void* (*generate)(hyperdag_rng* rng, size_t size_hint);
    void (*shrink)(void* input, void** smaller_inputs, size_t* count);
    void (*free_input)(void* input);
} hyperdag_generator;

typedef struct {
    bool (*property)(void* input);
    const char* description;
    hyperdag_generator* generator;
    size_t num_tests;
    uint64_t seed;
} hyperdag_property_test;

// Random generators
void* generate_random_bundle_data(hyperdag_rng* rng, size_t size_hint) {
    bundle_test_data* data = calloc(1, sizeof(bundle_test_data));
    
    // Generate random number of strings
    data->string_count = hyperdag_rng_range(rng, 0, 1000);
    data->strings = calloc(data->string_count, sizeof(char*));
    
    for (size_t i = 0; i < data->string_count; i++) {
        size_t len = hyperdag_rng_range(rng, 1, 1024);
        data->strings[i] = generate_random_string(rng, len);
    }
    
    // Generate random blobs
    data->blob_count = hyperdag_rng_range(rng, 0, 100);
    data->blobs = calloc(data->blob_count, sizeof(blob_data));
    
    for (size_t i = 0; i < data->blob_count; i++) {
        data->blobs[i].size = hyperdag_rng_range(rng, 0, size_hint);
        data->blobs[i].data = generate_random_bytes(rng, data->blobs[i].size);
    }
    
    // Generate random graph structure
    data->node_count = hyperdag_rng_range(rng, 1, 100);
    data->edge_probability = hyperdag_rng_float(rng);
    
    return data;
}

// Property: Bundle roundtrip preserves data
bool property_bundle_roundtrip(void* input) {
    bundle_test_data* data = (bundle_test_data*)input;
    
    // Create bundle
    hyperdag_builder* builder = hyperdag_builder_create();
    if (!builder) return false;
    
    // Add all strings
    for (size_t i = 0; i < data->string_count; i++) {
        uint32_t id = hyperdag_builder_add_string(builder, data->strings[i]);
        if (id != i) {
            hyperdag_builder_destroy(builder);
            return false;
        }
    }
    
    // Add all blobs
    for (size_t i = 0; i < data->blob_count; i++) {
        uint32_t id = hyperdag_builder_add_blob(builder, 
            data->blobs[i].data, data->blobs[i].size);
        if (id == INVALID_BLOB_ID) {
            hyperdag_builder_destroy(builder);
            return false;
        }
    }
    
    // Build graph structure
    if (!build_random_graph(builder, data)) {
        hyperdag_builder_destroy(builder);
        return false;
    }
    
    // Finalize bundle
    hyperdag_bundle* bundle = hyperdag_builder_finalize(builder);
    if (!bundle) {
        hyperdag_builder_destroy(builder);
        return false;
    }
    
    // Verify all data preserved
    bool success = verify_bundle_contents(bundle, data);
    
    hyperdag_close(bundle);
    hyperdag_builder_destroy(builder);
    
    return success;
}

// Run property-based tests
void run_property_tests(void) {
    hyperdag_property_test tests[] = {
        {
            .property = property_bundle_roundtrip,
            .description = "Bundle roundtrip preserves all data",
            .generator = &bundle_data_generator,
            .num_tests = 1000,
            .seed = 0  // 0 = random seed
        },
        {
            .property = property_compression_reduces_size,
            .description = "Compression never increases size",
            .generator = &compressible_data_generator,
            .num_tests = 100,
            .seed = 0
        },
        {
            .property = property_graph_acyclic_after_sort,
            .description = "Topological sort produces acyclic ordering",
            .generator = &dag_generator,
            .num_tests = 500,
            .seed = 0
        }
    };
    
    for (size_t i = 0; i < sizeof(tests) / sizeof(tests[0]); i++) {
        run_property_test(&tests[i]);
    }
}
```

### Performance Testing

Comprehensive performance benchmarking:

```c
// Performance test framework
typedef struct {
    const char* name;
    void (*setup)(void* context);
    void (*benchmark)(void* context);
    void (*teardown)(void* context);
    
    // Configuration
    size_t iterations;
    size_t warmup_iterations;
    double time_limit_seconds;
    
    // Results
    double* timings;
    size_t timing_count;
    double mean;
    double median;
    double stddev;
    double p95;
    double p99;
} hyperdag_benchmark;

// Benchmark bundle loading performance
void benchmark_bundle_loading(void* context) {
    benchmark_context* ctx = (benchmark_context*)context;
    
    // Clear caches
    clear_system_caches();
    
    // Time the operation
    uint64_t start = hyperdag_get_time_ns();
    
    hyperdag_bundle* bundle = hyperdag_open(ctx->bundle_path, HYPERDAG_OPEN_READ);
    if (bundle) {
        // Force load of root graph
        const hyperdag_graph* root = hyperdag_root(bundle);
        volatile uint32_t dummy = root->header.node_cnt;
        (void)dummy;
        
        hyperdag_close(bundle);
    }
    
    uint64_t elapsed = hyperdag_get_time_ns() - start;
    record_timing(ctx, elapsed);
}

// Benchmark graph traversal
void benchmark_graph_traversal(void* context) {
    benchmark_context* ctx = (benchmark_context*)context;
    
    uint64_t start = hyperdag_get_time_ns();
    
    size_t nodes_visited = 0;
    traverse_graph_dfs(ctx->bundle, ctx->root, &nodes_visited);
    
    uint64_t elapsed = hyperdag_get_time_ns() - start;
    
    // Record normalized timing (ns per node)
    double ns_per_node = (double)elapsed / nodes_visited;
    record_timing(ctx, ns_per_node);
}

// Run benchmarks and generate report
void run_performance_suite(void) {
    hyperdag_benchmark benchmarks[] = {
        {
            .name = "Bundle Open/Close",
            .setup = setup_bundle_benchmark,
            .benchmark = benchmark_bundle_loading,
            .teardown = teardown_bundle_benchmark,
            .iterations = 1000,
            .warmup_iterations = 100
        },
        {
            .name = "Graph Traversal (DFS)",
            .setup = setup_graph_benchmark,
            .benchmark = benchmark_graph_traversal,
            .teardown = teardown_graph_benchmark,
            .iterations = 10000,
            .warmup_iterations = 1000
        },
        {
            .name = "String Lookup",
            .setup = setup_string_benchmark,
            .benchmark = benchmark_string_lookup,
            .teardown = teardown_string_benchmark,
            .iterations = 100000,
            .warmup_iterations = 10000
        },
        {
            .name = "Memory-Mapped Access",
            .setup = setup_mmap_benchmark,
            .benchmark = benchmark_mmap_access,
            .teardown = teardown_mmap_benchmark,
            .iterations = 1000,
            .warmup_iterations = 100
        }
    };
    
    printf("TurtlGraph Performance Test Suite\n");
    printf("===============================\n\n");
    
    for (size_t i = 0; i < sizeof(benchmarks) / sizeof(benchmarks[0]); i++) {
        run_benchmark(&benchmarks[i]);
        print_benchmark_results(&benchmarks[i]);
    }
    
    // Generate comparison report
    generate_performance_report(benchmarks, 
        sizeof(benchmarks) / sizeof(benchmarks[0]),
        "performance_report.html");
}

// Statistical analysis of results
void analyze_benchmark_results(hyperdag_benchmark* bench) {
    if (bench->timing_count == 0) return;
    
    // Sort timings
    qsort(bench->timings, bench->timing_count, sizeof(double), compare_double);
    
    // Calculate statistics
    bench->mean = calculate_mean(bench->timings, bench->timing_count);
    bench->median = calculate_median(bench->timings, bench->timing_count);
    bench->stddev = calculate_stddev(bench->timings, bench->timing_count, bench->mean);
    bench->p95 = calculate_percentile(bench->timings, bench->timing_count, 0.95);
    bench->p99 = calculate_percentile(bench->timings, bench->timing_count, 0.99);
}
```

### Fuzz Testing

Automated security and robustness testing:

```c
// Fuzz testing harness
typedef struct {
    uint8_t* data;
    size_t size;
    uint32_t seed;
    size_t iteration;
} hyperdag_fuzz_input;

// Main fuzzing entry point
int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Set resource limits
    set_memory_limit(256 * 1024 * 1024);  // 256MB
    set_time_limit(30);  // 30 seconds
    
    // Create temporary file
    char temp_path[] = "/tmp/fuzz_XXXXXX";
    int fd = mkstemp(temp_path);
    if (fd < 0) return 0;
    
    write(fd, data, size);
    close(fd);
    
    // Test bundle parsing
    hyperdag_bundle* bundle = hyperdag_open_secure(temp_path, 
        &hyperdag_security_strict);
    
    if (bundle) {
        // Exercise API surface
        fuzz_test_api(bundle);
        hyperdag_close(bundle);
    }
    
    unlink(temp_path);
    return 0;
}

// Targeted fuzzing for specific components
void fuzz_test_api(hyperdag_bundle* bundle) {
    // Test graph traversal
    const hyperdag_graph* root = hyperdag_root(bundle);
    if (root) {
        fuzz_graph_traversal(root, 0, MAX_DEPTH);
    }
    
    // Test string operations
    for (uint32_t i = 0; i < 100 && i < bundle->header.string_count; i++) {
        const char* str = hyperdag_get_string(bundle, i);
        if (str) {
            volatile size_t len = strlen(str);
            (void)len;
        }
    }
    
    // Test property access
    fuzz_property_access(bundle);
    
    // Test chunk operations
    fuzz_chunk_operations(bundle);
}

// Mutation-based fuzzing
typedef struct {
    size_t pos;
    size_t len;
    enum { MUTATE_FLIP, MUTATE_INSERT, MUTATE_DELETE, MUTATE_REPLACE } type;
    uint8_t data[16];
} mutation;

void mutate_bundle(uint8_t* data, size_t size, hyperdag_rng* rng) {
    size_t num_mutations = hyperdag_rng_range(rng, 1, 10);
    
    for (size_t i = 0; i < num_mutations; i++) {
        mutation mut;
        mut.pos = hyperdag_rng_range(rng, 0, size);
        mut.type = hyperdag_rng_range(rng, 0, 4);
        
        switch (mut.type) {
            case MUTATE_FLIP:
                if (mut.pos < size) {
                    data[mut.pos] ^= 1 << hyperdag_rng_range(rng, 0, 8);
                }
                break;
                
            case MUTATE_INSERT:
                // Insert random bytes
                mut.len = hyperdag_rng_range(rng, 1, 16);
                insert_bytes(data, size, mut.pos, mut.data, mut.len);
                break;
                
            case MUTATE_DELETE:
                // Delete bytes
                mut.len = hyperdag_rng_range(rng, 1, min(16, size - mut.pos));
                delete_bytes(data, size, mut.pos, mut.len);
                break;
                
            case MUTATE_REPLACE:
                // Replace with random data
                mut.len = hyperdag_rng_range(rng, 1, min(16, size - mut.pos));
                for (size_t j = 0; j < mut.len; j++) {
                    if (mut.pos + j < size) {
                        data[mut.pos + j] = hyperdag_rng_byte(rng);
                    }
                }
                break;
        }
    }
}

// Coverage-guided fuzzing
typedef struct {
    uint64_t* coverage_map;
    size_t map_size;
    size_t new_coverage_count;
    hash_table* corpus;
} fuzzer_state;

void coverage_guided_fuzz(fuzzer_state* state, const uint8_t* data, size_t size) {
    // Reset coverage map
    memset(state->coverage_map, 0, state->map_size * sizeof(uint64_t));
    
    // Run with coverage tracking
    __sanitizer_cov_reset_edgeguards();
    
    hyperdag_fuzz_input input = {
        .data = (uint8_t*)data,
        .size = size,
        .seed = random(),
        .iteration = state->iteration++
    };
    
    fuzz_one_input(&input);
    
    // Check for new coverage
    size_t new_edges = count_new_edges(state->coverage_map, state->map_size);
    
    if (new_edges > 0) {
        // Add to corpus for future mutation
        add_to_corpus(state->corpus, data, size);
        state->new_coverage_count += new_edges;
        
        printf("New coverage found! Edges: %zu Total: %zu\n", 
               new_edges, state->new_coverage_count);
    }
}
```

---

### Cross-References
- See [Chapter 11: Platform Abstraction](../volume-1-foundation/part-4-platform-security.md#chapter-11-platform-abstraction-layer) for platform-specific build considerations
- See [Chapter 10: Security & Encryption](../volume-1-foundation/part-4-platform-security.md#chapter-10-security--encryption) for security testing
- See [Chapter 15: Performance Optimization](../volume-3-advanced-future/part-8-performance-optimization.md#chapter-15-performance-optimization) for performance benchmarking
- See [Chapter 22: Editor Integration](../volume-2-development-integration/part-6-integration-migration.md#chapter-22-editor-integration) for IDE integration

### Navigation
[← Volume 1: Foundation & Core Systems](../volume-1-foundation/part-4-platform-security.md) | [Table of Contents](#table-of-contents) | [Part 6: Integration & Migration →](../volume-2-development-integration/part-6-integration-migration.md)