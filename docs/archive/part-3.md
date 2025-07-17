# GRAPHITE Specification - Part 3: Advanced Integration & Implementation

*Continuing from Section 14 (Command Line Interface)*

---

## 15. Asset Pipeline Integration

GRAPHITE is designed for seamless integration into existing asset build pipelines, providing comprehensive hooks for automated bundle generation, dependency tracking, and incremental builds.

### 15.1 Build System Integration

#### 15.1.1 CMake Integration
```cmake
# FindGRAPHITE.cmake module
find_package(GRAPHITE REQUIRED)

# Create bundle target
graphite_add_bundle(MyGame_Assets
    SOURCES ${ASSET_FILES}
    OUTPUT ${CMAKE_BINARY_DIR}/assets/game.graphite
    COMPRESSION zstd
    INTEGRITY blake3
    HOT_RELOAD $<CONFIG:Debug>
)

# Dependency tracking
graphite_add_dependencies(MyGame_Assets
    DEPENDS texture_processor mesh_optimizer
    TRIGGERS ${SHADER_FILES} ${TEXTURE_FILES}
)

# Custom build rules
graphite_add_transform(texture_transform
    INPUT_PATTERN "*.png;*.jpg;*.tga"
    OUTPUT_PATTERN "*.g_tex"
    COMMAND texture_processor --format bc7 --mips --output $output $input
    CACHE_KEY content_hash
)
```

#### 15.1.2 Ninja Integration
```ninja
# Direct ninja rule for GRAPHITE bundles
rule graphite_pack
  command = graphite pack --config $config --output $out $in
  description = Packing GRAPHITE bundle $out
  deps = gcc
  depfile = $out.d

build assets/game.graphite: graphite_pack $assets | graphite_config.json
  config = release

# Incremental rebuild support
rule graphite_incremental
  command = graphite update --bundle $bundle --changed $in
  description = Updating GRAPHITE bundle $bundle
  deps = gcc
  depfile = $bundle.d

build assets/game.graphite: graphite_incremental assets/textures/player.png
  bundle = assets/game.graphite
```

#### 15.1.3 Bazel Integration
```python
# //tools/graphite:defs.bzl
def graphite_bundle(name, srcs, compression="zstd", integrity="blake3", **kwargs):
    native.genrule(
        name = name,
        srcs = srcs,
        outs = [name + ".graphite"],
        cmd = "$(location //tools/graphite:pack) " +
              "--compression=%s --integrity=%s " % (compression, integrity) +
              "--output $@ $(SRCS)",
        tools = ["//tools/graphite:pack"],
        **kwargs
    )

# BUILD file usage
graphite_bundle(
    name = "game_assets",
    srcs = glob(["assets/**/*"]),
    compression = "zstd",
    integrity = "blake3",
)
```

#### 15.1.4 Make Integration
```makefile
# Traditional Makefile support
GRAPHITE := graphite
ASSET_DIR := assets
OUTPUT_DIR := build

%.graphite: $(shell find $(ASSET_DIR) -type f)
	$(GRAPHITE) pack --config release.json --output $@ $(ASSET_DIR)

# Dependency tracking with .d files
include $(wildcard *.d)

game.graphite: assets.d
	$(GRAPHITE) pack --output $@ --depfile $@.d $(ASSET_DIR)
```

### 15.2 Continuous Integration Integration

#### 15.2.1 GitHub Actions
```yaml
name: Build Assets
on: [push, pull_request]

jobs:
  build-assets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Install GRAPHITE
        run: |
          wget https://github.com/graphite/releases/latest/graphite-linux-x64.tar.gz
          tar -xzf graphite-linux-x64.tar.gz
          sudo mv graphite /usr/local/bin/
          
      - name: Build Asset Bundles
        run: |
          graphite pack --config ci.json --output dist/game.graphite assets/
          
      - name: Validate Bundles
        run: |
          graphite verify dist/game.graphite
          graphite benchmark --quick dist/game.graphite
          
      - name: Upload Bundles
        uses: actions/upload-artifact@v4
        with:
          name: asset-bundles
          path: dist/*.graphite
```

#### 15.2.2 Jenkins Pipeline
```groovy
pipeline {
    agent any
    
    stages {
        stage('Build Assets') {
            steps {
                sh '''
                    graphite pack --config jenkins.json \
                        --output build/assets.graphite \
                        --progress \
                        assets/
                '''
            }
        }
        
        stage('Quality Gates') {
            parallel {
                stage('Integrity Check') {
                    steps {
                        sh 'graphite verify build/assets.graphite'
                    }
                }
                stage('Performance Check') {
                    steps {
                        sh '''
                            graphite benchmark build/assets.graphite > perf.txt
                            python3 scripts/check_performance_regression.py perf.txt
                        '''
                    }
                }
                stage('Size Check') {
                    steps {
                        sh '''
                            SIZE=$(stat -c%s build/assets.graphite)
                            if [ $SIZE -gt 104857600 ]; then  # 100MB
                                echo "Bundle size $SIZE exceeds 100MB limit"
                                exit 1
                            fi
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'build/*.graphite', fingerprint: true
            publishTestResults testResultsPattern: 'test-results.xml'
        }
    }
}
```

### 15.3 Asset Dependency Tracking

#### 15.3.1 Dependency Graph Generation
```c
// Automatic dependency extraction during pack
typedef struct {
    const char* asset_path;
    const char** dependencies;
    size_t dependency_count;
    uint64_t content_hash;
    uint64_t dependency_hash;
} graphite_asset_info;

// Plugin interface for custom dependency extractors
typedef struct {
    const char* file_extension;
    graphite_asset_info* (*extract_deps)(const char* file_path);
    void (*free_info)(graphite_asset_info* info);
} graphite_dependency_extractor;

// Built-in extractors
extern const graphite_dependency_extractor graphite_gltf_extractor;
extern const graphite_dependency_extractor graphite_fbx_extractor;
extern const graphite_dependency_extractor graphite_material_extractor;
```

#### 15.3.2 Incremental Build Support
```json
{
  "incremental_config": {
    "cache_directory": ".graphite_cache",
    "hash_algorithm": "blake3",
    "dependency_tracking": true,
    "parallel_analysis": true,
    "max_cache_size_mb": 1024
  },
  "build_rules": [
    {
      "pattern": "*.gltf",
      "dependencies": ["textures", "materials"],
      "cache_key": "content_and_deps"
    },
    {
      "pattern": "*.hlsl",
      "dependencies": ["includes"],
      "cache_key": "content_only"
    }
  ]
}
```

### 15.4 Custom Transform Pipeline

#### 15.4.1 Transform Plugin Architecture
```c
// Transform plugin interface
typedef struct {
    const char* name;
    const char* version;
    const char** input_formats;
    const char** output_formats;
    
    // Transform function
    graphite_result (*transform)(
        const void* input_data,
        size_t input_size,
        const char* input_format,
        void** output_data,
        size_t* output_size,
        const char* output_format,
        const graphite_transform_params* params
    );
    
    // Optional: validate parameters
    bool (*validate_params)(const graphite_transform_params* params);
    
    // Optional: estimate output size
    size_t (*estimate_output_size)(size_t input_size, const graphite_transform_params* params);
} graphite_transform_plugin;

// Registration
void graphite_register_transform(const graphite_transform_plugin* plugin);
```

#### 15.4.2 Transform Configuration
```json
{
  "transforms": [
    {
      "name": "texture_compress",
      "plugin": "texture_tools",
      "input_patterns": ["*.png", "*.jpg", "*.tga"],
      "output_format": "bc7",
      "parameters": {
        "quality": "high",
        "generate_mips": true,
        "max_size": 2048
      }
    },
    {
      "name": "mesh_optimize",
      "plugin": "meshoptimizer",
      "input_patterns": ["*.gltf", "*.fbx"],
      "output_format": "optimized_mesh",
      "parameters": {
        "vertex_cache_optimize": true,
        "overdraw_optimize": true,
        "vertex_fetch_optimize": true
      }
    },
    {
      "name": "shader_compile",
      "plugin": "shader_compiler",
      "input_patterns": ["*.hlsl", "*.glsl"],
      "output_format": "spirv",
      "parameters": {
        "optimization_level": 3,
        "target_vulkan": "1.3",
        "target_dx12": true
      }
    }
  ]
}
```

---

## 16. Content Delivery Network Integration

GRAPHITE provides first-class support for CDN deployment and delta updates, enabling efficient content delivery for live service games and applications.

### 16.1 CDN-Optimized Bundle Structure

#### 16.1.1 Chunk-Based Distribution
```c
// CDN chunk header
typedef struct {
    char magic[4];              // "GCHK"
    uint8_t version;            // Format version
    uint8_t flags;              // Compression, encryption flags
    uint16_t reserved;          // Future use
    uint64_t chunk_id;          // Unique chunk identifier
    uint64_t content_hash;      // BLAKE3 hash of decompressed content
    uint32_t compressed_size;   // Size of compressed data
    uint32_t uncompressed_size; // Size after decompression
    uint64_t bundle_id;         // Parent bundle identifier
    uint32_t chunk_index;       // Index within bundle
    uint32_t total_chunks;      // Total chunks in bundle
} graphite_cdn_chunk_header;

// CDN manifest
typedef struct {
    char magic[4];              // "GMAN"
    uint8_t version;
    uint8_t reserved[3];
    uint64_t bundle_id;
    uint64_t bundle_version;
    uint32_t chunk_count;
    uint32_t total_size;
    graphite_cdn_chunk_info chunks[];
} graphite_cdn_manifest;
```

#### 16.1.2 Delta Update System
```c
// Delta patch header
typedef struct {
    char magic[4];              // "GDLT"
    uint8_t version;
    uint8_t compression;        // zstd, lz4, etc.
    uint16_t flags;
    uint64_t base_version;      // Source bundle version
    uint64_t target_version;    // Destination bundle version
    uint32_t operation_count;   // Number of delta operations
    uint64_t base_hash;         // Hash of base bundle
    uint64_t target_hash;       // Hash of target bundle
} graphite_delta_header;

// Delta operations
typedef enum {
    GRAPHITE_DELTA_COPY,        // Copy from base
    GRAPHITE_DELTA_INSERT,      // Insert new data
    GRAPHITE_DELTA_DELETE,      // Delete range
    GRAPHITE_DELTA_REPLACE      // Replace range
} graphite_delta_operation_type;

typedef struct {
    graphite_delta_operation_type type;
    uint64_t offset;            // Offset in target
    uint32_t length;            // Length of operation
    uint64_t source_offset;     // For COPY operations
    // Followed by inline data for INSERT/REPLACE
} graphite_delta_operation;
```

### 16.2 CDN Deployment Tools

#### 16.2.1 Bundle Chunking
```bash
# Split bundle into CDN-optimized chunks
graphite cdn-split game.graphite \
    --chunk-size 4MB \
    --output-dir cdn_chunks/ \
    --manifest cdn_manifest.json \
    --compression zstd \
    --level 9

# Generate chunk URLs for CDN
graphite cdn-upload-config \
    --manifest cdn_manifest.json \
    --base-url https://cdn.example.com/assets/ \
    --output upload_config.json
```

#### 16.2.2 Delta Generation
```bash
# Generate delta between versions
graphite delta-create \
    --base game_v1.0.graphite \
    --target game_v1.1.graphite \
    --output game_v1.0_to_v1.1.delta \
    --compression zstd \
    --level 9

# Validate delta
graphite delta-verify \
    --base game_v1.0.graphite \
    --delta game_v1.0_to_v1.1.delta \
    --expected game_v1.1.graphite
```

### 16.3 CDN Integration APIs

#### 16.3.1 Chunk Download Manager
```c
// CDN download configuration
typedef struct {
    const char* base_url;
    const char* auth_token;
    uint32_t max_concurrent_downloads;
    uint32_t retry_count;
    uint32_t timeout_seconds;
    bool verify_chunks;
    const char* cache_directory;
} graphite_cdn_config;

// Async chunk downloader
typedef struct graphite_cdn_downloader graphite_cdn_downloader;

graphite_cdn_downloader* graphite_cdn_create(const graphite_cdn_config* config);
void graphite_cdn_destroy(graphite_cdn_downloader* downloader);

// Download bundle with progress tracking
typedef void (*graphite_progress_callback)(
    uint64_t bytes_downloaded,
    uint64_t total_bytes,
    uint32_t chunks_completed,
    uint32_t total_chunks,
    void* user_data
);

graphite_result graphite_cdn_download_bundle(
    graphite_cdn_downloader* downloader,
    const char* manifest_url,
    const char* output_path,
    graphite_progress_callback progress,
    void* user_data
);

// Check for updates
typedef struct {
    uint64_t current_version;
    uint64_t latest_version;
    uint64_t delta_size;
    bool update_available;
    const char* delta_url;
} graphite_update_info;

graphite_result graphite_cdn_check_update(
    graphite_cdn_downloader* downloader,
    const char* current_bundle_path,
    const char* version_check_url,
    graphite_update_info* update_info
);
```

#### 16.3.2 Delta Application
```c
// Apply delta patch
graphite_result graphite_delta_apply(
    const char* base_bundle_path,
    const char* delta_path,
    const char* output_path,
    graphite_progress_callback progress,
    void* user_data
);

// Verify applied delta
graphite_result graphite_delta_verify_result(
    const char* patched_bundle_path,
    const char* expected_hash,
    uint64_t expected_version
);

// Rollback support
graphite_result graphite_delta_create_rollback(
    const char* base_bundle_path,
    const char* updated_bundle_path,
    const char* rollback_delta_path
);
```

### 16.4 CDN Optimization Strategies

#### 16.4.1 Intelligent Chunking
```c
// Asset-aware chunking
typedef struct {
    const char* asset_path;
    uint64_t size;
    uint32_t access_frequency;  // From analytics
    uint32_t dependency_level;  // 0 = leaf, higher = more deps
    bool is_critical;           // Required for initial load
} graphite_asset_metadata;

// Chunking strategy
typedef enum {
    GRAPHITE_CHUNK_UNIFORM,     // Fixed size chunks
    GRAPHITE_CHUNK_ASSET_AWARE, // Group related assets
    GRAPHITE_CHUNK_FREQUENCY,   // Group by access patterns
    GRAPHITE_CHUNK_DEPENDENCY   // Group by dependency levels
} graphite_chunking_strategy;

graphite_result graphite_cdn_optimal_chunking(
    const char* bundle_path,
    const graphite_asset_metadata* metadata,
    size_t metadata_count,
    graphite_chunking_strategy strategy,
    uint32_t target_chunk_size,
    const char* output_dir
);
```

#### 16.4.2 Predictive Prefetching
```json
{
  "prefetch_config": {
    "strategy": "ml_predicted",
    "max_prefetch_mb": 50,
    "prefetch_triggers": [
      {
        "asset_pattern": "levels/*.level",
        "prefetch_radius": 2,
        "prefetch_assets": ["textures", "sounds", "scripts"]
      },
      {
        "user_action": "menu_navigate",
        "prefetch_probability_threshold": 0.7
      }
    ]
  }
}
```

---

## 17. Hot Reload System

GRAPHITE's hot reload system enables real-time asset updates during development and live service updates in production with sub-100ms reload times.

### 17.1 Hot Reload Architecture

#### 17.1.1 Memory-Mapped Hot Reload
```c
// Hot reload context
typedef struct {
    void* base_mapping;         // Current bundle mmap
    size_t mapping_size;        // Size of current mapping
    void* staging_mapping;      // New bundle mmap
    size_t staging_size;        // Size of staging mapping
    atomic_bool reload_pending; // Atomic reload flag
    atomic_uint64_t version;    // Current version counter
} graphite_hot_reload_context;

// Hot reload configuration
typedef struct {
    const char* watch_directory;    // Directory to monitor
    const char* bundle_path;        // Target bundle file
    uint32_t debounce_ms;          // Debounce file changes
    bool atomic_reload;            // Use atomic pointer swap
    bool preserve_handle_state;    // Keep existing handles valid
    graphite_reload_callback callback; // Notification callback
    void* user_data;
} graphite_hot_reload_config;

// Initialize hot reload system
graphite_hot_reload_context* graphite_hot_reload_init(
    const graphite_hot_reload_config* config
);

void graphite_hot_reload_shutdown(graphite_hot_reload_context* context);
```

#### 17.1.2 Atomic Pointer Swapping
```c
// Thread-safe bundle swapping
typedef struct {
    atomic_ptr bundle_ptr;      // Current active bundle
    atomic_ptr staging_ptr;     // Staging bundle for reload
    atomic_uint64_t version;    // Version counter
    rwlock_t reader_lock;       // Reader-writer lock
} graphite_atomic_bundle;

// Reader acquisition (fast path)
static inline const graphite_bundle* graphite_acquire_reader(
    graphite_atomic_bundle* atomic_bundle,
    uint64_t* version_out
) {
    rwlock_read_lock(&atomic_bundle->reader_lock);
    const graphite_bundle* bundle = atomic_load(&atomic_bundle->bundle_ptr);
    *version_out = atomic_load(&atomic_bundle->version);
    return bundle;
}

static inline void graphite_release_reader(
    graphite_atomic_bundle* atomic_bundle
) {
    rwlock_read_unlock(&atomic_bundle->reader_lock);
}

// Writer swap (reload path)
graphite_result graphite_atomic_swap_bundle(
    graphite_atomic_bundle* atomic_bundle,
    graphite_bundle* new_bundle
) {
    rwlock_write_lock(&atomic_bundle->reader_lock);
    
    graphite_bundle* old_bundle = atomic_load(&atomic_bundle->bundle_ptr);
    atomic_store(&atomic_bundle->bundle_ptr, new_bundle);
    atomic_fetch_add(&atomic_bundle->version, 1);
    
    rwlock_write_unlock(&atomic_bundle->reader_lock);
    
    // Schedule old bundle cleanup
    graphite_schedule_cleanup(old_bundle);
    return GRAPHITE_SUCCESS;
}
```

### 17.2 File System Monitoring

#### 17.2.1 Cross-Platform File Watching
```c
// File system event types
typedef enum {
    GRAPHITE_FS_CREATED,
    GRAPHITE_FS_MODIFIED,
    GRAPHITE_FS_DELETED,
    GRAPHITE_FS_MOVED
} graphite_fs_event_type;

typedef struct {
    graphite_fs_event_type type;
    const char* path;
    const char* old_path;   // For move events
    uint64_t timestamp;
} graphite_fs_event;

typedef void (*graphite_fs_callback)(
    const graphite_fs_event* event,
    void* user_data
);

// Platform-specific implementations
#ifdef _WIN32
    // Windows: ReadDirectoryChangesW
    typedef struct {
        HANDLE directory_handle;
        OVERLAPPED overlapped;
        char buffer[8192];
        graphite_fs_callback callback;
        void* user_data;
    } graphite_fs_watcher_win32;
#elif defined(__linux__)
    // Linux: inotify
    typedef struct {
        int inotify_fd;
        int watch_descriptor;
        graphite_fs_callback callback;
        void* user_data;
    } graphite_fs_watcher_linux;
#elif defined(__APPLE__)
    // macOS: FSEvents
    typedef struct {
        FSEventStreamRef stream;
        CFRunLoopRef run_loop;
        graphite_fs_callback callback;
        void* user_data;
    } graphite_fs_watcher_macos;
#endif

// Unified interface
typedef struct graphite_fs_watcher graphite_fs_watcher;

graphite_fs_watcher* graphite_fs_watcher_create(
    const char* path,
    bool recursive,
    graphite_fs_callback callback,
    void* user_data
);

void graphite_fs_watcher_destroy(graphite_fs_watcher* watcher);
```

#### 17.2.2 Debounced Change Detection
```c
// Debounce configuration
typedef struct {
    uint32_t debounce_ms;       // Minimum time between triggers
    uint32_t max_delay_ms;      // Maximum delay before forcing trigger
    bool coalesce_events;       // Merge multiple changes to same file
    const char** ignore_patterns; // Patterns to ignore (*.tmp, *.lock)
} graphite_debounce_config;

// Debounced file watcher
typedef struct {
    graphite_fs_watcher* watcher;
    hashtable* pending_changes;  // path -> timestamp
    timer_t debounce_timer;
    graphite_debounce_config config;
    graphite_fs_callback final_callback;
    void* user_data;
} graphite_debounced_watcher;

// Process debounced changes
static void graphite_debounce_timer_callback(timer_t timer, void* user_data) {
    graphite_debounced_watcher* debounced = user_data;
    uint64_t now = graphite_get_time_ms();
    
    // Check which changes are ready
    hashtable_iterator iter;
    hashtable_iter_init(&iter, debounced->pending_changes);
    
    while (hashtable_iter_next(&iter)) {
        const char* path = iter.key;
        uint64_t change_time = *(uint64_t*)iter.value;
        
        if (now - change_time >= debounced->config.debounce_ms) {
            // Change is debounced, trigger callback
            graphite_fs_event event = {
                .type = GRAPHITE_FS_MODIFIED,
                .path = path,
                .timestamp = change_time
            };
            debounced->final_callback(&event, debounced->user_data);
            
            // Remove from pending
            hashtable_remove(debounced->pending_changes, path);
        }
    }
}
```

### 17.3 Incremental Rebuild

#### 17.3.1 Dependency-Aware Rebuilds
```c
// Dependency tracking for hot reload
typedef struct {
    const char* asset_path;
    uint64_t content_hash;
    uint64_t modification_time;
    const char** dependencies;
    size_t dependency_count;
    bool needs_rebuild;
} graphite_asset_state;

// Hot reload rebuild context
typedef struct {
    hashtable* asset_states;    // path -> graphite_asset_state
    graph* dependency_graph;    // Asset dependency graph
    queue* rebuild_queue;       // Assets pending rebuild
    threadpool* worker_pool;    // Parallel rebuild workers
} graphite_rebuild_context;

// Check if asset needs rebuild
bool graphite_asset_needs_rebuild(
    const graphite_rebuild_context* context,
    const char* asset_path
) {
    graphite_asset_state* state = hashtable_get(context->asset_states, asset_path);
    if (!state) return true;  // Unknown asset, rebuild
    
    // Check file modification time
    struct stat file_stat;
    if (stat(asset_path, &file_stat) != 0) return true;
    
    if (file_stat.st_mtime > state->modification_time) {
        return true;  // File modified
    }
    
    // Check dependencies
    for (size_t i = 0; i < state->dependency_count; i++) {
        if (graphite_asset_needs_rebuild(context, state->dependencies[i])) {
            return true;  // Dependency changed
        }
    }
    
    return false;
}

// Topological sort for rebuild order
static void graphite_schedule_rebuilds(
    graphite_rebuild_context* context,
    const char* changed_asset
) {
    // Mark changed asset and all dependents for rebuild
    queue* to_visit = queue_create();
    queue_push(to_visit, (void*)changed_asset);
    
    while (!queue_empty(to_visit)) {
        const char* asset = queue_pop(to_visit);
        graphite_asset_state* state = hashtable_get(context->asset_states, asset);
        
        if (state && !state->needs_rebuild) {
            state->needs_rebuild = true;
            queue_push(context->rebuild_queue, (void*)asset);
            
            // Find all assets that depend on this one
            graph_node* node = graph_find_node(context->dependency_graph, asset);
            if (node) {
                for (size_t i = 0; i < node->edge_count; i++) {
                    if (node->edges[i].type == DEPENDENCY_EDGE) {
                        queue_push(to_visit, node->edges[i].target->data);
                    }
                }
            }
        }
    }
    
    queue_destroy(to_visit);
}
```

#### 17.3.2 Parallel Asset Processing
```c
// Worker thread for asset rebuilding
typedef struct {
    const char* asset_path;
    graphite_rebuild_context* context;
    graphite_transform_config* transform_config;
} graphite_rebuild_task;

static void* graphite_rebuild_worker(void* arg) {
    graphite_rebuild_task* task = arg;
    
    // Apply transforms to asset
    graphite_result result = graphite_apply_transforms(
        task->asset_path,
        task->transform_config
    );
    
    if (result == GRAPHITE_SUCCESS) {
        // Update asset state
        graphite_asset_state* state = hashtable_get(
            task->context->asset_states,
            task->asset_path
        );
        
        if (state) {
            state->content_hash = graphite_hash_file(task->asset_path);
            state->modification_time = graphite_get_file_mtime(task->asset_path);
            state->needs_rebuild = false;
        }
    }
    
    free(task);
    return NULL;
}

// Process rebuild queue
void graphite_process_rebuild_queue(graphite_rebuild_context* context) {
    while (!queue_empty(context->rebuild_queue)) {
        const char* asset_path = queue_pop(context->rebuild_queue);
        
        // Create rebuild task
        graphite_rebuild_task* task = malloc(sizeof(graphite_rebuild_task));
        task->asset_path = asset_path;
        task->context = context;
        task->transform_config = graphite_get_transform_config(asset_path);
        
        // Submit to thread pool
        threadpool_submit(context->worker_pool, graphite_rebuild_worker, task);
    }
    
    // Wait for all rebuilds to complete
    threadpool_wait(context->worker_pool);
}
```

### 17.4 Runtime Integration

#### 17.4.1 Game Engine Hot Reload Hooks
```c
// Unity integration
typedef struct {
    graphite_hot_reload_context* reload_context;
    UnityEngine_AssetDatabase* asset_db;
    void* asset_cache;
} graphite_unity_hot_reload;

// Hot reload callback for Unity
static void graphite_unity_reload_callback(
    const graphite_bundle* old_bundle,
    const graphite_bundle* new_bundle,
    void* user_data
) {
    graphite_unity_hot_reload* unity_reload = user_data;
    
    // Invalidate Unity asset cache
    UnityEngine_AssetDatabase_Refresh(unity_reload->asset_db);
    
    // Update native plugin asset references
    graphite_unity_update_asset_references(old_bundle, new_bundle);
    
    // Trigger Unity reimport for changed assets
    graphite_unity_trigger_reimport(new_bundle);
}

// Unreal Engine integration
typedef struct {
    graphite_hot_reload_context* reload_context;
    UAssetRegistry* asset_registry;
    FAssetData* cached_assets;
} graphite_unreal_hot_reload;

static void graphite_unreal_reload_callback(
    const graphite_bundle* old_bundle,
    const graphite_bundle* new_bundle,
    void* user_data
) {
    graphite_unreal_hot_reload* unreal_reload = user_data;
    
    // Mark assets for reload in Unreal's asset system
    graphite_unreal_mark_assets_dirty(old_bundle, new_bundle);
    
    // Broadcast asset change notifications
    unreal_reload->asset_registry->AssetRenamed.Broadcast(FAssetData(), FAssetData());
    
    // Force garbage collection of old assets
    graphite_unreal_force_gc();
}
```

#### 17.4.2 Hot Reload Performance Optimization
```c
// Pre-allocated reload buffers
typedef struct {
    void* staging_buffer;       // Pre-allocated staging memory
    size_t staging_size;        // Size of staging buffer
    void* copy_buffer;          // Temporary copy buffer
    size_t copy_size;           // Size of copy buffer
    atomic_bool buffer_in_use;  // Atomic flag for buffer usage
} graphite_reload_buffers;

// Optimize reload for minimal allocation
graphite_result graphite_optimized_reload(
    graphite_hot_reload_context* context,
    const char* new_bundle_path,
    graphite_reload_buffers* buffers
) {
    // Try to use pre-allocated buffers
    if (!atomic_exchange(&buffers->buffer_in_use, true)) {
        // Got the buffers, use them for reload
        graphite_result result = graphite_reload_with_buffers(
            context,
            new_bundle_path,
            buffers->staging_buffer,
            buffers->staging_size
        );
        
        atomic_store(&buffers->buffer_in_use, false);
        return result;
    } else {
        // Buffers in use, fall back to allocation
        return graphite_reload_with_allocation(context, new_bundle_path);
    }
}

// Memory pool for frequent reloads
typedef struct {
    memory_pool* small_pool;    // For headers and metadata
    memory_pool* large_pool;    // For asset data
    memory_pool* string_pool;   // For string interning
} graphite_reload_memory_pools;

// Pool-based reload to minimize fragmentation
graphite_result graphite_pooled_reload(
    graphite_hot_reload_context* context,
    const char* new_bundle_path,
    graphite_reload_memory_pools* pools
) {
    // Use memory pools for allocation-heavy operations
    memory_pool_reset(pools->small_pool);
    memory_pool_reset(pools->string_pool);
    // Keep large_pool for asset data reuse
    
    return graphite_reload_with_pools(context, new_bundle_path, pools);
}
```

This completes the Hot Reload System section. The hot reload architecture provides real-time asset updates with sub-100ms performance through atomic pointer swapping, intelligent dependency tracking, and optimized memory management.

---

## 18. Asset Streaming Architecture

GRAPHITE's streaming system enables efficient loading of large worlds and assets on-demand, with predictive prefetching and memory-aware resource management.

### 18.1 Streaming Core Architecture

#### 18.1.1 Streaming Context Management
```c
// Streaming priority levels
typedef enum {
    GRAPHITE_PRIORITY_CRITICAL = 0,  // Required for current frame
    GRAPHITE_PRIORITY_HIGH = 1,      // Required for next frame
    GRAPHITE_PRIORITY_MEDIUM = 2,    // Required soon
    GRAPHITE_PRIORITY_LOW = 3,       // Background/predictive
    GRAPHITE_PRIORITY_COUNT = 4
} graphite_streaming_priority;

// Streaming request
typedef struct {
    uint64_t asset_id;
    graphite_streaming_priority priority;
    uint64_t request_time;
    uint32_t estimated_size;
    float distance_factor;       // For LOD-based streaming
    void* user_data;
    graphite_stream_callback callback;
} graphite_stream_request;

// Streaming context
typedef struct {
    priority_queue* request_queues[GRAPHITE_PRIORITY_COUNT];
    thread_pool* io_pool;
    thread_pool* decompression_pool;
    memory_pool* streaming_pool;
    
    // Memory management
    size_t max_memory_budget;
    atomic_size_t current_memory_usage;
    lru_cache* asset_cache;
    
    // I/O management
    uint32_t max_concurrent_reads;
    atomic_uint32_t active_reads;
    queue* completion_queue;
    
    // Statistics
    atomic_uint64_t bytes_streamed;
    atomic_uint64_t requests_completed;
    atomic_uint64_t cache_hits;
    atomic_uint64_t cache_misses;
} graphite_streaming_context;
```

#### 18.1.2 Memory Budget Management
```c
// Memory budget configuration
typedef struct {
    size_t total_budget;         // Total memory budget
    size_t critical_reserve;     // Reserved for critical assets
    size_t texture_budget;       // Budget for textures
    size_t mesh_budget;          // Budget for meshes
    size_t audio_budget;         // Budget for audio
    size_t other_budget;         // Budget for other assets
    
    // Eviction policy
    float lru_factor;            // Weight for LRU eviction
    float size_factor;           // Weight for size-based eviction
    float priority_factor;       // Weight for priority-based eviction
} graphite_memory_budget;

// Memory pressure handling
typedef enum {
    GRAPHITE_PRESSURE_NONE,      // Memory usage < 70% budget
    GRAPHITE_PRESSURE_MILD,      // Memory usage 70-85% budget
    GRAPHITE_PRESSURE_MODERATE,  // Memory usage 85-95% budget
    GRAPHITE_PRESSURE_SEVERE     // Memory usage > 95% budget
} graphite_memory_pressure;

// Memory management functions
graphite_memory_pressure graphite_check_memory_pressure(
    const graphite_streaming_context* context
) {
    size_t current = atomic_load(&context->current_memory_usage);
    size_t total = context->memory_budget.total_budget;
    
    float usage_ratio = (float)current / total;
    
    if (usage_ratio < 0.70f) return GRAPHITE_PRESSURE_NONE;
    if (usage_ratio < 0.85f) return GRAPHITE_PRESSURE_MILD;
    if (usage_ratio < 0.95f) return GRAPHITE_PRESSURE_MODERATE;
    return GRAPHITE_PRESSURE_SEVERE;
}

// Eviction strategy
size_t graphite_evict_assets(
    graphite_streaming_context* context,
    size_t target_bytes
) {
    size_t evicted = 0;
    graphite_memory_pressure pressure = graphite_check_memory_pressure(context);
    
    // Adjust eviction aggressiveness based on pressure
    float eviction_threshold = 0.1f;  // 10% base threshold
    switch (pressure) {
        case GRAPHITE_PRESSURE_MILD:     eviction_threshold = 0.15f; break;
        case GRAPHITE_PRESSURE_MODERATE: eviction_threshold = 0.25f; break;
        case GRAPHITE_PRESSURE_SEVERE:   eviction_threshold = 0.40f; break;
    }
    
    // Find candidates for eviction using composite scoring
    dynamic_array* candidates = dynamic_array_create(sizeof(graphite_eviction_candidate));
    
    lru_cache_iterator iter;
    lru_cache_iter_init(&iter, context->asset_cache);
    
    while (lru_cache_iter_next(&iter)) {
        graphite_cached_asset* asset = iter.value;
        
        // Calculate eviction score (higher = more likely to evict)
        float score = 0.0f;
        score += context->memory_budget.lru_factor * asset->lru_score;
        score += context->memory_budget.size_factor * (asset->size / 1024.0f / 1024.0f);
        score += context->memory_budget.priority_factor * (1.0f / (asset->priority + 1));
        
        if (score > eviction_threshold) {
            graphite_eviction_candidate candidate = {
                .asset = asset,
                .score = score
            };
            dynamic_array_push(candidates, &candidate);
        }
    }
    
    // Sort by eviction score
    qsort(candidates->data, candidates->count, sizeof(graphite_eviction_candidate),
          graphite_compare_eviction_candidates);
    
    // Evict assets until target is reached
    for (size_t i = 0; i < candidates->count && evicted < target_bytes; i++) {
        graphite_eviction_candidate* candidate = dynamic_array_get(candidates, i);
        evicted += candidate->asset->size;
        
        // Remove from cache and free memory
        lru_cache_remove(context->asset_cache, &candidate->asset->id);
        graphite_free_asset(candidate->asset);
    }
    
    dynamic_array_destroy(candidates);
    return evicted;
}
```

### 18.2 Predictive Streaming

#### 18.2.1 Machine Learning Prediction
```c
// Streaming prediction model
typedef struct {
    neural_network* prediction_model;   // ML model for prediction
    feature_vector* current_features;   // Current game state features
    sliding_window* access_history;     // Recent asset access patterns
    
    // Prediction configuration
    float prediction_threshold;         // Confidence threshold
    uint32_t prediction_horizon_ms;     // How far ahead to predict
    uint32_t max_predictions;           // Max concurrent predictions
} graphite_prediction_context;

// Feature extraction from game state
typedef struct {
    float player_position[3];           // Player world position
    float player_velocity[3];           // Player movement vector
    float camera_direction[3];          // Camera facing direction
    uint32_t current_level_id;          // Current level/area
    uint32_t current_state;             // Game state (menu, gameplay, etc.)
    float time_in_state;                // Time spent in current state
    uint32_t recent_asset_accesses[16]; // Recently accessed assets
} graphite_game_features;

// Convert game state to ML features
void graphite_extract_features(
    const graphite_game_state* game_state,
    graphite_game_features* features
) {
    // Copy position and movement data
    memcpy(features->player_position, game_state->player_position, sizeof(float) * 3);
    memcpy(features->player_velocity, game_state->player_velocity, sizeof(float) * 3);
    memcpy(features->camera_direction, game_state->camera_direction, sizeof(float) * 3);
    
    // Extract discrete state information
    features->current_level_id = game_state->current_level;
    features->current_state = game_state->state;
    features->time_in_state = game_state->state_time;
    
    // Extract recent access patterns
    sliding_window_get_recent(game_state->access_history, 
                             features->recent_asset_accesses, 16);
}

// Run prediction model
graphite_result graphite_predict_asset_needs(
    graphite_prediction_context* context,
    const graphite_game_features* features,
    graphite_prediction_result* predictions,
    size_t max_predictions
) {
    // Normalize features for neural network
    float normalized_features[64];
    graphite_normalize_features(features, normalized_features);
    
    // Run inference
    float* output = neural_network_predict(context->prediction_model, 
                                          normalized_features);
    
    // Convert network output to asset predictions
    size_t prediction_count = 0;
    for (size_t i = 0; i < context->max_predictions && prediction_count < max_predictions; i++) {
        if (output[i] > context->prediction_threshold) {
            predictions[prediction_count].asset_id = i;
            predictions[prediction_count].confidence = output[i];
            predictions[prediction_count].predicted_time_ms = 
                (uint32_t)(output[i + context->max_predictions] * context->prediction_horizon_ms);
            prediction_count++;
        }
    }
    
    free(output);
    
    // Submit predictions to streaming system
    for (size_t i = 0; i < prediction_count; i++) {
        graphite_stream_request request = {
            .asset_id = predictions[i].asset_id,
            .priority = GRAPHITE_PRIORITY_LOW,
            .request_time = graphite_get_time_ms(),
            .estimated_size = graphite_get_asset_size(predictions[i].asset_id),
            .user_data = NULL,
            .callback = NULL
        };
        
        graphite_submit_stream_request(context->streaming_context, &request);
    }
    
    return GRAPHITE_SUCCESS;
}
```

#### 18.2.2 Heuristic-Based Prediction
```c
// Spatial prediction for open-world games
typedef struct {
    float position[3];          // Player position
    float view_direction[3];    // Camera direction
    float movement_vector[3];   // Movement direction
    float speed;                // Movement speed
    uint32_t area_id;           // Current area/zone
} graphite_spatial_context;

// Predict assets based on spatial movement
void graphite_spatial_prediction(
    const graphite_spatial_context* context,
    const graphite_streaming_context* streaming,
    float prediction_time_seconds
) {
    // Calculate predicted position
    float predicted_pos[3];
    for (int i = 0; i < 3; i++) {
        predicted_pos[i] = context->position[i] + 
                          context->movement_vector[i] * 
                          context->speed * 
                          prediction_time_seconds;
    }
    
    // Find assets in predicted area
    graphite_spatial_query query = {
        .center = predicted_pos,
        .radius = 100.0f,  // 100 meter radius
        .max_results = 50
    };
    
    graphite_asset_list* nearby_assets = graphite_spatial_query_assets(&query);
    
    // Submit streaming requests for nearby assets
    for (size_t i = 0; i < nearby_assets->count; i++) {
        uint64_t asset_id = nearby_assets->assets[i];
        
        // Skip if already loaded
        if (lru_cache_contains(streaming->asset_cache, &asset_id)) {
            continue;
        }
        
        // Calculate priority based on distance and view direction
        float distance = graphite_calculate_distance(predicted_pos, 
                                                    nearby_assets->positions[i]);
        float view_factor = graphite_calculate_view_factor(context->view_direction,
                                                          nearby_assets->positions[i],
                                                          predicted_pos);
        
        graphite_streaming_priority priority = GRAPHITE_PRIORITY_LOW;
        if (distance < 50.0f && view_factor > 0.7f) {
            priority = GRAPHITE_PRIORITY_MEDIUM;
        }
        
        graphite_stream_request request = {
            .asset_id = asset_id,
            .priority = priority,
            .request_time = graphite_get_time_ms(),
            .distance_factor = 1.0f / (distance + 1.0f),
            .user_data = NULL,
            .callback = NULL
        };
        
        graphite_submit_stream_request(streaming, &request);
    }
    
    graphite_free_asset_list(nearby_assets);
}

// Pattern-based prediction
typedef struct {
    uint32_t pattern_id;
    uint32_t trigger_asset;
    uint32_t* predicted_assets;
    size_t predicted_count;
    float confidence;
    uint32_t delay_ms;
} graphite_access_pattern;

// Database of learned access patterns
typedef struct {
    hashtable* patterns;        // trigger_asset -> pattern array
    sliding_window* history;    // Recent access history
    uint32_t pattern_length;    // Length of patterns to learn
} graphite_pattern_database;

// Learn access patterns from history
void graphite_learn_patterns(graphite_pattern_database* db) {
    uint32_t* recent_accesses = sliding_window_get_data(db->history);
    size_t access_count = sliding_window_get_count(db->history);
    
    // Extract patterns of specified length
    for (size_t i = 0; i < access_count - db->pattern_length; i++) {
        uint32_t trigger = recent_accesses[i];
        
        // Find existing pattern or create new one
        dynamic_array* patterns = hashtable_get(db->patterns, &trigger);
        if (!patterns) {
            patterns = dynamic_array_create(sizeof(graphite_access_pattern));
            hashtable_insert(db->patterns, &trigger, patterns);
        }
        
        // Create new pattern
        graphite_access_pattern pattern = {
            .pattern_id = graphite_generate_pattern_id(),
            .trigger_asset = trigger,
            .predicted_assets = malloc(sizeof(uint32_t) * (db->pattern_length - 1)),
            .predicted_count = db->pattern_length - 1,
            .confidence = 1.0f,
            .delay_ms = 1000  // 1 second default delay
        };
        
        // Copy predicted assets
        memcpy(pattern.predicted_assets, 
               &recent_accesses[i + 1], 
               sizeof(uint32_t) * pattern.predicted_count);
        
        // Check if pattern already exists
        bool found_existing = false;
        for (size_t j = 0; j < patterns->count; j++) {
            graphite_access_pattern* existing = dynamic_array_get(patterns, j);
            if (graphite_patterns_match(&pattern, existing)) {
                existing->confidence += 0.1f;  // Increase confidence
                found_existing = true;
                free(pattern.predicted_assets);
                break;
            }
        }
        
        if (!found_existing) {
            dynamic_array_push(patterns, &pattern);
        }
    }
}
```

### 18.3 I/O Optimization

#### 18.3.1 Asynchronous I/O with io_uring
```c
#ifdef __linux__
// Linux-specific io_uring implementation
typedef struct {
    struct io_uring ring;
    struct io_uring_sqe* sqe_pool;
    struct io_uring_cqe* cqe_pool;
    uint32_t queue_depth;
    
    // Request tracking
    hashtable* active_requests;  // sqe -> request mapping
    atomic_uint32_t active_count;
} graphite_uring_context;

// Initialize io_uring for streaming
graphite_result graphite_uring_init(
    graphite_uring_context* context,
    uint32_t queue_depth
) {
    context->queue_depth = queue_depth;
    
    // Initialize io_uring with specified queue depth
    int ret = io_uring_queue_init(queue_depth, &context->ring, 0);
    if (ret < 0) {
        return GRAPHITE_ERROR_IO_INIT;
    }
    
    // Allocate tracking structures
    context->active_requests = hashtable_create(queue_depth * 2, 
                                               hash_ptr, compare_ptr);
    atomic_init(&context->active_count, 0);
    
    return GRAPHITE_SUCCESS;
}

// Submit async read request
graphite_result graphite_uring_read_async(
    graphite_uring_context* context,
    const graphite_stream_request* request,
    int fd,
    void* buffer,
    size_t size,
    off_t offset
) {
    // Get submission queue entry
    struct io_uring_sqe* sqe = io_uring_get_sqe(&context->ring);
    if (!sqe) {
        return GRAPHITE_ERROR_QUEUE_FULL;
    }
    
    // Prepare read operation
    io_uring_prep_read(sqe, fd, buffer, size, offset);
    io_uring_sqe_set_data(sqe, (void*)request);
    
    // Track request
    hashtable_insert(context->active_requests, sqe, (void*)request);
    atomic_fetch_add(&context->active_count, 1);
    
    // Submit to kernel
    int submitted = io_uring_submit(&context->ring);
    if (submitted < 0) {
        hashtable_remove(context->active_requests, sqe);
        atomic_fetch_sub(&context->active_count, 1);
        return GRAPHITE_ERROR_IO_SUBMIT;
    }
    
    return GRAPHITE_SUCCESS;
}

// Process completed I/O operations
uint32_t graphite_uring_process_completions(
    graphite_uring_context* context,
    graphite_streaming_context* streaming
) {
    struct io_uring_cqe* cqe;
    uint32_t completed = 0;
    
    // Process all available completions
    while (io_uring_peek_cqe(&context->ring, &cqe) == 0) {
        graphite_stream_request* request = io_uring_cqe_get_data(cqe);
        
        if (cqe->res >= 0) {
            // Success - process the loaded asset
            graphite_process_loaded_asset(streaming, request, cqe->res);
        } else {
            // Error - handle failure
            graphite_handle_stream_error(streaming, request, cqe->res);
        }
        
        // Clean up tracking
        hashtable_remove(context->active_requests, cqe);
        atomic_fetch_sub(&context->active_count, 1);
        
        // Mark completion as seen
        io_uring_cqe_seen(&context->ring, cqe);
        completed++;
    }
    
    return completed;
}
#endif // __linux__

// Cross-platform async I/O abstraction
typedef struct {
#ifdef __linux__
    graphite_uring_context uring;
#elif defined(_WIN32)
    HANDLE completion_port;
    OVERLAPPED* overlapped_pool;
#elif defined(__APPLE__)
    dispatch_queue_t io_queue;
    dispatch_source_t* source_pool;
#endif
    
    thread_pool* callback_pool;
    atomic_uint32_t active_operations;
} graphite_async_io_context;
```

#### 18.3.2 Smart Prefetching and Caching
```c
// Prefetch strategy configuration
typedef struct {
    uint32_t prefetch_distance;     // How far ahead to prefetch
    uint32_t max_prefetch_size;     // Maximum prefetch buffer size
    float prefetch_threshold;       // Confidence threshold for prefetch
    bool adaptive_prefetch;         // Adapt based on access patterns
    uint32_t prefetch_threads;      // Number of prefetch worker threads
} graphite_prefetch_config;

// Smart prefetch implementation
typedef struct {
    graphite_prefetch_config config;
    thread_pool* prefetch_pool;
    circular_buffer* access_pattern;
    hashtable* prefetch_cache;
    
    // Adaptive parameters
    float hit_rate;                 // Current prefetch hit rate
    uint32_t recent_hits;           // Recent prefetch hits
    uint32_t recent_misses;         // Recent prefetch misses
    uint64_t last_adaptation;       // Last adaptation timestamp
} graphite_smart_prefetch;

// Analyze access patterns and trigger prefetch
void graphite_analyze_and_prefetch(
    graphite_smart_prefetch* prefetch,
    uint64_t accessed_asset_id,
    const graphite_streaming_context* streaming
) {
    // Record access in pattern buffer
    circular_buffer_push(prefetch->access_pattern, &accessed_asset_id);
    
    // Analyze recent pattern
    uint64_t* recent_accesses = circular_buffer_get_data(prefetch->access_pattern);
    size_t access_count = circular_buffer_get_count(prefetch->access_pattern);
    
    if (access_count < 3) return;  // Need at least 3 accesses for pattern
    
    // Look for sequential patterns
    bool is_sequential = true;
    uint64_t stride = recent_accesses[1] - recent_accesses[0];
    
    for (size_t i = 2; i < access_count; i++) {
        uint64_t current_stride = recent_accesses[i] - recent_accesses[i-1];
        if (current_stride != stride) {
            is_sequential = false;
            break;
        }
    }
    
    if (is_sequential) {
        // Prefetch next assets in sequence
        for (uint32_t i = 1; i <= prefetch->config.prefetch_distance; i++) {
            uint64_t predicted_id = accessed_asset_id + (stride * i);
            
            // Check if already cached
            if (!lru_cache_contains(streaming->asset_cache, &predicted_id)) {
                graphite_submit_prefetch_request(prefetch, predicted_id);
            }
        }
    } else {
        // Look for other patterns (spatial, temporal, etc.)
        graphite_analyze_complex_patterns(prefetch, recent_accesses, access_count);
    }
    
    // Adapt prefetch parameters based on hit rate
    if (graphite_should_adapt_prefetch(prefetch)) {
        graphite_adapt_prefetch_parameters(prefetch);
    }
}

// Adaptive prefetch parameter tuning
void graphite_adapt_prefetch_parameters(graphite_smart_prefetch* prefetch) {
    float current_hit_rate = (float)prefetch->recent_hits / 
                            (prefetch->recent_hits + prefetch->recent_misses);
    
    // Update moving average hit rate
    prefetch->hit_rate = 0.9f * prefetch->hit_rate + 0.1f * current_hit_rate;
    
    // Adapt parameters based on hit rate
    if (prefetch->hit_rate > 0.8f) {
        // High hit rate - can be more aggressive
        prefetch->config.prefetch_distance = min(prefetch->config.prefetch_distance + 1, 10);
        prefetch->config.prefetch_threshold *= 0.95f;  // Lower threshold
    } else if (prefetch->hit_rate < 0.4f) {
        // Low hit rate - be more conservative
        prefetch->config.prefetch_distance = max(prefetch->config.prefetch_distance - 1, 1);
        prefetch->config.prefetch_threshold *= 1.05f;  // Higher threshold
    }
    
    // Reset counters
    prefetch->recent_hits = 0;
    prefetch->recent_misses = 0;
    prefetch->last_adaptation = graphite_get_time_ms();
}
```

This completes Section 18 on Asset Streaming Architecture. The system provides intelligent, predictive asset streaming with machine learning-based prediction, memory-aware resource management, and high-performance asynchronous I/O.

---

## 19. Cross-Platform Considerations

GRAPHITE is designed for seamless operation across all major platforms, with careful attention to architecture-specific optimizations and platform limitations.

### 19.1 Platform Abstraction Layer

#### 19.1.1 Core Platform Abstraction
```c
// Platform detection macros
#if defined(_WIN32) || defined(_WIN64)
    #define GRAPHITE_PLATFORM_WINDOWS
#elif defined(__linux__)
    #define GRAPHITE_PLATFORM_LINUX
#elif defined(__APPLE__)
    #include <TargetConditionals.h>
    #if TARGET_OS_MAC
        #define GRAPHITE_PLATFORM_MACOS
    #elif TARGET_OS_IPHONE
        #define GRAPHITE_PLATFORM_IOS
    #endif
#elif defined(__ANDROID__)
    #define GRAPHITE_PLATFORM_ANDROID
#elif defined(__EMSCRIPTEN__)
    #define GRAPHITE_PLATFORM_WEB
#endif

// Architecture detection
#if defined(_M_X64) || defined(__x86_64__)
    #define GRAPHITE_ARCH_X64
#elif defined(_M_IX86) || defined(__i386__)
    #define GRAPHITE_ARCH_X86
#elif defined(_M_ARM64) || defined(__aarch64__)
    #define GRAPHITE_ARCH_ARM64
#elif defined(_M_ARM) || defined(__arm__)
    #define GRAPHITE_ARCH_ARM32
#endif

// Endianness detection
#include <stdint.h>
static inline bool graphite_is_little_endian(void) {
    const uint16_t test = 0x0001;
    return *(const uint8_t*)&test == 1;
}

#define GRAPHITE_LITTLE_ENDIAN (graphite_is_little_endian())
#define GRAPHITE_BIG_ENDIAN (!graphite_is_little_endian())
```

#### 19.1.2 File System Abstraction
```c
// Platform-agnostic file system operations
typedef struct {
    void* handle;               // Platform-specific file handle
    const char* path;           // File path
    uint64_t size;              // File size
    bool is_memory_mapped;      // Whether file is memory-mapped
    void* mapped_memory;        // Memory-mapped region
    size_t mapped_size;         // Size of mapped region
} graphite_file;

// Cross-platform file operations
graphite_result graphite_file_open(
    graphite_file* file,
    const char* path,
    uint32_t flags
);

graphite_result graphite_file_close(graphite_file* file);

graphite_result graphite_file_read(
    graphite_file* file,
    void* buffer,
    size_t size,
    size_t offset,
    size_t* bytes_read
);

graphite_result graphite_file_mmap(
    graphite_file* file,
    size_t offset,
    size_t size,
    void** mapped_ptr
);

graphite_result graphite_file_munmap(
    graphite_file* file,
    void* mapped_ptr,
    size_t size
);

// Platform-specific implementations
#ifdef GRAPHITE_PLATFORM_WINDOWS
graphite_result graphite_file_open_win32(
    graphite_file* file,
    const char* path,
    uint32_t flags
) {
    DWORD access = 0;
    DWORD creation = 0;
    
    if (flags & GRAPHITE_FILE_READ) access |= GENERIC_READ;
    if (flags & GRAPHITE_FILE_WRITE) access |= GENERIC_WRITE;
    
    creation = (flags & GRAPHITE_FILE_CREATE) ? CREATE_ALWAYS : OPEN_EXISTING;
    
    HANDLE handle = CreateFileA(
        path,
        access,
        FILE_SHARE_READ,
        NULL,
        creation,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );
    
    if (handle == INVALID_HANDLE_VALUE) {
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    LARGE_INTEGER file_size;
    if (!GetFileSizeEx(handle, &file_size)) {
        CloseHandle(handle);
        return GRAPHITE_ERROR_FILE_SIZE;
    }
    
    file->handle = handle;
    file->path = path;
    file->size = file_size.QuadPart;
    file->is_memory_mapped = false;
    file->mapped_memory = NULL;
    file->mapped_size = 0;
    
    return GRAPHITE_SUCCESS;
}

graphite_result graphite_file_mmap_win32(
    graphite_file* file,
    size_t offset,
    size_t size,
    void** mapped_ptr
) {
    HANDLE mapping = CreateFileMappingA(
        (HANDLE)file->handle,
        NULL,
        PAGE_READONLY,
        (DWORD)(file->size >> 32),
        (DWORD)(file->size & 0xFFFFFFFF),
        NULL
    );
    
    if (!mapping) {
        return GRAPHITE_ERROR_MMAP_CREATE;
    }
    
    void* mapped = MapViewOfFile(
        mapping,
        FILE_MAP_READ,
        (DWORD)(offset >> 32),
        (DWORD)(offset & 0xFFFFFFFF),
        size
    );
    
    CloseHandle(mapping);
    
    if (!mapped) {
        return GRAPHITE_ERROR_MMAP_VIEW;
    }
    
    *mapped_ptr = mapped;
    file->mapped_memory = mapped;
    file->mapped_size = size;
    file->is_memory_mapped = true;
    
    return GRAPHITE_SUCCESS;
}
#endif // GRAPHITE_PLATFORM_WINDOWS

#ifdef GRAPHITE_PLATFORM_LINUX
graphite_result graphite_file_open_linux(
    graphite_file* file,
    const char* path,
    uint32_t flags
) {
    int open_flags = 0;
    
    if ((flags & GRAPHITE_FILE_READ) && (flags & GRAPHITE_FILE_WRITE)) {
        open_flags = O_RDWR;
    } else if (flags & GRAPHITE_FILE_WRITE) {
        open_flags = O_WRONLY;
    } else {
        open_flags = O_RDONLY;
    }
    
    if (flags & GRAPHITE_FILE_CREATE) {
        open_flags |= O_CREAT | O_TRUNC;
    }
    
    int fd = open(path, open_flags, 0644);
    if (fd < 0) {
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    struct stat st;
    if (fstat(fd, &st) < 0) {
        close(fd);
        return GRAPHITE_ERROR_FILE_SIZE;
    }
    
    file->handle = (void*)(intptr_t)fd;
    file->path = path;
    file->size = st.st_size;
    file->is_memory_mapped = false;
    file->mapped_memory = NULL;
    file->mapped_size = 0;
    
    return GRAPHITE_SUCCESS;
}

graphite_result graphite_file_mmap_linux(
    graphite_file* file,
    size_t offset,
    size_t size,
    void** mapped_ptr
) {
    int fd = (int)(intptr_t)file->handle;
    
    void* mapped = mmap(
        NULL,
        size,
        PROT_READ,
        MAP_PRIVATE,
        fd,
        offset
    );
    
    if (mapped == MAP_FAILED) {
        return GRAPHITE_ERROR_MMAP_FAILED;
    }
    
    // Advise kernel about access pattern
    madvise(mapped, size, MADV_SEQUENTIAL | MADV_WILLNEED);
    
    *mapped_ptr = mapped;
    file->mapped_memory = mapped;
    file->mapped_size = size;
    file->is_memory_mapped = true;
    
    return GRAPHITE_SUCCESS;
}
#endif // GRAPHITE_PLATFORM_LINUX
```

### 19.2 Memory Management

#### 19.2.1 Platform-Specific Memory Allocation
```c
// Memory alignment requirements per platform
#ifdef GRAPHITE_ARCH_X64
    #define GRAPHITE_CACHE_LINE_SIZE 64
    #define GRAPHITE_PAGE_SIZE 4096
#elif defined(GRAPHITE_ARCH_ARM64)
    #define GRAPHITE_CACHE_LINE_SIZE 64
    #define GRAPHITE_PAGE_SIZE 4096  // Can be 16KB on some systems
#elif defined(GRAPHITE_ARCH_X86)
    #define GRAPHITE_CACHE_LINE_SIZE 64
    #define GRAPHITE_PAGE_SIZE 4096
#elif defined(GRAPHITE_ARCH_ARM32)
    #define GRAPHITE_CACHE_LINE_SIZE 32
    #define GRAPHITE_PAGE_SIZE 4096
#endif

// Aligned allocation wrapper
static inline void* graphite_aligned_alloc(size_t size, size_t alignment) {
#ifdef GRAPHITE_PLATFORM_WINDOWS
    return _aligned_malloc(size, alignment);
#elif defined(GRAPHITE_PLATFORM_LINUX) || defined(GRAPHITE_PLATFORM_MACOS)
    void* ptr;
    if (posix_memalign(&ptr, alignment, size) == 0) {
        return ptr;
    }
    return NULL;
#else
    // Fallback for platforms without aligned allocation
    void* ptr = malloc(size + alignment - 1);
    if (!ptr) return NULL;
    
    uintptr_t addr = (uintptr_t)ptr;
    uintptr_t aligned_addr = (addr + alignment - 1) & ~(alignment - 1);
    return (void*)aligned_addr;
#endif
}

static inline void graphite_aligned_free(void* ptr) {
#ifdef GRAPHITE_PLATFORM_WINDOWS
    _aligned_free(ptr);
#else
    free(ptr);
#endif
}

// Large page allocation for performance-critical areas
graphite_result graphite_alloc_large_pages(
    void** ptr,
    size_t size
) {
#ifdef GRAPHITE_PLATFORM_WINDOWS
    // Windows: VirtualAlloc with MEM_LARGE_PAGES
    *ptr = VirtualAlloc(
        NULL,
        size,
        MEM_COMMIT | MEM_RESERVE | MEM_LARGE_PAGES,
        PAGE_READWRITE
    );
    return *ptr ? GRAPHITE_SUCCESS : GRAPHITE_ERROR_ALLOCATION;
    
#elif defined(GRAPHITE_PLATFORM_LINUX)
    // Linux: mmap with MAP_HUGETLB
    *ptr = mmap(
        NULL,
        size,
        PROT_READ | PROT_WRITE,
        MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB,
        -1,
        0
    );
    return (*ptr != MAP_FAILED) ? GRAPHITE_SUCCESS : GRAPHITE_ERROR_ALLOCATION;
    
#elif defined(GRAPHITE_PLATFORM_MACOS)
    // macOS: vm_allocate with superpage support
    vm_address_t address = 0;
    kern_return_t result = vm_allocate(
        mach_task_self(),
        &address,
        size,
        VM_FLAGS_ANYWHERE | VM_FLAGS_SUPERPAGE_SIZE_2MB
    );
    *ptr = (void*)address;
    return (result == KERN_SUCCESS) ? GRAPHITE_SUCCESS : GRAPHITE_ERROR_ALLOCATION;
    
#else
    // Fallback to regular allocation
    *ptr = graphite_aligned_alloc(size, GRAPHITE_PAGE_SIZE);
    return *ptr ? GRAPHITE_SUCCESS : GRAPHITE_ERROR_ALLOCATION;
#endif
}
```

#### 19.2.2 NUMA-Aware Memory Management
```c
// NUMA topology detection
typedef struct {
    uint32_t node_count;
    uint32_t cpu_count;
    uint32_t* cpu_to_node;      // CPU ID -> NUMA node mapping
    size_t* node_memory;        // Available memory per node
} graphite_numa_topology;

#ifdef GRAPHITE_PLATFORM_LINUX
#include <numa.h>
#include <numaif.h>

graphite_result graphite_detect_numa_topology(
    graphite_numa_topology* topology
) {
    if (numa_available() < 0) {
        // NUMA not available, use single node
        topology->node_count = 1;
        topology->cpu_count = sysconf(_SC_NPROCESSORS_ONLN);
        topology->cpu_to_node = malloc(sizeof(uint32_t) * topology->cpu_count);
        topology->node_memory = malloc(sizeof(size_t));
        
        // All CPUs on node 0
        for (uint32_t i = 0; i < topology->cpu_count; i++) {
            topology->cpu_to_node[i] = 0;
        }
        
        topology->node_memory[0] = sysconf(_SC_PHYS_PAGES) * sysconf(_SC_PAGE_SIZE);
        return GRAPHITE_SUCCESS;
    }
    
    topology->node_count = numa_max_node() + 1;
    topology->cpu_count = sysconf(_SC_NPROCESSORS_ONLN);
    topology->cpu_to_node = malloc(sizeof(uint32_t) * topology->cpu_count);
    topology->node_memory = malloc(sizeof(size_t) * topology->node_count);
    
    // Map CPUs to NUMA nodes
    for (uint32_t cpu = 0; cpu < topology->cpu_count; cpu++) {
        topology->cpu_to_node[cpu] = numa_node_of_cpu(cpu);
    }
    
    // Get memory information per node
    for (uint32_t node = 0; node < topology->node_count; node++) {
        long long free_mem;
        long long total_mem = numa_node_size64(node, &free_mem);
        topology->node_memory[node] = total_mem;
    }
    
    return GRAPHITE_SUCCESS;
}

// NUMA-aware allocation
void* graphite_numa_alloc(size_t size, uint32_t preferred_node) {
    if (numa_available() < 0) {
        return malloc(size);
    }
    
    // Allocate on preferred NUMA node
    return numa_alloc_onnode(size, preferred_node);
}

void graphite_numa_free(void* ptr, size_t size) {
    if (numa_available() < 0) {
        free(ptr);
    } else {
        numa_free(ptr, size);
    }
}
#endif // GRAPHITE_PLATFORM_LINUX

// Cross-platform worker thread affinity
graphite_result graphite_set_thread_affinity(
    pthread_t thread,
    uint32_t cpu_id
) {
#ifdef GRAPHITE_PLATFORM_LINUX
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);
    return pthread_setaffinity_np(thread, sizeof(cpuset), &cpuset) == 0 ?
           GRAPHITE_SUCCESS : GRAPHITE_ERROR_AFFINITY;
           
#elif defined(GRAPHITE_PLATFORM_WINDOWS)
    DWORD_PTR mask = 1ULL << cpu_id;
    return SetThreadAffinityMask(GetCurrentThread(), mask) != 0 ?
           GRAPHITE_SUCCESS : GRAPHITE_ERROR_AFFINITY;
           
#elif defined(GRAPHITE_PLATFORM_MACOS)
    // macOS: Use thread affinity policy
    thread_affinity_policy_data_t policy = { cpu_id };
    return thread_policy_set(
        pthread_mach_thread_np(thread),
        THREAD_AFFINITY_POLICY,
        (thread_policy_t)&policy,
        THREAD_AFFINITY_POLICY_COUNT
    ) == KERN_SUCCESS ? GRAPHITE_SUCCESS : GRAPHITE_ERROR_AFFINITY;
    
#else
    // Platform doesn't support thread affinity
    return GRAPHITE_ERROR_NOT_SUPPORTED;
#endif
}
```

### 19.3 SIMD Optimization

#### 19.3.1 Cross-Platform SIMD Abstraction
```c
// SIMD capability detection
typedef struct {
    bool sse2_available;
    bool sse4_available;
    bool avx_available;
    bool avx2_available;
    bool avx512_available;
    bool neon_available;
    bool neon64_available;
} graphite_simd_caps;

static graphite_simd_caps g_simd_caps = {0};

void graphite_detect_simd_capabilities(void) {
#ifdef GRAPHITE_ARCH_X64
    // x86/x64 CPUID detection
    uint32_t eax, ebx, ecx, edx;
    
    // Check for SSE2 (standard on x64)
    g_simd_caps.sse2_available = true;
    
    // Check for SSE4.1
    __cpuid_count(1, 0, eax, ebx, ecx, edx);
    g_simd_caps.sse4_available = (ecx & (1 << 19)) != 0;
    
    // Check for AVX
    g_simd_caps.avx_available = (ecx & (1 << 28)) != 0;
    
    // Check for AVX2
    __cpuid_count(7, 0, eax, ebx, ecx, edx);
    g_simd_caps.avx2_available = (ebx & (1 << 5)) != 0;
    
    // Check for AVX-512
    g_simd_caps.avx512_available = (ebx & (1 << 16)) != 0;
    
#elif defined(GRAPHITE_ARCH_ARM64)
    // ARM64: NEON is standard
    g_simd_caps.neon_available = true;
    g_simd_caps.neon64_available = true;
    
#elif defined(GRAPHITE_ARCH_ARM32)
    // ARM32: Check for NEON availability
    #ifdef __ARM_NEON__
        g_simd_caps.neon_available = true;
    #endif
#endif
}

// Cross-platform SIMD vector types
#ifdef GRAPHITE_ARCH_X64
    typedef __m128i graphite_vec128i;
    typedef __m128 graphite_vec128f;
    typedef __m256i graphite_vec256i;
    typedef __m256 graphite_vec256f;
    
    #define GRAPHITE_VEC128_LOAD(ptr) _mm_loadu_si128((const __m128i*)(ptr))
    #define GRAPHITE_VEC128_STORE(ptr, vec) _mm_storeu_si128((__m128i*)(ptr), (vec))
    #define GRAPHITE_VEC128_ADD(a, b) _mm_add_epi32((a), (b))
    
#elif defined(GRAPHITE_ARCH_ARM64)
    typedef int32x4_t graphite_vec128i;
    typedef float32x4_t graphite_vec128f;
    
    #define GRAPHITE_VEC128_LOAD(ptr) vld1q_s32((const int32_t*)(ptr))
    #define GRAPHITE_VEC128_STORE(ptr, vec) vst1q_s32((int32_t*)(ptr), (vec))
    #define GRAPHITE_VEC128_ADD(a, b) vaddq_s32((a), (b))
    
#else
    // Fallback: scalar operations
    typedef struct { int32_t data[4]; } graphite_vec128i;
    typedef struct { float data[4]; } graphite_vec128f;
    
    static inline graphite_vec128i GRAPHITE_VEC128_LOAD(const void* ptr) {
        graphite_vec128i result;
        memcpy(result.data, ptr, sizeof(result.data));
        return result;
    }
    
    static inline void GRAPHITE_VEC128_STORE(void* ptr, graphite_vec128i vec) {
        memcpy(ptr, vec.data, sizeof(vec.data));
    }
    
    static inline graphite_vec128i GRAPHITE_VEC128_ADD(graphite_vec128i a, graphite_vec128i b) {
        graphite_vec128i result;
        for (int i = 0; i < 4; i++) {
            result.data[i] = a.data[i] + b.data[i];
        }
        return result;
    }
#endif
```

#### 19.3.2 Optimized CRC32 Implementation
```c
// Platform-specific CRC32 implementations
uint32_t graphite_crc32_simd(const void* data, size_t length, uint32_t crc) {
    const uint8_t* bytes = (const uint8_t*)data;
    
#ifdef GRAPHITE_ARCH_X64
    if (g_simd_caps.sse4_available) {
        // Use hardware CRC32 instruction on x86/x64
        while (length >= 8) {
            crc = _mm_crc32_u64(crc, *(const uint64_t*)bytes);
            bytes += 8;
            length -= 8;
        }
        
        while (length >= 4) {
            crc = _mm_crc32_u32(crc, *(const uint32_t*)bytes);
            bytes += 4;
            length -= 4;
        }
        
        while (length > 0) {
            crc = _mm_crc32_u8(crc, *bytes);
            bytes++;
            length--;
        }
        
        return crc;
    }
    
#elif defined(GRAPHITE_ARCH_ARM64)
    if (g_simd_caps.neon64_available) {
        // Use ARM64 CRC32 instructions
        while (length >= 8) {
            crc = __crc32d(crc, *(const uint64_t*)bytes);
            bytes += 8;
            length -= 8;
        }
        
        while (length >= 4) {
            crc = __crc32w(crc, *(const uint32_t*)bytes);
            bytes += 4;
            length -= 4;
        }
        
        while (length > 0) {
            crc = __crc32b(crc, *bytes);
            bytes++;
            length--;
        }
        
        return crc;
    }
#endif
    
    // Fallback to software CRC32
    return graphite_crc32_software(data, length, crc);
}

// Vectorized memory copy
void graphite_memcpy_simd(void* dst, const void* src, size_t size) {
#ifdef GRAPHITE_ARCH_X64
    if (g_simd_caps.avx2_available && size >= 32) {
        uint8_t* d = (uint8_t*)dst;
        const uint8_t* s = (const uint8_t*)src;
        
        // Copy 32-byte chunks with AVX2
        while (size >= 32) {
            __m256i data = _mm256_loadu_si256((const __m256i*)s);
            _mm256_storeu_si256((__m256i*)d, data);
            d += 32;
            s += 32;
            size -= 32;
        }
        
        // Handle remaining bytes
        memcpy(d, s, size);
        return;
    }
    
    if (g_simd_caps.sse2_available && size >= 16) {
        uint8_t* d = (uint8_t*)dst;
        const uint8_t* s = (const uint8_t*)src;
        
        // Copy 16-byte chunks with SSE2
        while (size >= 16) {
            __m128i data = _mm_loadu_si128((const __m128i*)s);
            _mm_storeu_si128((__m128i*)d, data);
            d += 16;
            s += 16;
            size -= 16;
        }
        
        // Handle remaining bytes
        memcpy(d, s, size);
        return;
    }
    
#elif defined(GRAPHITE_ARCH_ARM64)
    if (g_simd_caps.neon64_available && size >= 16) {
        uint8_t* d = (uint8_t*)dst;
        const uint8_t* s = (const uint8_t*)src;
        
        // Copy 16-byte chunks with NEON
        while (size >= 16) {
            uint8x16_t data = vld1q_u8(s);
            vst1q_u8(d, data);
            d += 16;
            s += 16;
            size -= 16;
        }
        
        // Handle remaining bytes
        memcpy(d, s, size);
        return;
    }
#endif
    
    // Fallback to standard memcpy
    memcpy(dst, src, size);
}
```

This completes Section 19 on Cross-Platform Considerations, covering platform detection, file system abstraction, memory management, and SIMD optimization across Windows, Linux, macOS, and mobile platforms.

Would you like me to continue with the remaining sections?