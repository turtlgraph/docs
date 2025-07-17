# GRAPHITE Specification - Part 4: Performance, Security & Production

*Continuing from Section 19 (Cross-Platform Considerations)*

---

## 20. Performance Benchmarks & Testing

GRAPHITE includes comprehensive performance testing frameworks to ensure consistent performance across platforms, workloads, and hardware configurations.

### 20.1 Benchmarking Framework

#### 20.1.1 Core Benchmark Infrastructure
```c
// Benchmark context and measurement
typedef struct {
    const char* name;
    const char* description;
    uint64_t iterations;
    uint64_t warmup_iterations;
    bool measure_memory;
    bool measure_cpu;
    bool measure_io;
    void* user_data;
} graphite_benchmark_config;

typedef struct {
    uint64_t min_time_ns;
    uint64_t max_time_ns;
    uint64_t mean_time_ns;
    uint64_t median_time_ns;
    uint64_t p95_time_ns;
    uint64_t p99_time_ns;
    uint64_t std_dev_ns;
    
    size_t min_memory_bytes;
    size_t max_memory_bytes;
    size_t mean_memory_bytes;
    
    uint64_t cpu_cycles;
    uint64_t cache_misses;
    uint64_t branch_misses;
    uint64_t page_faults;
    
    uint64_t bytes_read;
    uint64_t bytes_written;
    uint32_t io_operations;
} graphite_benchmark_result;

// High-precision timing
static inline uint64_t graphite_get_precise_time_ns(void) {
#ifdef GRAPHITE_PLATFORM_WINDOWS
    LARGE_INTEGER freq, counter;
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&counter);
    return (counter.QuadPart * 1000000000ULL) / freq.QuadPart;
#elif defined(GRAPHITE_PLATFORM_LINUX) || defined(GRAPHITE_PLATFORM_MACOS)
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000000000ULL + ts.tv_nsec;
#else
    // Fallback to microsecond precision
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000000000ULL + tv.tv_usec * 1000ULL;
#endif
}

// CPU cycle counting
static inline uint64_t graphite_get_cpu_cycles(void) {
#ifdef GRAPHITE_ARCH_X64
    return __rdtsc();
#elif defined(GRAPHITE_ARCH_ARM64)
    uint64_t cycles;
    asm volatile("mrs %0, cntvct_el0" : "=r"(cycles));
    return cycles;
#else
    // Fallback to time-based estimation
    static uint64_t cycles_per_ns = 0;
    if (cycles_per_ns == 0) {
        // Calibrate once (rough approximation)
        cycles_per_ns = 3; // Assume 3GHz average
    }
    return graphite_get_precise_time_ns() * cycles_per_ns;
#endif
}

// Memory usage tracking
typedef struct {
    size_t peak_rss;
    size_t current_rss;
    size_t heap_allocated;
    size_t heap_freed;
    size_t mmap_allocated;
    size_t mmap_freed;
} graphite_memory_stats;

graphite_memory_stats graphite_get_memory_stats(void) {
    graphite_memory_stats stats = {0};
    
#ifdef GRAPHITE_PLATFORM_LINUX
    FILE* statm = fopen("/proc/self/statm", "r");
    if (statm) {
        unsigned long size, resident, shared, text, lib, data, dt;
        if (fscanf(statm, "%lu %lu %lu %lu %lu %lu %lu", 
                   &size, &resident, &shared, &text, &lib, &data, &dt) == 7) {
            long page_size = sysconf(_SC_PAGESIZE);
            stats.current_rss = resident * page_size;
        }
        fclose(statm);
    }
    
    FILE* status = fopen("/proc/self/status", "r");
    if (status) {
        char line[256];
        while (fgets(line, sizeof(line), status)) {
            if (strncmp(line, "VmHWM:", 6) == 0) {
                sscanf(line + 6, "%zu", &stats.peak_rss);
                stats.peak_rss *= 1024; // Convert from KB to bytes
                break;
            }
        }
        fclose(status);
    }
    
#elif defined(GRAPHITE_PLATFORM_MACOS)
    struct rusage usage;
    if (getrusage(RUSAGE_SELF, &usage) == 0) {
        stats.peak_rss = usage.ru_maxrss;
        stats.current_rss = usage.ru_maxrss; // macOS doesn't provide current RSS easily
    }
    
#elif defined(GRAPHITE_PLATFORM_WINDOWS)
    PROCESS_MEMORY_COUNTERS_EX pmc;
    if (GetProcessMemoryInfo(GetCurrentProcess(), (PROCESS_MEMORY_COUNTERS*)&pmc, sizeof(pmc))) {
        stats.current_rss = pmc.WorkingSetSize;
        stats.peak_rss = pmc.PeakWorkingSetSize;
    }
#endif
    
    return stats;
}
```

#### 20.1.2 Performance Counter Integration
```c
// Hardware performance counters (Linux perf_event)
#ifdef GRAPHITE_PLATFORM_LINUX
#include <linux/perf_event.h>
#include <sys/syscall.h>

typedef struct {
    int cache_misses_fd;
    int cache_references_fd;
    int branch_misses_fd;
    int branch_instructions_fd;
    int page_faults_fd;
    int context_switches_fd;
} graphite_perf_counters;

static long perf_event_open(struct perf_event_attr *hw_event, pid_t pid,
                           int cpu, int group_fd, unsigned long flags) {
    return syscall(__NR_perf_event_open, hw_event, pid, cpu, group_fd, flags);
}

graphite_result graphite_init_perf_counters(graphite_perf_counters* counters) {
    struct perf_event_attr pe = {0};
    pe.type = PERF_TYPE_HARDWARE;
    pe.size = sizeof(struct perf_event_attr);
    pe.disabled = 1;
    pe.exclude_kernel = 1;
    pe.exclude_hv = 1;
    
    // Cache misses
    pe.config = PERF_COUNT_HW_CACHE_MISSES;
    counters->cache_misses_fd = perf_event_open(&pe, 0, -1, -1, 0);
    
    // Cache references
    pe.config = PERF_COUNT_HW_CACHE_REFERENCES;
    counters->cache_references_fd = perf_event_open(&pe, 0, -1, -1, 0);
    
    // Branch misses
    pe.config = PERF_COUNT_HW_BRANCH_MISSES;
    counters->branch_misses_fd = perf_event_open(&pe, 0, -1, -1, 0);
    
    // Branch instructions
    pe.config = PERF_COUNT_HW_BRANCH_INSTRUCTIONS;
    counters->branch_instructions_fd = perf_event_open(&pe, 0, -1, -1, 0);
    
    // Page faults
    pe.type = PERF_TYPE_SOFTWARE;
    pe.config = PERF_COUNT_SW_PAGE_FAULTS;
    counters->page_faults_fd = perf_event_open(&pe, 0, -1, -1, 0);
    
    // Context switches
    pe.config = PERF_COUNT_SW_CONTEXT_SWITCHES;
    counters->context_switches_fd = perf_event_open(&pe, 0, -1, -1, 0);
    
    return GRAPHITE_SUCCESS;
}

void graphite_start_perf_counters(graphite_perf_counters* counters) {
    ioctl(counters->cache_misses_fd, PERF_EVENT_IOC_RESET, 0);
    ioctl(counters->cache_references_fd, PERF_EVENT_IOC_RESET, 0);
    ioctl(counters->branch_misses_fd, PERF_EVENT_IOC_RESET, 0);
    ioctl(counters->branch_instructions_fd, PERF_EVENT_IOC_RESET, 0);
    ioctl(counters->page_faults_fd, PERF_EVENT_IOC_RESET, 0);
    ioctl(counters->context_switches_fd, PERF_EVENT_IOC_RESET, 0);
    
    ioctl(counters->cache_misses_fd, PERF_EVENT_IOC_ENABLE, 0);
    ioctl(counters->cache_references_fd, PERF_EVENT_IOC_ENABLE, 0);
    ioctl(counters->branch_misses_fd, PERF_EVENT_IOC_ENABLE, 0);
    ioctl(counters->branch_instructions_fd, PERF_EVENT_IOC_ENABLE, 0);
    ioctl(counters->page_faults_fd, PERF_EVENT_IOC_ENABLE, 0);
    ioctl(counters->context_switches_fd, PERF_EVENT_IOC_ENABLE, 0);
}

void graphite_stop_perf_counters(graphite_perf_counters* counters,
                                graphite_benchmark_result* result) {
    ioctl(counters->cache_misses_fd, PERF_EVENT_IOC_DISABLE, 0);
    ioctl(counters->cache_references_fd, PERF_EVENT_IOC_DISABLE, 0);
    ioctl(counters->branch_misses_fd, PERF_EVENT_IOC_DISABLE, 0);
    ioctl(counters->branch_instructions_fd, PERF_EVENT_IOC_DISABLE, 0);
    ioctl(counters->page_faults_fd, PERF_EVENT_IOC_DISABLE, 0);
    ioctl(counters->context_switches_fd, PERF_EVENT_IOC_DISABLE, 0);
    
    long long cache_misses, branch_misses, page_faults;
    read(counters->cache_misses_fd, &cache_misses, sizeof(cache_misses));
    read(counters->branch_misses_fd, &branch_misses, sizeof(branch_misses));
    read(counters->page_faults_fd, &page_faults, sizeof(page_faults));
    
    result->cache_misses = cache_misses;
    result->branch_misses = branch_misses;
    result->page_faults = page_faults;
}
#endif // GRAPHITE_PLATFORM_LINUX
```

### 20.2 Standard Benchmark Suite

#### 20.2.1 Core Operation Benchmarks
```c
// Bundle open/close benchmarks
void benchmark_bundle_open(const char* bundle_path, graphite_benchmark_result* result) {
    graphite_benchmark_config config = {
        .name = "bundle_open",
        .description = "Time to open and validate a GRAPHITE bundle",
        .iterations = 1000,
        .warmup_iterations = 100,
        .measure_memory = true,
        .measure_cpu = true,
        .measure_io = true
    };
    
    graphite_perf_counters counters;
    graphite_init_perf_counters(&counters);
    
    uint64_t* times = malloc(sizeof(uint64_t) * config.iterations);
    graphite_memory_stats initial_memory = graphite_get_memory_stats();
    
    // Warmup
    for (uint64_t i = 0; i < config.warmup_iterations; i++) {
        graphite_bundle* bundle = graphite_open(bundle_path);
        if (bundle) {
            graphite_close(bundle);
        }
    }
    
    // Actual benchmark
    graphite_start_perf_counters(&counters);
    uint64_t start_cycles = graphite_get_cpu_cycles();
    
    for (uint64_t i = 0; i < config.iterations; i++) {
        uint64_t start_time = graphite_get_precise_time_ns();
        
        graphite_bundle* bundle = graphite_open(bundle_path);
        if (!bundle) {
            times[i] = UINT64_MAX; // Mark as failed
            continue;
        }
        
        uint64_t end_time = graphite_get_precise_time_ns();
        times[i] = end_time - start_time;
        
        graphite_close(bundle);
    }
    
    uint64_t end_cycles = graphite_get_cpu_cycles();
    graphite_stop_perf_counters(&counters, result);
    
    // Calculate statistics
    qsort(times, config.iterations, sizeof(uint64_t), compare_uint64);
    
    result->min_time_ns = times[0];
    result->max_time_ns = times[config.iterations - 1];
    result->median_time_ns = times[config.iterations / 2];
    result->p95_time_ns = times[(config.iterations * 95) / 100];
    result->p99_time_ns = times[(config.iterations * 99) / 100];
    
    // Calculate mean and standard deviation
    uint64_t sum = 0;
    for (uint64_t i = 0; i < config.iterations; i++) {
        sum += times[i];
    }
    result->mean_time_ns = sum / config.iterations;
    
    uint64_t variance_sum = 0;
    for (uint64_t i = 0; i < config.iterations; i++) {
        uint64_t diff = times[i] > result->mean_time_ns ? 
                       times[i] - result->mean_time_ns : 
                       result->mean_time_ns - times[i];
        variance_sum += diff * diff;
    }
    result->std_dev_ns = sqrt(variance_sum / config.iterations);
    
    result->cpu_cycles = end_cycles - start_cycles;
    
    graphite_memory_stats final_memory = graphite_get_memory_stats();
    result->max_memory_bytes = final_memory.peak_rss - initial_memory.peak_rss;
    
    free(times);
}

// Asset streaming benchmark
void benchmark_asset_streaming(const char* bundle_path, 
                              uint32_t concurrent_requests,
                              graphite_benchmark_result* result) {
    graphite_bundle* bundle = graphite_open(bundle_path);
    if (!bundle) {
        result->min_time_ns = UINT64_MAX;
        return;
    }
    
    const graphite_graph* root = graphite_root(bundle);
    if (!root || root->header.node_cnt == 0) {
        graphite_close(bundle);
        result->min_time_ns = UINT64_MAX;
        return;
    }
    
    // Create streaming context
    graphite_streaming_context* streaming = graphite_streaming_create(bundle);
    
    // Generate random asset access pattern
    uint32_t* asset_sequence = malloc(sizeof(uint32_t) * concurrent_requests);
    for (uint32_t i = 0; i < concurrent_requests; i++) {
        asset_sequence[i] = rand() % root->header.node_cnt;
    }
    
    uint64_t start_time = graphite_get_precise_time_ns();
    uint64_t start_cycles = graphite_get_cpu_cycles();
    
    // Submit streaming requests
    for (uint32_t i = 0; i < concurrent_requests; i++) {
        graphite_stream_request request = {
            .asset_id = asset_sequence[i],
            .priority = GRAPHITE_PRIORITY_MEDIUM,
            .request_time = graphite_get_precise_time_ns()
        };
        graphite_submit_stream_request(streaming, &request);
    }
    
    // Wait for all requests to complete
    while (graphite_streaming_pending_count(streaming) > 0) {
        graphite_streaming_process(streaming);
        usleep(100); // 100 microseconds
    }
    
    uint64_t end_time = graphite_get_precise_time_ns();
    uint64_t end_cycles = graphite_get_cpu_cycles();
    
    result->mean_time_ns = end_time - start_time;
    result->cpu_cycles = end_cycles - start_cycles;
    
    // Get streaming statistics
    graphite_streaming_stats stats = graphite_streaming_get_stats(streaming);
    result->cache_hits = stats.cache_hits;
    result->cache_misses = stats.cache_misses;
    result->bytes_read = stats.bytes_streamed;
    result->io_operations = stats.requests_completed;
    
    free(asset_sequence);
    graphite_streaming_destroy(streaming);
    graphite_close(bundle);
}

// Hot reload benchmark
void benchmark_hot_reload(const char* bundle_path, graphite_benchmark_result* result) {
    graphite_bundle* original = graphite_open(bundle_path);
    if (!original) {
        result->min_time_ns = UINT64_MAX;
        return;
    }
    
    // Create temporary modified bundle
    char temp_path[256];
    snprintf(temp_path, sizeof(temp_path), "%s.tmp", bundle_path);
    
    // Create hot reload context
    graphite_hot_reload_config reload_config = {
        .bundle_path = temp_path,
        .atomic_reload = true,
        .preserve_handle_state = true
    };
    
    graphite_hot_reload_context* reload_ctx = graphite_hot_reload_init(&reload_config);
    
    uint64_t* reload_times = malloc(sizeof(uint64_t) * 100);
    
    for (int i = 0; i < 100; i++) {
        // Copy original to temp (simulating file change)
        graphite_copy_file(bundle_path, temp_path);
        
        uint64_t start_time = graphite_get_precise_time_ns();
        
        // Trigger hot reload
        graphite_trigger_reload(reload_ctx);
        
        // Wait for reload to complete
        while (graphite_reload_pending(reload_ctx)) {
            usleep(10);
        }
        
        uint64_t end_time = graphite_get_precise_time_ns();
        reload_times[i] = end_time - start_time;
    }
    
    // Calculate statistics
    qsort(reload_times, 100, sizeof(uint64_t), compare_uint64);
    result->min_time_ns = reload_times[0];
    result->max_time_ns = reload_times[99];
    result->median_time_ns = reload_times[50];
    result->p95_time_ns = reload_times[95];
    result->p99_time_ns = reload_times[99];
    
    free(reload_times);
    graphite_hot_reload_shutdown(reload_ctx);
    graphite_close(original);
    unlink(temp_path);
}
```

#### 20.2.2 Scalability Benchmarks
```c
// Large bundle performance scaling
void benchmark_bundle_scaling(graphite_benchmark_result* results, size_t result_count) {
    const char* test_bundles[] = {
        "test_1mb.graphite",    // 1 MB
        "test_10mb.graphite",   // 10 MB  
        "test_100mb.graphite",  // 100 MB
        "test_1gb.graphite",    // 1 GB
        "test_10gb.graphite"    // 10 GB
    };
    
    const size_t bundle_sizes[] = {
        1024 * 1024,           // 1 MB
        10 * 1024 * 1024,      // 10 MB
        100 * 1024 * 1024,     // 100 MB
        1024 * 1024 * 1024,    // 1 GB
        10ULL * 1024 * 1024 * 1024  // 10 GB
    };
    
    size_t num_bundles = sizeof(test_bundles) / sizeof(test_bundles[0]);
    if (result_count < num_bundles) {
        num_bundles = result_count;
    }
    
    for (size_t i = 0; i < num_bundles; i++) {
        printf("Benchmarking %s (%zu bytes)...\n", test_bundles[i], bundle_sizes[i]);
        
        // Open/close benchmark
        benchmark_bundle_open(test_bundles[i], &results[i]);
        
        // Calculate throughput (MB/s)
        double mb_size = bundle_sizes[i] / (1024.0 * 1024.0);
        double seconds = results[i].mean_time_ns / 1e9;
        double throughput = mb_size / seconds;
        
        printf("  Open time: %.2f ms (%.2f MB/s)\n", 
               results[i].mean_time_ns / 1e6, throughput);
        printf("  P95: %.2f ms, P99: %.2f ms\n",
               results[i].p95_time_ns / 1e6, results[i].p99_time_ns / 1e6);
        printf("  Memory: %.2f MB peak\n", 
               results[i].max_memory_bytes / (1024.0 * 1024.0));
        printf("  Cache misses: %llu, Page faults: %llu\n",
               results[i].cache_misses, results[i].page_faults);
    }
}

// Concurrent access benchmark
void benchmark_concurrent_access(const char* bundle_path, 
                                uint32_t thread_count,
                                graphite_benchmark_result* result) {
    typedef struct {
        const char* bundle_path;
        uint32_t thread_id;
        uint64_t iterations;
        uint64_t total_time_ns;
        uint64_t successful_operations;
    } thread_context;
    
    thread_context* contexts = malloc(sizeof(thread_context) * thread_count);
    pthread_t* threads = malloc(sizeof(pthread_t) * thread_count);
    
    // Worker thread function
    auto worker_thread = [](void* arg) -> void* {
        thread_context* ctx = (thread_context*)arg;
        
        uint64_t start_time = graphite_get_precise_time_ns();
        
        for (uint64_t i = 0; i < ctx->iterations; i++) {
            graphite_bundle* bundle = graphite_open(ctx->bundle_path);
            if (bundle) {
                // Perform some operations
                const graphite_graph* root = graphite_root(bundle);
                if (root && root->header.node_cnt > 0) {
                    // Access random asset
                    uint32_t asset_idx = rand() % root->header.node_cnt;
                    const graphite_graph* asset = graphite_get_child_graph(root, asset_idx);
                    if (asset) {
                        ctx->successful_operations++;
                    }
                }
                graphite_close(bundle);
            }
        }
        
        uint64_t end_time = graphite_get_precise_time_ns();
        ctx->total_time_ns = end_time - start_time;
        
        return NULL;
    };
    
    uint64_t iterations_per_thread = 1000;
    
    // Initialize contexts and start threads
    uint64_t overall_start = graphite_get_precise_time_ns();
    
    for (uint32_t i = 0; i < thread_count; i++) {
        contexts[i] = (thread_context){
            .bundle_path = bundle_path,
            .thread_id = i,
            .iterations = iterations_per_thread,
            .total_time_ns = 0,
            .successful_operations = 0
        };
        
        pthread_create(&threads[i], NULL, worker_thread, &contexts[i]);
    }
    
    // Wait for all threads to complete
    for (uint32_t i = 0; i < thread_count; i++) {
        pthread_join(threads[i], NULL);
    }
    
    uint64_t overall_end = graphite_get_precise_time_ns();
    
    // Aggregate results
    uint64_t total_operations = 0;
    uint64_t total_successful = 0;
    uint64_t min_thread_time = UINT64_MAX;
    uint64_t max_thread_time = 0;
    
    for (uint32_t i = 0; i < thread_count; i++) {
        total_operations += contexts[i].iterations;
        total_successful += contexts[i].successful_operations;
        
        if (contexts[i].total_time_ns < min_thread_time) {
            min_thread_time = contexts[i].total_time_ns;
        }
        if (contexts[i].total_time_ns > max_thread_time) {
            max_thread_time = contexts[i].total_time_ns;
        }
    }
    
    result->mean_time_ns = overall_end - overall_start;
    result->min_time_ns = min_thread_time;
    result->max_time_ns = max_thread_time;
    result->io_operations = total_successful;
    
    // Calculate throughput (operations per second)
    double seconds = result->mean_time_ns / 1e9;
    double ops_per_second = total_successful / seconds;
    
    printf("Concurrent access (%u threads):\n", thread_count);
    printf("  Total time: %.2f ms\n", result->mean_time_ns / 1e6);
    printf("  Operations: %llu successful / %llu total\n", total_successful, total_operations);
    printf("  Throughput: %.2f ops/sec\n", ops_per_second);
    printf("  Thread variance: %.2f ms (min: %.2f ms, max: %.2f ms)\n",
           (max_thread_time - min_thread_time) / 1e6,
           min_thread_time / 1e6, max_thread_time / 1e6);
    
    free(contexts);
    free(threads);
}
```

### 20.3 Performance Regression Testing

#### 20.3.1 Automated Performance CI
```bash
#!/bin/bash
# performance_ci.sh - Automated performance regression testing

set -e

BASELINE_RESULTS="baseline_performance.json"
CURRENT_RESULTS="current_performance.json"
REGRESSION_THRESHOLD=5.0  # 5% regression threshold

echo "=== GRAPHITE Performance Regression Testing ==="

# Build current version
echo "Building current version..."
make clean && make release

# Generate test bundles if they don't exist
echo "Generating test bundles..."
if [ ! -f "test_1mb.graphite" ]; then
    ./tools/generate_test_bundles.sh
fi

# Run performance benchmarks
echo "Running performance benchmarks..."
./graphite benchmark \
    --bundle test_1mb.graphite \
    --bundle test_10mb.graphite \
    --bundle test_100mb.graphite \
    --output ${CURRENT_RESULTS} \
    --iterations 1000 \
    --warmup 100 \
    --detailed

# Compare with baseline if it exists
if [ -f "${BASELINE_RESULTS}" ]; then
    echo "Comparing with baseline performance..."
    
    # Parse results and calculate regressions
    python3 << EOF
import json
import sys

def load_results(filename):
    with open(filename, 'r') as f:
        return json.load(f)

def calculate_regression(baseline, current, metric):
    if baseline[metric] == 0:
        return 0.0
    return ((current[metric] - baseline[metric]) / baseline[metric]) * 100.0

baseline = load_results("${BASELINE_RESULTS}")
current = load_results("${CURRENT_RESULTS}")

regressions = {}
critical_regressions = []

for test_name in baseline:
    if test_name not in current:
        continue
        
    baseline_test = baseline[test_name]
    current_test = current[test_name]
    
    # Check key metrics
    metrics = ['mean_time_ns', 'p95_time_ns', 'p99_time_ns', 'max_memory_bytes']
    
    for metric in metrics:
        if metric in baseline_test and metric in current_test:
            regression = calculate_regression(baseline_test, current_test, metric)
            regressions[f"{test_name}.{metric}"] = regression
            
            if regression > ${REGRESSION_THRESHOLD}:
                critical_regressions.append({
                    'test': test_name,
                    'metric': metric,
                    'regression': regression,
                    'baseline': baseline_test[metric],
                    'current': current_test[metric]
                })

# Print results
print(f"Performance Analysis Report")
print(f"=" * 50)

if critical_regressions:
    print(f"❌ CRITICAL REGRESSIONS DETECTED (>{REGRESSION_THRESHOLD}%):")
    for reg in critical_regressions:
        print(f"  {reg['test']}.{reg['metric']}: {reg['regression']:.1f}% regression")
        print(f"    Baseline: {reg['baseline']:,}")
        print(f"    Current:  {reg['current']:,}")
        print()
    sys.exit(1)
else:
    print("✅ No critical performance regressions detected")
    
    # Show summary of all changes
    print(f"\nPerformance Changes Summary:")
    for test_metric, regression in regressions.items():
        status = "📈" if regression > 0 else "📉" if regression < 0 else "➡️"
        print(f"  {status} {test_metric}: {regression:+.1f}%")

print(f"\nFull results saved to: ${CURRENT_RESULTS}")
EOF

    echo "Performance regression check completed"
else
    echo "No baseline found. Saving current results as new baseline."
    cp "${CURRENT_RESULTS}" "${BASELINE_RESULTS}"
fi

# Generate performance report
echo "Generating performance report..."
python3 tools/generate_perf_report.py "${CURRENT_RESULTS}" > performance_report.html

echo "Performance testing completed successfully"
```

#### 20.3.2 Performance Profiling Integration
```c
// Profiling hooks for development builds
#ifdef GRAPHITE_PROFILE_BUILD

#include <gperftools/profiler.h>

typedef struct {
    const char* profile_name;
    bool cpu_profiling;
    bool heap_profiling;
    bool enabled;
} graphite_profiling_config;

void graphite_start_profiling(const graphite_profiling_config* config) {
    if (!config->enabled) return;
    
    if (config->cpu_profiling) {
        char cpu_profile_path[256];
        snprintf(cpu_profile_path, sizeof(cpu_profile_path), 
                "%s_cpu.prof", config->profile_name);
        ProfilerStart(cpu_profile_path);
    }
    
    if (config->heap_profiling) {
        char heap_profile_path[256];
        snprintf(heap_profile_path, sizeof(heap_profile_path),
                "%s_heap", config->profile_name);
        HeapProfilerStart(heap_profile_path);
    }
}

void graphite_stop_profiling(const graphite_profiling_config* config) {
    if (!config->enabled) return;
    
    if (config->cpu_profiling) {
        ProfilerStop();
    }
    
    if (config->heap_profiling) {
        HeapProfilerStop();
    }
}

// Automatic profiling for slow operations
#define GRAPHITE_PROFILE_SLOW_OP(name, threshold_ms, block) do { \
    uint64_t start_time = graphite_get_precise_time_ns(); \
    graphite_profiling_config prof_config = { \
        .profile_name = name, \
        .cpu_profiling = true, \
        .heap_profiling = false, \
        .enabled = false \
    }; \
    \
    block; \
    \
    uint64_t end_time = graphite_get_precise_time_ns(); \
    uint64_t duration_ms = (end_time - start_time) / 1000000; \
    \
    if (duration_ms > threshold_ms) { \
        printf("Slow operation detected: %s took %llu ms\n", name, duration_ms); \
        /* Re-run with profiling enabled */ \
        prof_config.enabled = true; \
        graphite_start_profiling(&prof_config); \
        block; \
        graphite_stop_profiling(&prof_config); \
    } \
} while(0)

#else
#define GRAPHITE_PROFILE_SLOW_OP(name, threshold_ms, block) do { block; } while(0)
#endif // GRAPHITE_PROFILE_BUILD
```

---

## 21. Security Model & Threat Analysis

GRAPHITE implements comprehensive security measures to protect against malicious bundles, data corruption, and supply chain attacks.

### 21.1 Threat Model

#### 21.1.1 Attack Vectors
```c
// Comprehensive threat classification
typedef enum {
    GRAPHITE_THREAT_MALFORMED_BUNDLE,     // Crafted to crash parser
    GRAPHITE_THREAT_OVERSIZED_ALLOCATION, // Memory exhaustion attack
    GRAPHITE_THREAT_INFINITE_RECURSION,   // Stack overflow via deep graphs
    GRAPHITE_THREAT_INTEGER_OVERFLOW,     // Arithmetic overflow exploitation
    GRAPHITE_THREAT_PATH_TRAVERSAL,       // Directory traversal in paths
    GRAPHITE_THREAT_DECOMPRESSION_BOMB,   // ZIP bomb style attack
    GRAPHITE_THREAT_TIMING_ATTACK,        // Side-channel timing analysis
    GRAPHITE_THREAT_CACHE_COLLISION,      // Algorithmic complexity attack
    GRAPHITE_THREAT_SUPPLY_CHAIN,         // Compromised build pipeline
    GRAPHITE_THREAT_PRIVILEGE_ESCALATION, // Local privilege escalation
    GRAPHITE_THREAT_COUNT
} graphite_threat_type;

// Threat assessment matrix
typedef struct {
    graphite_threat_type type;
    const char* description;
    uint8_t likelihood;     // 1-5 scale
    uint8_t impact;         // 1-5 scale
    uint8_t risk_score;     // likelihood * impact
    const char* mitigation;
} graphite_threat_assessment;

static const graphite_threat_assessment threat_matrix[] = {
    {
        .type = GRAPHITE_THREAT_MALFORMED_BUNDLE,
        .description = "Maliciously crafted bundle designed to crash parser",
        .likelihood = 4,
        .impact = 3,
        .risk_score = 12,
        .mitigation = "Comprehensive input validation, fuzzing, bounds checking"
    },
    {
        .type = GRAPHITE_THREAT_OVERSIZED_ALLOCATION,
        .description = "Bundle with huge size fields causing memory exhaustion",
        .likelihood = 3,
        .impact = 4,
        .risk_score = 12,
        .mitigation = "Memory limits, allocation guards, progressive loading"
    },
    {
        .type = GRAPHITE_THREAT_INFINITE_RECURSION,
        .description = "Circular graph references causing stack overflow",
        .likelihood = 3,
        .impact = 3,
        .risk_score = 9,
        .mitigation = "Recursion depth limits, cycle detection, iterative traversal"
    },
    {
        .type = GRAPHITE_THREAT_DECOMPRESSION_BOMB,
        .description = "Highly compressed data expanding to huge size",
        .likelihood = 2,
        .impact = 4,
        .risk_score = 8,
        .mitigation = "Decompression ratio limits, incremental decompression"
    },
    {
        .type = GRAPHITE_THREAT_SUPPLY_CHAIN,
        .description = "Compromised build tools injecting malicious content",
        .likelihood = 2,
        .impact = 5,
        .risk_score = 10,
        .mitigation = "Code signing, reproducible builds, dependency verification"
    }
};
```

#### 21.1.2 Security Boundaries
```c
// Security context for sandboxed operations
typedef struct {
    // Memory limits
    size_t max_memory_allocation;    // Maximum single allocation
    size_t max_total_memory;         // Maximum total memory usage
    size_t max_decompressed_size;    // Maximum decompressed data size
    
    // Processing limits
    uint32_t max_recursion_depth;    // Maximum graph traversal depth
    uint32_t max_processing_time_ms; // Maximum processing time per operation
    uint32_t max_file_size;          // Maximum bundle file size
    
    // Feature restrictions
    bool allow_external_references;  // Allow external file references
    bool allow_code_execution;       // Allow embedded scripts/plugins
    bool allow_network_access;       // Allow network operations
    bool allow_file_system_access;   // Allow file system operations
    
    // Validation requirements
    bool require_signature;          // Require cryptographic signature
    bool require_integrity_check;    // Require full integrity verification
    bool strict_parsing;             // Reject any ambiguous content
} graphite_security_context;

// Default security contexts for different environments
static const graphite_security_context graphite_security_strict = {
    .max_memory_allocation = 64 * 1024 * 1024,  // 64 MB
    .max_total_memory = 512 * 1024 * 1024,      // 512 MB
    .max_decompressed_size = 1024 * 1024 * 1024, // 1 GB
    .max_recursion_depth = 64,
    .max_processing_time_ms = 30000,             // 30 seconds
    .max_file_size = 2ULL * 1024 * 1024 * 1024, // 2 GB
    .allow_external_references = false,
    .allow_code_execution = false,
    .allow_network_access = false,
    .allow_file_system_access = false,
    .require_signature = true,
    .require_integrity_check = true,
    .strict_parsing = true
};

static const graphite_security_context graphite_security_development = {
    .max_memory_allocation = 1024 * 1024 * 1024,  // 1 GB
    .max_total_memory = 8ULL * 1024 * 1024 * 1024, // 8 GB
    .max_decompressed_size = 16ULL * 1024 * 1024 * 1024, // 16 GB
    .max_recursion_depth = 256,
    .max_processing_time_ms = 300000,              // 5 minutes
    .max_file_size = 100ULL * 1024 * 1024 * 1024, // 100 GB
    .allow_external_references = true,
    .allow_code_execution = false,
    .allow_network_access = false,
    .allow_file_system_access = true,
    .require_signature = false,
    .require_integrity_check = true,
    .strict_parsing = false
};
```

### 21.2 Input Validation & Sanitization

#### 21.2.1 Defensive Bundle Parsing
```c
// Secure bundle parser with comprehensive validation
typedef struct {
    const uint8_t* data;
    size_t size;
    size_t position;
    const graphite_security_context* security;
    uint32_t recursion_depth;
    uint64_t start_time_ms;
    size_t allocated_memory;
} graphite_secure_parser;

// Safe integer operations with overflow checking
static inline bool safe_add_size_t(size_t a, size_t b, size_t* result) {
    if (a > SIZE_MAX - b) {
        return false; // Overflow would occur
    }
    *result = a + b;
    return true;
}

static inline bool safe_mul_size_t(size_t a, size_t b, size_t* result) {
    if (a != 0 && b > SIZE_MAX / a) {
        return false; // Overflow would occur
    }
    *result = a * b;
    return true;
}

// Secure memory allocation with limits
static void* secure_alloc(graphite_secure_parser* parser, size_t size) {
    // Check individual allocation limit
    if (size > parser->security->max_memory_allocation) {
        return NULL;
    }
    
    // Check total memory limit
    size_t new_total;
    if (!safe_add_size_t(parser->allocated_memory, size, &new_total)) {
        return NULL; // Overflow
    }
    
    if (new_total > parser->security->max_total_memory) {
        return NULL; // Would exceed limit
    }
    
    void* ptr = malloc(size);
    if (ptr) {
        parser->allocated_memory = new_total;
    }
    
    return ptr;
}

// Secure data reading with bounds checking
static graphite_result secure_read_bytes(graphite_secure_parser* parser,
                                        void* dest, size_t count) {
    // Check time limit
    uint64_t current_time = graphite_get_time_ms();
    if (current_time - parser->start_time_ms > parser->security->max_processing_time_ms) {
        return GRAPHITE_ERROR_TIMEOUT;
    }
    
    // Check bounds
    size_t new_position;
    if (!safe_add_size_t(parser->position, count, &new_position)) {
        return GRAPHITE_ERROR_OVERFLOW;
    }
    
    if (new_position > parser->size) {
        return GRAPHITE_ERROR_BOUNDS;
    }
    
    memcpy(dest, parser->data + parser->position, count);
    parser->position = new_position;
    
    return GRAPHITE_SUCCESS;
}

// Secure varint decoding with overflow protection
static graphite_result secure_read_varint(graphite_secure_parser* parser,
                                         uint64_t* value) {
    *value = 0;
    uint32_t shift = 0;
    
    for (int i = 0; i < 10; i++) { // Max 10 bytes for 64-bit varint
        uint8_t byte;
        graphite_result result = secure_read_bytes(parser, &byte, 1);
        if (result != GRAPHITE_SUCCESS) {
            return result;
        }
        
        // Check for overflow
        if (shift >= 64) {
            return GRAPHITE_ERROR_VARINT_OVERFLOW;
        }
        
        uint64_t value_part = (byte & 0x7F);
        if (shift < 64 && value_part > (UINT64_MAX >> shift)) {
            return GRAPHITE_ERROR_VARINT_OVERFLOW;
        }
        
        *value |= value_part << shift;
        
        if ((byte & 0x80) == 0) {
            return GRAPHITE_SUCCESS;
        }
        
        shift += 7;
    }
    
    return GRAPHITE_ERROR_VARINT_TOO_LONG;
}

// Secure string validation
static graphite_result secure_validate_string(const char* str, size_t length) {
    if (!str) return GRAPHITE_ERROR_NULL_POINTER;
    
    // Check for null termination
    bool found_null = false;
    for (size_t i = 0; i < length; i++) {
        if (str[i] == '\0') {
            found_null = true;
            break;
        }
        
        // Check for valid UTF-8 (basic validation)
        uint8_t byte = (uint8_t)str[i];
        if (byte >= 0x80) {
            // Multi-byte UTF-8 character - validate sequence
            // (Simplified validation - full UTF-8 validation is more complex)
            if ((byte & 0xE0) == 0xC0) {
                // 2-byte sequence
                if (i + 1 >= length || (str[i + 1] & 0xC0) != 0x80) {
                    return GRAPHITE_ERROR_INVALID_UTF8;
                }
                i++; // Skip next byte
            } else if ((byte & 0xF0) == 0xE0) {
                // 3-byte sequence
                if (i + 2 >= length || 
                    (str[i + 1] & 0xC0) != 0x80 || 
                    (str[i + 2] & 0xC0) != 0x80) {
                    return GRAPHITE_ERROR_INVALID_UTF8;
                }
                i += 2; // Skip next 2 bytes
            } else if ((byte & 0xF8) == 0xF0) {
                // 4-byte sequence
                if (i + 3 >= length ||
                    (str[i + 1] & 0xC0) != 0x80 ||
                    (str[i + 2] & 0xC0) != 0x80 ||
                    (str[i + 3] & 0xC0) != 0x80) {
                    return GRAPHITE_ERROR_INVALID_UTF8;
                }
                i += 3; // Skip next 3 bytes
            } else {
                return GRAPHITE_ERROR_INVALID_UTF8;
            }
        }
    }
    
    if (!found_null) {
        return GRAPHITE_ERROR_STRING_NOT_TERMINATED;
    }
    
    return GRAPHITE_SUCCESS;
}
```

#### 21.2.2 Decompression Safety
```c
// Secure decompression with bomb protection
typedef struct {
    size_t max_output_size;
    size_t max_ratio;
    uint32_t max_time_ms;
    void* (*progress_callback)(size_t decompressed, void* user_data);
    void* user_data;
} graphite_secure_decompress_config;

graphite_result graphite_secure_decompress(
    const void* compressed_data,
    size_t compressed_size,
    void** decompressed_data,
    size_t* decompressed_size,
    const graphite_secure_decompress_config* config
) {
    if (!compressed_data || !decompressed_data || !decompressed_size || !config) {
        return GRAPHITE_ERROR_NULL_POINTER;
    }
    
    // Initialize decompression stream
    z_stream stream = {0};
    if (inflateInit(&stream) != Z_OK) {
        return GRAPHITE_ERROR_DECOMPRESS_INIT;
    }
    
    stream.next_in = (Bytef*)compressed_data;
    stream.avail_in = compressed_size;
    
    // Allocate initial output buffer
    size_t output_capacity = compressed_size * 4; // Initial guess
    if (output_capacity > config->max_output_size) {
        output_capacity = config->max_output_size;
    }
    
    uint8_t* output_buffer = malloc(output_capacity);
    if (!output_buffer) {
        inflateEnd(&stream);
        return GRAPHITE_ERROR_ALLOCATION;
    }
    
    stream.next_out = output_buffer;
    stream.avail_out = output_capacity;
    
    size_t total_output = 0;
    uint64_t start_time = graphite_get_time_ms();
    
    int ret;
    do {
        ret = inflate(&stream, Z_NO_FLUSH);
        
        if (ret != Z_OK && ret != Z_STREAM_END) {
            free(output_buffer);
            inflateEnd(&stream);
            return GRAPHITE_ERROR_DECOMPRESS_FAILED;
        }
        
        // Check decompression ratio
        size_t current_output = output_capacity - stream.avail_out;
        if (compressed_size > 0) {
            size_t ratio = current_output / compressed_size;
            if (ratio > config->max_ratio) {
                free(output_buffer);
                inflateEnd(&stream);
                return GRAPHITE_ERROR_DECOMPRESS_RATIO;
            }
        }
        
        // Check time limit
        uint64_t current_time = graphite_get_time_ms();
        if (current_time - start_time > config->max_time_ms) {
            free(output_buffer);
            inflateEnd(&stream);
            return GRAPHITE_ERROR_TIMEOUT;
        }
        
        // Check size limit
        if (current_output > config->max_output_size) {
            free(output_buffer);
            inflateEnd(&stream);
            return GRAPHITE_ERROR_SIZE_LIMIT;
        }
        
        // Progress callback
        if (config->progress_callback) {
            config->progress_callback(current_output, config->user_data);
        }
        
        // Resize buffer if needed
        if (stream.avail_out == 0 && ret != Z_STREAM_END) {
            size_t new_capacity = output_capacity * 2;
            if (new_capacity > config->max_output_size) {
                new_capacity = config->max_output_size;
            }
            
            if (new_capacity <= output_capacity) {
                // Can't grow further
                free(output_buffer);
                inflateEnd(&stream);
                return GRAPHITE_ERROR_SIZE_LIMIT;
            }
            
            uint8_t* new_buffer = realloc(output_buffer, new_capacity);
            if (!new_buffer) {
                free(output_buffer);
                inflateEnd(&stream);
                return GRAPHITE_ERROR_ALLOCATION;
            }
            
            output_buffer = new_buffer;
            stream.next_out = output_buffer + output_capacity;
            stream.avail_out = new_capacity - output_capacity;
            output_capacity = new_capacity;
        }
        
    } while (ret != Z_STREAM_END);
    
    inflateEnd(&stream);
    
    *decompressed_data = output_buffer;
    *decompressed_size = output_capacity - stream.avail_out;
    
    return GRAPHITE_SUCCESS;
}
```

### 21.3 Cryptographic Integrity

#### 21.3.1 Digital Signatures
```c
// Digital signature support using Ed25519
#include <sodium.h>

typedef struct {
    uint8_t public_key[crypto_sign_PUBLICKEYBYTES];
    uint8_t signature[crypto_sign_BYTES];
    uint64_t timestamp;
    char signer_id[64];
} graphite_signature;

typedef struct {
    uint32_t signature_count;
    graphite_signature signatures[];
} graphite_signature_block;

// Sign a bundle
graphite_result graphite_sign_bundle(
    const char* bundle_path,
    const uint8_t* private_key,
    const char* signer_id
) {
    if (!bundle_path || !private_key || !signer_id) {
        return GRAPHITE_ERROR_NULL_POINTER;
    }
    
    // Read bundle data
    FILE* file = fopen(bundle_path, "rb");
    if (!file) {
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    fseek(file, 0, SEEK_END);
    long file_size = ftell(file);
    fseek(file, 0, SEEK_SET);
    
    uint8_t* bundle_data = malloc(file_size);
    if (!bundle_data) {
        fclose(file);
        return GRAPHITE_ERROR_ALLOCATION;
    }
    
    if (fread(bundle_data, 1, file_size, file) != file_size) {
        free(bundle_data);
        fclose(file);
        return GRAPHITE_ERROR_IO;
    }
    fclose(file);
    
    // Calculate BLAKE3 hash of bundle
    uint8_t bundle_hash[32];
    blake3_hasher hasher;
    blake3_hasher_init(&hasher);
    blake3_hasher_update(&hasher, bundle_data, file_size);
    blake3_hasher_finalize(&hasher, bundle_hash, 32);
    
    // Create signature
    graphite_signature sig = {0};
    
    // Get public key from private key
    uint8_t full_keypair[crypto_sign_SECRETKEYBYTES];
    memcpy(full_keypair, private_key, crypto_sign_SEEDBYTES);
    crypto_sign_seed_keypair(sig.public_key, full_keypair, private_key);
    
    // Sign the hash
    unsigned long long sig_len;
    crypto_sign_detached(sig.signature, &sig_len, bundle_hash, 32, full_keypair);
    
    sig.timestamp = time(NULL);
    strncpy(sig.signer_id, signer_id, sizeof(sig.signer_id) - 1);
    
    // Append signature to bundle
    file = fopen(bundle_path, "ab");
    if (!file) {
        free(bundle_data);
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    // Write signature magic and data
    uint32_t sig_magic = 0x47534947; // "GSIG"
    fwrite(&sig_magic, sizeof(sig_magic), 1, file);
    fwrite(&sig, sizeof(sig), 1, file);
    
    fclose(file);
    free(bundle_data);
    
    // Clear sensitive data
    sodium_memzero(full_keypair, sizeof(full_keypair));
    
    return GRAPHITE_SUCCESS;
}

// Verify bundle signatures
graphite_result graphite_verify_bundle_signatures(
    const char* bundle_path,
    const uint8_t* trusted_public_keys,
    size_t key_count,
    bool* verified
) {
    *verified = false;
    
    FILE* file = fopen(bundle_path, "rb");
    if (!file) {
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    // Find signature block
    fseek(file, -4, SEEK_END);
    uint32_t magic;
    if (fread(&magic, sizeof(magic), 1, file) != 1 || magic != 0x47534947) {
        fclose(file);
        return GRAPHITE_SUCCESS; // No signature, not an error
    }
    
    // Read signature
    fseek(file, -(long)(sizeof(graphite_signature) + 4), SEEK_END);
    graphite_signature sig;
    if (fread(&sig, sizeof(sig), 1, file) != 1) {
        fclose(file);
        return GRAPHITE_ERROR_IO;
    }
    
    // Get bundle size without signature
    long bundle_size = ftell(file) - sizeof(graphite_signature) - 4;
    
    // Read bundle data
    fseek(file, 0, SEEK_SET);
    uint8_t* bundle_data = malloc(bundle_size);
    if (!bundle_data) {
        fclose(file);
        return GRAPHITE_ERROR_ALLOCATION;
    }
    
    if (fread(bundle_data, 1, bundle_size, file) != bundle_size) {
        free(bundle_data);
        fclose(file);
        return GRAPHITE_ERROR_IO;
    }
    fclose(file);
    
    // Calculate bundle hash
    uint8_t bundle_hash[32];
    blake3_hasher hasher;
    blake3_hasher_init(&hasher);
    blake3_hasher_update(&hasher, bundle_data, bundle_size);
    blake3_hasher_finalize(&hasher, bundle_hash, 32);
    
    free(bundle_data);
    
    // Check if signature is from a trusted key
    bool key_trusted = false;
    for (size_t i = 0; i < key_count; i++) {
        if (memcmp(sig.public_key, 
                   trusted_public_keys + i * crypto_sign_PUBLICKEYBYTES,
                   crypto_sign_PUBLICKEYBYTES) == 0) {
            key_trusted = true;
            break;
        }
    }
    
    if (!key_trusted) {
        return GRAPHITE_SUCCESS; // Signature present but not from trusted key
    }
    
    // Verify signature
    if (crypto_sign_verify_detached(sig.signature, bundle_hash, 32, sig.public_key) == 0) {
        *verified = true;
    }
    
    return GRAPHITE_SUCCESS;
}
```

#### 21.3.2 Authenticated Encryption
```c
// Encrypted bundle support using ChaCha20-Poly1305
typedef struct {
    uint8_t nonce[crypto_aead_chacha20poly1305_NPUBBYTES];
    uint8_t key_id[16];
    uint32_t encrypted_size;
    // Followed by encrypted data + authentication tag
} graphite_encryption_header;

// Encrypt a bundle
graphite_result graphite_encrypt_bundle(
    const char* input_path,
    const char* output_path,
    const uint8_t* encryption_key,
    const uint8_t* key_id
) {
    FILE* input = fopen(input_path, "rb");
    if (!input) {
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    FILE* output = fopen(output_path, "wb");
    if (!output) {
        fclose(input);
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    // Get input size
    fseek(input, 0, SEEK_END);
    long input_size = ftell(input);
    fseek(input, 0, SEEK_SET);
    
    // Read input data
    uint8_t* plaintext = malloc(input_size);
    if (!plaintext) {
        fclose(input);
        fclose(output);
        return GRAPHITE_ERROR_ALLOCATION;
    }
    
    if (fread(plaintext, 1, input_size, input) != input_size) {
        free(plaintext);
        fclose(input);
        fclose(output);
        return GRAPHITE_ERROR_IO;
    }
    fclose(input);
    
    // Prepare encryption header
    graphite_encryption_header header = {0};
    randombytes_buf(header.nonce, sizeof(header.nonce));
    memcpy(header.key_id, key_id, 16);
    header.encrypted_size = input_size + crypto_aead_chacha20poly1305_ABYTES;
    
    // Write header
    fwrite(&header, sizeof(header), 1, output);
    
    // Allocate ciphertext buffer
    uint8_t* ciphertext = malloc(header.encrypted_size);
    if (!ciphertext) {
        free(plaintext);
        fclose(output);
        return GRAPHITE_ERROR_ALLOCATION;
    }
    
    // Encrypt data
    unsigned long long ciphertext_len;
    if (crypto_aead_chacha20poly1305_encrypt(
            ciphertext, &ciphertext_len,
            plaintext, input_size,
            (uint8_t*)&header, sizeof(header) - 4, // Use header as additional data
            NULL, header.nonce, encryption_key) != 0) {
        free(plaintext);
        free(ciphertext);
        fclose(output);
        return GRAPHITE_ERROR_ENCRYPTION;
    }
    
    // Write encrypted data
    fwrite(ciphertext, 1, ciphertext_len, output);
    
    fclose(output);
    free(plaintext);
    free(ciphertext);
    
    return GRAPHITE_SUCCESS;
}

// Decrypt a bundle
graphite_result graphite_decrypt_bundle(
    const char* input_path,
    const char* output_path,
    const uint8_t* encryption_key
) {
    FILE* input = fopen(input_path, "rb");
    if (!input) {
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    // Read encryption header
    graphite_encryption_header header;
    if (fread(&header, sizeof(header), 1, input) != 1) {
        fclose(input);
        return GRAPHITE_ERROR_IO;
    }
    
    // Read encrypted data
    uint8_t* ciphertext = malloc(header.encrypted_size);
    if (!ciphertext) {
        fclose(input);
        return GRAPHITE_ERROR_ALLOCATION;
    }
    
    if (fread(ciphertext, 1, header.encrypted_size, input) != header.encrypted_size) {
        free(ciphertext);
        fclose(input);
        return GRAPHITE_ERROR_IO;
    }
    fclose(input);
    
    // Allocate plaintext buffer
    size_t plaintext_size = header.encrypted_size - crypto_aead_chacha20poly1305_ABYTES;
    uint8_t* plaintext = malloc(plaintext_size);
    if (!plaintext) {
        free(ciphertext);
        return GRAPHITE_ERROR_ALLOCATION;
    }
    
    // Decrypt data
    unsigned long long plaintext_len;
    if (crypto_aead_chacha20poly1305_decrypt(
            plaintext, &plaintext_len,
            NULL,
            ciphertext, header.encrypted_size,
            (uint8_t*)&header, sizeof(header) - 4,
            header.nonce, encryption_key) != 0) {
        free(plaintext);
        free(ciphertext);
        return GRAPHITE_ERROR_DECRYPTION;
    }
    
    // Write decrypted data
    FILE* output = fopen(output_path, "wb");
    if (!output) {
        free(plaintext);
        free(ciphertext);
        return GRAPHITE_ERROR_FILE_OPEN;
    }
    
    fwrite(plaintext, 1, plaintext_len, output);
    fclose(output);
    
    free(plaintext);
    free(ciphertext);
    
    return GRAPHITE_SUCCESS;
}
```

---

## 22. Implementation Guidelines

This section provides comprehensive guidance for implementing GRAPHITE in production environments, covering architectural decisions, coding standards, and best practices.

### 22.1 Architecture Guidelines

#### 22.1.1 Modular Design Principles
```c
// Core module structure for GRAPHITE implementation
typedef struct graphite_module {
    const char* name;
    const char* version;
    const graphite_module_interface* interface;
    void* private_data;
    graphite_module_state state;
} graphite_module;

// Well-defined module interfaces
typedef struct {
    graphite_result (*init)(void** context, const graphite_config* config);
    graphite_result (*process)(void* context, const graphite_input* input, graphite_output* output);
    void (*cleanup)(void* context);
    graphite_result (*get_info)(void* context, graphite_module_info* info);
} graphite_module_interface;

// Example: Compression module implementation
static const graphite_module_interface compression_interface = {
    .init = compression_init,
    .process = compression_process,
    .cleanup = compression_cleanup,
    .get_info = compression_get_info
};

// Module registry for dependency injection
typedef struct {
    hashtable* modules;           // name -> module mapping
    dynamic_array* load_order;   // Module initialization order
    graphite_config* global_config;
} graphite_module_registry;

// Register modules with clear dependencies
graphite_result graphite_register_module(
    graphite_module_registry* registry,
    const char* name,
    const graphite_module_interface* interface,
    const char** dependencies,
    size_t dependency_count
) {
    // Validate dependencies exist
    for (size_t i = 0; i < dependency_count; i++) {
        if (!hashtable_contains(registry->modules, dependencies[i])) {
            return GRAPHITE_ERROR_MISSING_DEPENDENCY;
        }
    }
    
    // Create module instance
    graphite_module* module = malloc(sizeof(graphite_module));
    module->name = strdup(name);
    module->interface = interface;
    module->state = GRAPHITE_MODULE_REGISTERED;
    
    // Initialize module
    graphite_result result = interface->init(&module->private_data, registry->global_config);
    if (result != GRAPHITE_SUCCESS) {
        free(module);
        return result;
    }
    
    module->state = GRAPHITE_MODULE_INITIALIZED;
    hashtable_insert(registry->modules, name, module);
    
    return GRAPHITE_SUCCESS;
}
```

#### 22.1.2 Error Handling Strategy
```c
// Comprehensive error handling system
typedef enum {
    GRAPHITE_ERROR_CATEGORY_SYSTEM,     // OS/system level errors
    GRAPHITE_ERROR_CATEGORY_FORMAT,     // File format errors
    GRAPHITE_ERROR_CATEGORY_MEMORY,     // Memory allocation errors
    GRAPHITE_ERROR_CATEGORY_IO,         // I/O operation errors
    GRAPHITE_ERROR_CATEGORY_SECURITY,   // Security/validation errors
    GRAPHITE_ERROR_CATEGORY_CONFIG,     // Configuration errors
    GRAPHITE_ERROR_CATEGORY_NETWORK,    // Network operation errors
    GRAPHITE_ERROR_CATEGORY_USER        // User input errors
} graphite_error_category;

typedef struct {
    graphite_result code;
    graphite_error_category category;
    const char* message;
    const char* file;
    int line;
    const char* function;
    uint64_t timestamp;
    void* context_data;
} graphite_error_info;

// Error context for detailed debugging
typedef struct {
    graphite_error_info* error_stack;
    size_t stack_size;
    size_t stack_capacity;
    graphite_log_callback log_callback;
    void* log_user_data;
} graphite_error_context;

// Error reporting macros
#define GRAPHITE_SET_ERROR(ctx, code, cat, msg) \
    graphite_set_error_impl(ctx, code, cat, msg, __FILE__, __LINE__, __func__)

#define GRAPHITE_PROPAGATE_ERROR(ctx, result) \
    do { \
        if (result != GRAPHITE_SUCCESS) { \
            graphite_add_error_context(ctx, __FILE__, __LINE__, __func__); \
            return result; \
        } \
    } while(0)

// Error handling implementation
void graphite_set_error_impl(
    graphite_error_context* ctx,
    graphite_result code,
    graphite_error_category category,
    const char* message,
    const char* file,
    int line,
    const char* function
) {
    if (!ctx) return;
    
    // Ensure stack capacity
    if (ctx->stack_size >= ctx->stack_capacity) {
        size_t new_capacity = ctx->stack_capacity ? ctx->stack_capacity * 2 : 16;
        graphite_error_info* new_stack = realloc(ctx->error_stack, 
                                                 new_capacity * sizeof(graphite_error_info));
        if (!new_stack) return; // Can't report out of memory for error reporting
        
        ctx->error_stack = new_stack;
        ctx->stack_capacity = new_capacity;
    }
    
    // Add error to stack
    graphite_error_info* error = &ctx->error_stack[ctx->stack_size++];
    error->code = code;
    error->category = category;
    error->message = message;
    error->file = file;
    error->line = line;
    error->function = function;
    error->timestamp = graphite_get_time_ms();
    
    // Log error if callback provided
    if (ctx->log_callback) {
        ctx->log_callback(GRAPHITE_LOG_ERROR, message, ctx->log_user_data);
    }
}

// Resource cleanup with error handling
typedef struct {
    void (*cleanup_func)(void* resource);
    void* resource;
    const char* description;
} graphite_cleanup_item;

typedef struct {
    graphite_cleanup_item* items;
    size_t count;
    size_t capacity;
} graphite_cleanup_stack;

#define GRAPHITE_DEFER_CLEANUP(stack, func, resource, desc) \
    graphite_defer_cleanup(stack, (void(*)(void*))func, resource, desc)

void graphite_defer_cleanup(
    graphite_cleanup_stack* stack,
    void (*cleanup_func)(void*),
    void* resource,
    const char* description
) {
    if (stack->count >= stack->capacity) {
        size_t new_capacity = stack->capacity ? stack->capacity * 2 : 16;
        graphite_cleanup_item* new_items = realloc(stack->items,
                                                  new_capacity * sizeof(graphite_cleanup_item));
        if (!new_items) return; // Silent failure in cleanup registration
        
        stack->items = new_items;
        stack->capacity = new_capacity;
    }
    
    graphite_cleanup_item* item = &stack->items[stack->count++];
    item->cleanup_func = cleanup_func;
    item->resource = resource;
    item->description = description;
}

void graphite_execute_cleanup(graphite_cleanup_stack* stack) {
    // Execute in reverse order (LIFO)
    for (int i = (int)stack->count - 1; i >= 0; i--) {
        graphite_cleanup_item* item = &stack->items[i];
        if (item->cleanup_func && item->resource) {
            item->cleanup_func(item->resource);
        }
    }
    stack->count = 0;
}
```

### 22.2 Performance Implementation Guidelines

#### 22.2.1 Memory Management Best Practices
```c
// Pool-based allocation for frequent small objects
typedef struct {
    void* memory_block;
    size_t block_size;
    size_t object_size;
    size_t object_count;
    uint32_t* free_bitmap;      // Bitmap of free objects
    size_t next_free_hint;      // Hint for next free object
    atomic_size_t allocated_count;
} graphite_object_pool;

graphite_object_pool* graphite_pool_create(size_t object_size, size_t initial_count) {
    // Align object size to cache line boundary
    size_t aligned_size = (object_size + 63) & ~63;
    
    graphite_object_pool* pool = malloc(sizeof(graphite_object_pool));
    if (!pool) return NULL;
    
    pool->object_size = aligned_size;
    pool->object_count = initial_count;
    pool->block_size = aligned_size * initial_count;
    
    // Allocate aligned memory block
    pool->memory_block = graphite_aligned_alloc(pool->block_size, 64);
    if (!pool->memory_block) {
        free(pool);
        return NULL;
    }
    
    // Initialize free bitmap (all objects initially free)
    size_t bitmap_size = (initial_count + 31) / 32;
    pool->free_bitmap = calloc(bitmap_size, sizeof(uint32_t));
    if (!pool->free_bitmap) {
        graphite_aligned_free(pool->memory_block);
        free(pool);
        return NULL;
    }
    
    // Mark all objects as free
    for (size_t i = 0; i < initial_count; i++) {
        size_t word_idx = i / 32;
        size_t bit_idx = i % 32;
        pool->free_bitmap[word_idx] |= (1U << bit_idx);
    }
    
    pool->next_free_hint = 0;
    atomic_init(&pool->allocated_count, 0);
    
    return pool;
}

void* graphite_pool_alloc(graphite_object_pool* pool) {
    // Fast path: check hint first
    size_t start_idx = pool->next_free_hint;
    for (size_t i = 0; i < pool->object_count; i++) {
        size_t idx = (start_idx + i) % pool->object_count;
        size_t word_idx = idx / 32;
        size_t bit_idx = idx % 32;
        
        uint32_t mask = 1U << bit_idx;
        if (pool->free_bitmap[word_idx] & mask) {
            // Found free object, mark as allocated
            pool->free_bitmap[word_idx] &= ~mask;
            pool->next_free_hint = (idx + 1) % pool->object_count;
            
            atomic_fetch_add(&pool->allocated_count, 1);
            
            // Return aligned pointer
            return (uint8_t*)pool->memory_block + (idx * pool->object_size);
        }
    }
    
    return NULL; // Pool exhausted
}

void graphite_pool_free(graphite_object_pool* pool, void* ptr) {
    if (!ptr || ptr < pool->memory_block) return;
    
    // Calculate object index
    ptrdiff_t offset = (uint8_t*)ptr - (uint8_t*)pool->memory_block;
    if (offset % pool->object_size != 0) return; // Invalid pointer
    
    size_t idx = offset / pool->object_size;
    if (idx >= pool->object_count) return; // Out of bounds
    
    // Mark as free
    size_t word_idx = idx / 32;
    size_t bit_idx = idx % 32;
    uint32_t mask = 1U << bit_idx;
    
    if (!(pool->free_bitmap[word_idx] & mask)) {
        pool->free_bitmap[word_idx] |= mask;
        atomic_fetch_sub(&pool->allocated_count, 1);
        
        // Update hint for faster allocation
        if (idx < pool->next_free_hint) {
            pool->next_free_hint = idx;
        }
    }
}

// RAII-style resource management
#define GRAPHITE_SCOPED_POOL_ALLOC(pool, type) \
    __attribute__((cleanup(graphite_pool_free_cleanup))) \
    type* ptr = (type*)graphite_pool_alloc(pool); \
    if (!ptr) return GRAPHITE_ERROR_ALLOCATION

static void graphite_pool_free_cleanup(void* ptr_to_ptr) {
    void** ptr = (void**)ptr_to_ptr;
    if (*ptr) {
        // Note: This requires pool context - in practice, use a wrapper
        // that stores pool reference with the allocation
        graphite_pool_free(current_pool, *ptr);
    }
}
```

#### 22.2.2 Cache-Friendly Data Structures
```c
// Structure of Arrays (SoA) for better cache utilization
typedef struct {
    // Instead of Array of Structures (AoS):
    // struct { uint32_t id; float position[3]; uint32_t flags; } nodes[];
    
    // Use Structure of Arrays (SoA):
    uint32_t* ids;              // Cache line aligned
    float* positions;           // 3 floats per node, packed
    uint32_t* flags;            // Cache line aligned
    
    size_t count;
    size_t capacity;
    
    // Memory layout optimized for common access patterns
    void* memory_block;         // Single allocation for all arrays
} graphite_node_array;

graphite_node_array* graphite_nodes_create(size_t initial_capacity) {
    graphite_node_array* nodes = malloc(sizeof(graphite_node_array));
    if (!nodes) return NULL;
    
    // Calculate memory layout with cache line alignment
    size_t ids_size = initial_capacity * sizeof(uint32_t);
    size_t positions_size = initial_capacity * 3 * sizeof(float);
    size_t flags_size = initial_capacity * sizeof(uint32_t);
    
    // Align each array to cache line boundary
    ids_size = (ids_size + 63) & ~63;
    positions_size = (positions_size + 63) & ~63;
    flags_size = (flags_size + 63) & ~63;
    
    size_t total_size = ids_size + positions_size + flags_size;
    
    nodes->memory_block = graphite_aligned_alloc(total_size, 64);
    if (!nodes->memory_block) {
        free(nodes);
        return NULL;
    }
    
    // Set up array pointers
    uint8_t* base = (uint8_t*)nodes->memory_block;
    nodes->ids = (uint32_t*)base;
    nodes->positions = (float*)(base + ids_size);
    nodes->flags = (uint32_t*)(base + ids_size + positions_size);
    
    nodes->count = 0;
    nodes->capacity = initial_capacity;
    
    return nodes;
}

// Cache-friendly iteration patterns
void graphite_nodes_process_positions(graphite_node_array* nodes, 
                                     graphite_matrix4 transform) {
    // Process in chunks that fit in cache
    const size_t chunk_size = 64; // Tune based on cache size
    
    for (size_t start = 0; start < nodes->count; start += chunk_size) {
        size_t end = start + chunk_size;
        if (end > nodes->count) end = nodes->count;
        
        // Prefetch next chunk
        if (end < nodes->count) {
            __builtin_prefetch(&nodes->positions[end * 3], 0, 3);
        }
        
        // Process current chunk
        for (size_t i = start; i < end; i++) {
            float* pos = &nodes->positions[i * 3];
            graphite_transform_point(transform, pos);
        }
    }
}

// Branch-friendly data organization
typedef struct {
    // Group related data to minimize branches
    struct {
        uint32_t visible_count;
        uint32_t* visible_indices;
    } visible;
    
    struct {
        uint32_t culled_count;
        uint32_t* culled_indices;
    } culled;
    
    struct {
        uint32_t dirty_count;
        uint32_t* dirty_indices;
    } dirty;
} graphite_node_buckets;

// Bucket sorting for cache-friendly processing
void graphite_bucket_nodes_by_state(graphite_node_array* nodes, 
                                   graphite_node_buckets* buckets) {
    buckets->visible.visible_count = 0;
    buckets->culled.culled_count = 0;
    buckets->dirty.dirty_count = 0;
    
    // Single pass bucketing
    for (size_t i = 0; i < nodes->count; i++) {
        uint32_t flags = nodes->flags[i];
        
        if (flags & GRAPHITE_NODE_VISIBLE) {
            buckets->visible.visible_indices[buckets->visible.visible_count++] = i;
        } else if (flags & GRAPHITE_NODE_CULLED) {
            buckets->culled.culled_indices[buckets->culled.culled_count++] = i;
        }
        
        if (flags & GRAPHITE_NODE_DIRTY) {
            buckets->dirty.dirty_indices[buckets->dirty.dirty_count++] = i;
        }
    }
}
```

### 22.3 Threading and Concurrency Guidelines

#### 22.3.1 Lock-Free Data Structures
```c
// Lock-free queue for work distribution
typedef struct {
    atomic_ptr head;
    atomic_ptr tail;
    atomic_size_t size;
} graphite_lockfree_queue;

typedef struct graphite_queue_node {
    atomic_ptr next;
    void* data;
} graphite_queue_node;

graphite_lockfree_queue* graphite_queue_create(void) {
    graphite_lockfree_queue* queue = malloc(sizeof(graphite_lockfree_queue));
    if (!queue) return NULL;
    
    // Create dummy node
    graphite_queue_node* dummy = malloc(sizeof(graphite_queue_node));
    if (!dummy) {
        free(queue);
        return NULL;
    }
    
    atomic_init(&dummy->next, NULL);
    dummy->data = NULL;
    
    atomic_init(&queue->head, dummy);
    atomic_init(&queue->tail, dummy);
    atomic_init(&queue->size, 0);
    
    return queue;
}

bool graphite_queue_enqueue(graphite_lockfree_queue* queue, void* data) {
    graphite_queue_node* node = malloc(sizeof(graphite_queue_node));
    if (!node) return false;
    
    atomic_init(&node->next, NULL);
    node->data = data;
    
    graphite_queue_node* tail;
    
    while (true) {
        tail = atomic_load(&queue->tail);
        graphite_queue_node* next = atomic_load(&tail->next);
        
        if (tail == atomic_load(&queue->tail)) { // Tail hasn't changed
            if (next == NULL) {
                // Try to link node at the end of the list
                if (atomic_compare_exchange_weak(&tail->next, &next, node)) {
                    break; // Successfully enqueued
                }
            } else {
                // Try to swing tail to the next node
                atomic_compare_exchange_weak(&queue->tail, &tail, next);
            }
        }
    }
    
    // Try to swing tail to the inserted node
    atomic_compare_exchange_weak(&queue->tail, &tail, node);
    atomic_fetch_add(&queue->size, 1);
    
    return true;
}

bool graphite_queue_dequeue(graphite_lockfree_queue* queue, void** data) {
    graphite_queue_node* head;
    graphite_queue_node* tail;
    graphite_queue_node* next;
    
    while (true) {
        head = atomic_load(&queue->head);
        tail = atomic_load(&queue->tail);
        next = atomic_load(&head->next);
        
        if (head == atomic_load(&queue->head)) { // Head hasn't changed
            if (head == tail) {
                if (next == NULL) {
                    return false; // Queue is empty
                }
                // Try to swing tail to the next node
                atomic_compare_exchange_weak(&queue->tail, &tail, next);
            } else {
                if (next == NULL) {
                    continue; // Another thread is modifying the queue
                }
                
                // Read data before CAS
                *data = next->data;
                
                // Try to swing head to the next node
                if (atomic_compare_exchange_weak(&queue->head, &head, next)) {
                    atomic_fetch_sub(&queue->size, 1);
                    free(head);
                    return true;
                }
            }
        }
    }
}

// Work-stealing thread pool
typedef struct {
    graphite_lockfree_queue* global_queue;
    graphite_lockfree_queue** worker_queues;
    pthread_t* threads;
    atomic_bool shutdown;
    uint32_t thread_count;
    uint32_t current_thread_id; // Thread-local
} graphite_thread_pool;

typedef struct {
    void (*function)(void* arg);
    void* argument;
    atomic_int* completion_counter; // For synchronization
} graphite_work_item;

static __thread uint32_t tls_thread_id = UINT32_MAX;

void* graphite_worker_thread(void* arg) {
    graphite_thread_pool* pool = (graphite_thread_pool*)arg;
    uint32_t thread_id = atomic_fetch_add(&pool->current_thread_id, 1);
    tls_thread_id = thread_id;
    
    graphite_lockfree_queue* my_queue = pool->worker_queues[thread_id];
    
    while (!atomic_load(&pool->shutdown)) {
        graphite_work_item* work = NULL;
        
        // Try to get work from own queue first
        if (graphite_queue_dequeue(my_queue, (void**)&work)) {
            work->function(work->argument);
            if (work->completion_counter) {
                atomic_fetch_sub(work->completion_counter, 1);
            }
            free(work);
            continue;
        }
        
        // Try to steal work from other threads
        for (uint32_t i = 0; i < pool->thread_count; i++) {
            if (i == thread_id) continue;
            
            if (graphite_queue_dequeue(pool->worker_queues[i], (void**)&work)) {
                work->function(work->argument);
                if (work->completion_counter) {
                    atomic_fetch_sub(work->completion_counter, 1);
                }
                free(work);
                goto continue_work;
            }
        }
        
        // Try global queue
        if (graphite_queue_dequeue(pool->global_queue, (void**)&work)) {
            work->function(work->argument);
            if (work->completion_counter) {
                atomic_fetch_sub(work->completion_counter, 1);
            }
            free(work);
            continue;
        }
        
        continue_work:
        // No work available, yield CPU
        sched_yield();
    }
    
    return NULL;
}

// Submit work to thread pool
void graphite_pool_submit(graphite_thread_pool* pool, 
                         void (*function)(void*),
                         void* argument,
                         atomic_int* completion_counter) {
    graphite_work_item* work = malloc(sizeof(graphite_work_item));
    work->function = function;
    work->argument = argument;
    work->completion_counter = completion_counter;
    
    if (completion_counter) {
        atomic_fetch_add(completion_counter, 1);
    }
    
    // If called from worker thread, use local queue
    if (tls_thread_id < pool->thread_count) {
        graphite_queue_enqueue(pool->worker_queues[tls_thread_id], work);
    } else {
        // Use global queue for external submissions
        graphite_queue_enqueue(pool->global_queue, work);
    }
}
```

---

## 23. Quality Assurance & Testing Framework

Comprehensive testing strategy covering unit tests, integration tests, performance tests, and security validation.

### 23.1 Testing Pyramid

#### 23.1.1 Unit Testing Framework
```c
// Lightweight unit testing framework for GRAPHITE
typedef enum {
    GRAPHITE_TEST_PASS,
    GRAPHITE_TEST_FAIL,
    GRAPHITE_TEST_SKIP
} graphite_test_result;

typedef struct {
    const char* name;
    const char* suite;
    graphite_test_result (*test_func)(void);
    uint64_t execution_time_ns;
    const char* failure_message;
} graphite_test_case;

typedef struct {
    graphite_test_case* tests;
    size_t test_count;
    size_t capacity;
    
    // Statistics
    size_t passed;
    size_t failed;
    size_t skipped;
    uint64_t total_time_ns;
} graphite_test_suite;

// Test assertion macros
#define GRAPHITE_ASSERT_EQ(expected, actual) \
    do { \
        if ((expected) != (actual)) { \
            printf("ASSERTION FAILED: %s:%d\n", __FILE__, __LINE__); \
            printf("  Expected: %lld\n", (long long)(expected)); \
            printf("  Actual:   %lld\n", (long long)(actual)); \
            return GRAPHITE_TEST_FAIL; \
        } \
    } while(0)

#define GRAPHITE_ASSERT_STR_EQ(expected, actual) \
    do { \
        if (strcmp((expected), (actual)) != 0) { \
            printf("ASSERTION FAILED: %s:%d\n", __FILE__, __LINE__); \
            printf("  Expected: \"%s\"\n", (expected)); \
            printf("  Actual:   \"%s\"\n", (actual)); \
            return GRAPHITE_TEST_FAIL; \
        } \
    } while(0)

#define GRAPHITE_ASSERT_NOT_NULL(ptr) \
    do { \
        if ((ptr) == NULL) { \
            printf("ASSERTION FAILED: %s:%d\n", __FILE__, __LINE__); \
            printf("  Expected non-NULL pointer\n"); \
            return GRAPHITE_TEST_FAIL; \
        } \
    } while(0)

#define GRAPHITE_ASSERT_NULL(ptr) \
    do { \
        if ((ptr) != NULL) { \
            printf("ASSERTION FAILED: %s:%d\n", __FILE__, __LINE__); \
            printf("  Expected NULL pointer\n"); \
            return GRAPHITE_TEST_FAIL; \
        } \
    } while(0)

// Test fixture support
typedef struct {
    void (*setup)(void);
    void (*teardown)(void);
    void* fixture_data;
} graphite_test_fixture;

// Example unit tests
graphite_test_result test_bundle_creation(void) {
    // Test basic bundle creation
    graphite_bundle_builder* builder = graphite_builder_create();
    GRAPHITE_ASSERT_NOT_NULL(builder);
    
    // Add some test data
    const char* test_string = "Hello, GRAPHITE!";
    uint32_t string_id = graphite_builder_add_string(builder, test_string);
    GRAPHITE_ASSERT_EQ(0, string_id); // First string should have ID 0
    
    // Create bundle
    graphite_bundle* bundle = graphite_builder_finalize(builder);
    GRAPHITE_ASSERT_NOT_NULL(bundle);
    
    // Verify string retrieval
    const char* retrieved = graphite_get_string(bundle, string_id);
    GRAPHITE_ASSERT_STR_EQ(test_string, retrieved);
    
    // Cleanup
    graphite_close(bundle);
    graphite_builder_destroy(builder);
    
    return GRAPHITE_TEST_PASS;
}

graphite_test_result test_graph_traversal(void) {
    // Create a simple graph: A -> B -> C
    graphite_bundle_builder* builder = graphite_builder_create();
    
    // Create nodes
    graphite_node_id node_a = graphite_builder_create_node(builder);
    graphite_node_id node_b = graphite_builder_create_node(builder);
    graphite_node_id node_c = graphite_builder_create_node(builder);
    
    // Create edges
    graphite_builder_add_edge(builder, node_a, node_b, GRAPHITE_EDGE_REFERENCE);
    graphite_builder_add_edge(builder, node_b, node_c, GRAPHITE_EDGE_REFERENCE);
    
    // Finalize bundle
    graphite_bundle* bundle = graphite_builder_finalize(builder);
    GRAPHITE_ASSERT_NOT_NULL(bundle);
    
    // Test traversal
    const graphite_graph* root = graphite_root(bundle);
    GRAPHITE_ASSERT_NOT_NULL(root);
    
    // Verify graph structure
    GRAPHITE_ASSERT_EQ(3, root->header.node_cnt);
    GRAPHITE_ASSERT_EQ(2, root->header.edge_cnt);
    
    // Test edge traversal
    const graphite_graph* node_a_graph = graphite_get_child_graph(root, 0);
    GRAPHITE_ASSERT_NOT_NULL(node_a_graph);
    
    // Cleanup
    graphite_close(bundle);
    graphite_builder_destroy(builder);
    
    return GRAPHITE_TEST_PASS;
}

// Property-based testing support
typedef struct {
    void* (*generate)(size_t seed);
    void (*free_data)(void* data);
    bool (*property)(void* data);
    const char* description;
} graphite_property_test;

bool property_bundle_roundtrip(void* data) {
    graphite_test_data* test = (graphite_test_data*)data;
    
    // Create bundle with random data
    graphite_bundle_builder* builder = graphite_builder_create();
    
    for (size_t i = 0; i < test->string_count; i++) {
        graphite_builder_add_string(builder, test->strings[i]);
    }
    
    for (size_t i = 0; i < test->blob_count; i++) {
        graphite_builder_add_blob(builder, test->blobs[i].data, test->blobs[i].size);
    }
    
    graphite_bundle* bundle = graphite_builder_finalize(builder);
    if (!bundle) return false;
    
    // Verify all data can be retrieved correctly
    for (size_t i = 0; i < test->string_count; i++) {
        const char* retrieved = graphite_get_string(bundle, i);
        if (strcmp(test->strings[i], retrieved) != 0) {
            graphite_close(bundle);
            graphite_builder_destroy(builder);
            return false;
        }
    }
    
    for (size_t i = 0; i < test->blob_count; i++) {
        size_t size;
        const void* retrieved = graphite_get_blob(bundle, i, &size);
        if (size != test->blobs[i].size || 
            memcmp(retrieved, test->blobs[i].data, size) != 0) {
            graphite_close(bundle);
            graphite_builder_destroy(builder);
            return false;
        }
    }
    
    graphite_close(bundle);
    graphite_builder_destroy(builder);
    return true;
}

// Fuzz testing integration
typedef struct {
    uint8_t* data;
    size_t size;
    uint32_t seed;
} graphite_fuzz_input;

void graphite_fuzz_bundle_parser(const graphite_fuzz_input* input) {
    // Parse bundle with fuzzing input
    graphite_bundle* bundle = graphite_open_from_memory(input->data, input->size);
    
    if (bundle) {
        // If parsing succeeded, try to access data safely
        const graphite_graph* root = graphite_root(bundle);
        if (root) {
            // Traverse graph carefully
            for (uint32_t i = 0; i < root->header.node_cnt && i < 1000; i++) {
                const graphite_graph* child = graphite_get_child_graph(root, i);
                if (child) {
                    // Access some data
                    volatile uint32_t dummy = child->header.node_cnt;
                    (void)dummy;
                }
            }
        }
        
        graphite_close(bundle);
    }
}

// Test runner
void graphite_run_tests(void) {
    graphite_test_suite suite = {0};
    
    // Register tests
    graphite_test_case tests[] = {
        {"Bundle Creation", "Core", test_bundle_creation, 0, NULL},
        {"Graph Traversal", "Core", test_graph_traversal, 0, NULL},
        // Add more tests...
    };
    
    suite.tests = tests;
    suite.test_count = sizeof(tests) / sizeof(tests[0]);
    
    printf("Running GRAPHITE test suite...\n");
    printf("================================\n");
    
    for (size_t i = 0; i < suite.test_count; i++) {
        graphite_test_case* test = &suite.tests[i];
        
        printf("Running %s::%s... ", test->suite, test->name);
        fflush(stdout);
        
        uint64_t start_time = graphite_get_precise_time_ns();
        graphite_test_result result = test->test_func();
        uint64_t end_time = graphite_get_precise_time_ns();
        
        test->execution_time_ns = end_time - start_time;
        
        switch (result) {
            case GRAPHITE_TEST_PASS:
                printf("PASS (%.2f ms)\n", test->execution_time_ns / 1e6);
                suite.passed++;
                break;
            case GRAPHITE_TEST_FAIL:
                printf("FAIL (%.2f ms)\n", test->execution_time_ns / 1e6);
                suite.failed++;
                break;
            case GRAPHITE_TEST_SKIP:
                printf("SKIP\n");
                suite.skipped++;
                break;
        }
        
        suite.total_time_ns += test->execution_time_ns;
    }
    
    printf("\n");
    printf("Results: %zu passed, %zu failed, %zu skipped\n",
           suite.passed, suite.failed, suite.skipped);
    printf("Total time: %.2f ms\n", suite.total_time_ns / 1e6);
    
    if (suite.failed > 0) {
        printf("TESTS FAILED!\n");
        exit(1);
    } else {
        printf("All tests passed!\n");
    }
}
```

### 23.2 Integration Testing

#### 23.2.1 End-to-End Pipeline Tests
```c
// Integration test framework
typedef struct {
    const char* temp_directory;
    char** created_files;
    size_t file_count;
    graphite_test_fixture fixture;
} graphite_integration_context;

// Test full asset pipeline
graphite_test_result test_asset_pipeline_integration(void) {
    graphite_integration_context ctx = {0};
    ctx.temp_directory = "/tmp/graphite_test_XXXXXX";
    
    // Create temporary directory
    char* temp_dir = mkdtemp((char*)ctx.temp_directory);
    GRAPHITE_ASSERT_NOT_NULL(temp_dir);
    
    // Create test assets
    char texture_path[256];
    snprintf(texture_path, sizeof(texture_path), "%s/test_texture.png", temp_dir);
    
    // Generate test PNG file
    graphite_result result = graphite_create_test_png(texture_path, 256, 256);
    GRAPHITE_ASSERT_EQ(GRAPHITE_SUCCESS, result);
    
    char mesh_path[256];
    snprintf(mesh_path, sizeof(mesh_path), "%s/test_mesh.gltf", temp_dir);
    
    // Generate test GLTF file
    result = graphite_create_test_gltf(mesh_path, texture_path);
    GRAPHITE_ASSERT_EQ(GRAPHITE_SUCCESS, result);
    
    // Create transform configuration
    char config_path[256];
    snprintf(config_path, sizeof(config_path), "%s/transform_config.json", temp_dir);
    
    const char* config_json = 
        "{\n"
        "  \"transforms\": [\n"
        "    {\n"
        "      \"name\": \"texture_compress\",\n"
        "      \"input_patterns\": [\"*.png\"],\n"
        "      \"output_format\": \"bc7\",\n"
        "      \"parameters\": {\"quality\": \"high\"}\n"
        "    },\n"
        "    {\n"
        "      \"name\": \"mesh_optimize\",\n"
        "      \"input_patterns\": [\"*.gltf\"],\n"
        "      \"output_format\": \"optimized_mesh\"\n"
        "    }\n"
        "  ]\n"
        "}\n";
    
    FILE* config_file = fopen(config_path, "w");
    GRAPHITE_ASSERT_NOT_NULL(config_file);
    fwrite(config_json, 1, strlen(config_json), config_file);
    fclose(config_file);
    
    // Run asset pipeline
    char bundle_path[256];
    snprintf(bundle_path, sizeof(bundle_path), "%s/output.graphite", temp_dir);
    
    char command[1024];
    snprintf(command, sizeof(command),
             "graphite pack --config %s --output %s %s",
             config_path, bundle_path, temp_dir);
    
    int exit_code = system(command);
    GRAPHITE_ASSERT_EQ(0, exit_code);
    
    // Verify bundle was created
    FILE* bundle_file = fopen(bundle_path, "rb");
    GRAPHITE_ASSERT_NOT_NULL(bundle_file);
    fclose(bundle_file);
    
    // Load and verify bundle
    graphite_bundle* bundle = graphite_open(bundle_path);
    GRAPHITE_ASSERT_NOT_NULL(bundle);
    
    const graphite_graph* root = graphite_root(bundle);
    GRAPHITE_ASSERT_NOT_NULL(root);
    GRAPHITE_ASSERT_EQ(true, root->header.node_cnt > 0);
    
    // Verify integrity
    bool integrity_ok = graphite_verify_integrity(bundle);
    GRAPHITE_ASSERT_EQ(true, integrity_ok);
    
    // Cleanup
    graphite_close(bundle);
    
    // Remove temporary files
    char rm_command[512];
    snprintf(rm_command, sizeof(rm_command), "rm -rf %s", temp_dir);
    system(rm_command);
    
    return GRAPHITE_TEST_PASS;
}

// Multi-platform testing
typedef struct {
    const char* platform_name;
    const char* compiler;
    const char* flags;
    const char* test_command;
} graphite_platform_config;

static const graphite_platform_config test_platforms[] = {
    {"Linux x86_64", "gcc", "-O3 -DNDEBUG", "./test_suite"},
    {"Linux x86_64 Debug", "gcc", "-O0 -g -fsanitize=address", "./test_suite"},
    {"Linux ARM64", "aarch64-linux-gnu-gcc", "-O3 -DNDEBUG", "qemu-aarch64 ./test_suite"},
    {"Windows x64", "x86_64-w64-mingw32-gcc", "-O3 -DNDEBUG", "wine ./test_suite.exe"},
    {"macOS x64", "clang", "-O3 -DNDEBUG -target x86_64-apple-macos10.12", "./test_suite"}
};

void graphite_run_cross_platform_tests(void) {
    size_t platform_count = sizeof(test_platforms) / sizeof(test_platforms[0]);
    
    for (size_t i = 0; i < platform_count; i++) {
        const graphite_platform_config* platform = &test_platforms[i];
        
        printf("Testing on %s...\n", platform->platform_name);
        
        // Build for platform
        char build_command[1024];
        snprintf(build_command, sizeof(build_command),
                "make clean && CC=%s CFLAGS='%s' make test_suite",
                platform->compiler, platform->flags);
        
        int build_result = system(build_command);
        if (build_result != 0) {
            printf("  Build FAILED\n");
            continue;
        }
        
        // Run tests
        int test_result = system(platform->test_command);
        if (test_result == 0) {
            printf("  Tests PASSED\n");
        } else {
            printf("  Tests FAILED (exit code: %d)\n", test_result);
        }
    }
}
```

### 23.3 Performance Testing

#### 23.3.1 Automated Performance Validation
```c
// Performance test framework
typedef struct {
    const char* test_name;
    void (*setup)(void** context);
    void (*benchmark)(void* context, graphite_benchmark_result* result);
    void (*teardown)(void* context);
    
    // Performance thresholds
    uint64_t max_time_ns;
    size_t max_memory_bytes;
    double min_throughput_mbps;
} graphite_performance_test;

// Regression detection
typedef struct {
    graphite_benchmark_result baseline;
    graphite_benchmark_result current;
    double regression_threshold;  // Percentage
} graphite_regression_analysis;

bool graphite_detect_regression(const graphite_regression_analysis* analysis) {
    // Check time regression
    if (analysis->baseline.mean_time_ns > 0) {
        double time_regression = 
            ((double)analysis->current.mean_time_ns - analysis->baseline.mean_time_ns) / 
            analysis->baseline.mean_time_ns * 100.0;
        
        if (time_regression > analysis->regression_threshold) {
            printf("REGRESSION: Time increased by %.1f%%\n", time_regression);
            return true;
        }
    }
    
    // Check memory regression
    if (analysis->baseline.max_memory_bytes > 0) {
        double memory_regression = 
            ((double)analysis->current.max_memory_bytes - analysis->baseline.max_memory_bytes) / 
            analysis->baseline.max_memory_bytes * 100.0;
        
        if (memory_regression > analysis->regression_threshold) {
            printf("REGRESSION: Memory usage increased by %.1f%%\n", memory_regression);
            return true;
        }
    }
    
    return false;
}

// Performance test implementation
void setup_large_bundle_test(void** context) {
    // Create large test bundle
    *context = graphite_create_large_test_bundle(1000000); // 1M nodes
}

void benchmark_large_bundle_open(void* context, graphite_benchmark_result* result) {
    graphite_bundle* test_bundle = (graphite_bundle*)context;
    
    // Benchmark bundle opening
    const int iterations = 100;
    uint64_t* times = malloc(sizeof(uint64_t) * iterations);
    
    for (int i = 0; i < iterations; i++) {
        uint64_t start = graphite_get_precise_time_ns();
        
        graphite_bundle* bundle = graphite_open("large_test.graphite");
        
        uint64_t end = graphite_get_precise_time_ns();
        times[i] = end - start;
        
        if (bundle) {
            graphite_close(bundle);
        }
    }
    
    // Calculate statistics
    qsort(times, iterations, sizeof(uint64_t), compare_uint64);
    result->min_time_ns = times[0];
    result->max_time_ns = times[iterations - 1];
    result->median_time_ns = times[iterations / 2];
    result->p95_time_ns = times[(iterations * 95) / 100];
    result->p99_time_ns = times[(iterations * 99) / 100];
    
    uint64_t sum = 0;
    for (int i = 0; i < iterations; i++) {
        sum += times[i];
    }
    result->mean_time_ns = sum / iterations;
    
    free(times);
}

void teardown_large_bundle_test(void* context) {
    graphite_bundle* test_bundle = (graphite_bundle*)context;
    if (test_bundle) {
        graphite_close(test_bundle);
    }
    unlink("large_test.graphite");
}

// Stress testing
void graphite_stress_test_concurrent_access(void) {
    const char* bundle_path = "stress_test.graphite";
    const int thread_count = 32;
    const int iterations_per_thread = 10000;
    
    // Create test bundle
    graphite_create_test_bundle(bundle_path, 10000); // 10K nodes
    
    // Shared state
    atomic_int successful_operations = ATOMIC_VAR_INIT(0);
    atomic_int failed_operations = ATOMIC_VAR_INIT(0);
    
    // Thread function
    auto stress_worker = [](void* arg) -> void* {
        const char* path = (const char*)arg;
        
        for (int i = 0; i < iterations_per_thread; i++) {
            graphite_bundle* bundle = graphite_open(path);
            if (bundle) {
                // Perform random operations
                const graphite_graph* root = graphite_root(bundle);
                if (root && root->header.node_cnt > 0) {
                    uint32_t random_node = rand() % root->header.node_cnt;
                    const graphite_graph* node = graphite_get_child_graph(root, random_node);
                    if (node) {
                        atomic_fetch_add(&successful_operations, 1);
                    } else {
                        atomic_fetch_add(&failed_operations, 1);
                    }
                } else {
                    atomic_fetch_add(&failed_operations, 1);
                }
                graphite_close(bundle);
            } else {
                atomic_fetch_add(&failed_operations, 1);
            }
        }
        
        return NULL;
    };
    
    // Create threads
    pthread_t* threads = malloc(sizeof(pthread_t) * thread_count);
    
    uint64_t start_time = graphite_get_precise_time_ns();
    
    for (int i = 0; i < thread_count; i++) {
        pthread_create(&threads[i], NULL, stress_worker, (void*)bundle_path);
    }
    
    // Wait for completion
    for (int i = 0; i < thread_count; i++) {
        pthread_join(threads[i], NULL);
    }
    
    uint64_t end_time = graphite_get_precise_time_ns();
    
    // Report results
    int successful = atomic_load(&successful_operations);
    int failed = atomic_load(&failed_operations);
    double duration_seconds = (end_time - start_time) / 1e9;
    double ops_per_second = (successful + failed) / duration_seconds;
    
    printf("Stress Test Results:\n");
    printf("  Threads: %d\n", thread_count);
    printf("  Total operations: %d\n", successful + failed);
    printf("  Successful: %d (%.1f%%)\n", successful, 
           100.0 * successful / (successful + failed));
    printf("  Failed: %d (%.1f%%)\n", failed,
           100.0 * failed / (successful + failed));
    printf("  Duration: %.2f seconds\n", duration_seconds);
    printf("  Throughput: %.0f ops/sec\n", ops_per_second);
    
    free(threads);
    unlink(bundle_path);
    
    // Fail test if too many operations failed
    if (failed > (successful + failed) * 0.01) { // More than 1% failure rate
        printf("STRESS TEST FAILED: Too many failed operations\n");
        exit(1);
    }
}
```

---

## 24. Deployment & Distribution

Comprehensive deployment strategies for GRAPHITE across different environments and distribution channels.

### 24.1 Package Management Integration

#### 24.1.1 Homebrew Formula
```ruby
# Formula/graphite.rb
class Graphite < Formula
  desc "High-performance binary graph format for asset management"
  homepage "https://github.com/graphite-format/graphite"
  url "https://github.com/graphite-format/graphite/archive/v3.0.0.tar.gz"
  sha256 "abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890"
  license "MIT"
  
  depends_on "cmake" => :build
  depends_on "zstd"
  depends_on "blake3"
  depends_on "libsodium"
  
  def install
    # Build with optimizations
    system "cmake", "-S", ".", "-B", "build",
           "-DCMAKE_BUILD_TYPE=Release",
           "-DGRAPHITE_BUILD_TOOLS=ON",
           "-DGRAPHITE_BUILD_TESTS=OFF",
           "-DGRAPHITE_ENABLE_SIMD=ON",
           *std_cmake_args
    system "cmake", "--build", "build"
    system "cmake", "--install", "build"
    
    # Install additional tools
    bin.install "build/tools/graphite-pack"
    bin.install "build/tools/graphite-info"
    bin.install "build/tools/graphite-verify"
    
    # Install headers and libraries
    lib.install "build/lib/libgraphite.a"
    lib.install "build/lib/libgraphite.dylib" if OS.mac?
    lib.install "build/lib/libgraphite.so" if OS.linux?
    
    include.install Dir["include/graphite/*.h"]
    
    # Install CMake configuration
    (lib/"cmake/graphite").install Dir["cmake/*.cmake"]
  end

  test do
    # Create a simple test bundle
    (testpath/"test.json").write <<~EOS
      {
        "assets": [
          {
            "name": "test_asset",
            "type": "generic",
            "data": "SGVsbG8gV29ybGQ="
          }
        ]
      }
    EOS
    
    # Test bundle creation
    system bin/"graphite-pack", "--input", "test.json", "--output", "test.graphite"
    assert_predicate testpath/"test.graphite", :exist?
    
    # Test bundle verification
    system bin/"graphite-verify", "test.graphite"
    
    # Test bundle info
    system bin/"graphite-info", "test.graphite"
  end
end
```

#### 24.1.2 vcpkg Port
```cmake
# ports/graphite/portfile.cmake
vcpkg_from_github(
    OUT_SOURCE_PATH SOURCE_PATH
    REPO graphite-format/graphite
    REF v3.0.0
    SHA512 abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
    HEAD_REF main
)

vcpkg_check_features(OUT_FEATURE_OPTIONS FEATURE_OPTIONS
    FEATURES
        tools       GRAPHITE_BUILD_TOOLS
        tests       GRAPHITE_BUILD_TESTS
        simd        GRAPHITE_ENABLE_SIMD
        encryption  GRAPHITE_ENABLE_ENCRYPTION
)

vcpkg_cmake_configure(
    SOURCE_PATH "${SOURCE_PATH}"
    OPTIONS
        ${FEATURE_OPTIONS}
        -DGRAPHITE_BUILD_SHARED_LIBS=ON
        -DGRAPHITE_INSTALL_CMAKE_CONFIG=ON
)

vcpkg_cmake_build()

if("tests" IN_LIST FEATURES)
    vcpkg_cmake_config_fixup(CONFIG_PATH lib/cmake/graphite-tests)
endif()

vcpkg_cmake_config_fixup(CONFIG_PATH lib/cmake/graphite)

if("tools" IN_LIST FEATURES)
    vcpkg_copy_tools(
        TOOL_NAMES
            graphite-pack
            graphite-info  
            graphite-verify
            graphite-benchmark
        AUTO_CLEAN
    )
endif()

file(REMOVE_RECURSE "${CURRENT_PACKAGES_DIR}/debug/include")
file(REMOVE_RECURSE "${CURRENT_PACKAGES_DIR}/debug/share")

file(INSTALL "${SOURCE_PATH}/LICENSE" DESTINATION "${CURRENT_PACKAGES_DIR}/share/${PORT}" RENAME copyright)
```

### 24.2 Container Deployment

#### 24.2.1 Docker Images
```dockerfile
# Dockerfile.alpine - Minimal production image
FROM alpine:3.18 AS builder

# Install build dependencies
RUN apk add --no-cache \
    build-base \
    cmake \
    git \
    zstd-dev \
    libsodium-dev

# Copy source code
COPY . /src
WORKDIR /src

# Build GRAPHITE
RUN cmake -S . -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DGRAPHITE_BUILD_TOOLS=ON \
    -DGRAPHITE_BUILD_TESTS=OFF \
    -DGRAPHITE_STATIC_LINKING=ON && \
    cmake --build build --parallel $(nproc) && \
    cmake --install build --prefix /usr/local

# Production stage
FROM alpine:3.18

# Install runtime dependencies
RUN apk add --no-cache \
    zstd \
    libsodium

# Copy built binaries
COPY --from=builder /usr/local/bin/graphite* /usr/local/bin/
COPY --from=builder /usr/local/lib/libgraphite* /usr/local/lib/
COPY --from=builder /usr/local/include/graphite /usr/local/include/graphite

# Create working directory
WORKDIR /data

# Default command
ENTRYPOINT ["graphite"]
CMD ["--help"]

# Labels
LABEL org.opencontainers.image.title="GRAPHITE"
LABEL org.opencontainers.image.description="High-performance binary graph format"
LABEL org.opencontainers.image.version="3.0.0"
LABEL org.opencontainers.image.source="https://github.com/graphite-format/graphite"
```

```dockerfile
# Dockerfile.ubuntu - Development image with debugging tools
FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    cmake \
    git \
    libzstd-dev \
    libsodium-dev \
    valgrind \
    gdb \
    perf-tools-unstable \
    strace \
    && rm -rf /var/lib/apt/lists/*

# Copy source and build
COPY . /src
WORKDIR /src

RUN cmake -S . -B build \
    -DCMAKE_BUILD_TYPE=Debug \
    -DGRAPHITE_BUILD_TOOLS=ON \
    -DGRAPHITE_BUILD_TESTS=ON \
    -DGRAPHITE_ENABLE_SANITIZERS=ON && \
    cmake --build build --parallel $(nproc) && \
    cmake --install build

# Install debugging symbols
RUN mkdir -p /usr/lib/debug/usr/local/bin && \
    cp build/tools/*.debug /usr/lib/debug/usr/local/bin/ 2>/dev/null || true

WORKDIR /data
ENTRYPOINT ["graphite"]
```

#### 24.2.2 Kubernetes Deployment
```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: graphite-system
  labels:
    name: graphite-system

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: graphite-config
  namespace: graphite-system
data:
  config.json: |
    {
      "compression": {
        "algorithm": "zstd",
        "level": 9
      },
      "security": {
        "require_signature": true,
        "max_bundle_size_mb": 1024
      },
      "performance": {
        "worker_threads": 4,
        "memory_pool_size_mb": 256
      }
    }

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graphite-processor
  namespace: graphite-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: graphite-processor
  template:
    metadata:
      labels:
        app: graphite-processor
    spec:
      containers:
      - name: graphite
        image: graphite:3.0.0
        ports:
        - containerPort: 8080
        env:
        - name: GRAPHITE_CONFIG
          value: "/etc/graphite/config.json"
        - name: GRAPHITE_LOG_LEVEL
          value: "INFO"
        volumeMounts:
        - name: config
          mountPath: /etc/graphite
        - name: data
          mountPath: /data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: graphite-config
      - name: data
        persistentVolumeClaim:
          claimName: graphite-storage

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: graphite-service
  namespace: graphite-system
spec:
  selector:
    app: graphite-processor
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  type: ClusterIP

---
# k8s/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: graphite-storage
  namespace: graphite-system
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd
```

### 24.3 Cloud Deployment

#### 24.3.1 AWS Deployment with Terraform
```hcl
# terraform/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC and Networking
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "graphite-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Environment = var.environment
    Project     = "graphite"
  }
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks/aws"
  
  cluster_name    = "graphite-cluster"
  cluster_version = "1.27"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  
  # Node groups
  eks_managed_node_groups = {
    general = {
      min_size     = 2
      max_size     = 10
      desired_size = 3
      
      instance_types = ["m5.large"]
      capacity_type  = "ON_DEMAND"
      
      k8s_labels = {
        Environment = var.environment
        NodeGroup   = "general"
      }
    }
    
    compute_optimized = {
      min_size     = 0
      max_size     = 5
      desired_size = 1
      
      instance_types = ["c5.xlarge"]
      capacity_type  = "SPOT"
      
      k8s_labels = {
        Environment = var.environment
        NodeGroup   = "compute"
      }
      
      taints = {
        dedicated = {
          key    = "compute-optimized"
          value  = "true"
          effect = "NO_SCHEDULE"
        }
      }
    }
  }
  
  tags = {
    Environment = var.environment
    Project     = "graphite"
  }
}

# S3 Bucket for asset storage
resource "aws_s3_bucket" "graphite_assets" {
  bucket = "graphite-assets-${var.environment}-${random_string.bucket_suffix.result}"
  
  tags = {
    Environment = var.environment
    Project     = "graphite"
  }
}

resource "aws_s3_bucket_versioning" "graphite_assets" {
  bucket = aws_s3_bucket.graphite_assets.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "graphite_assets" {
  bucket = aws_s3_bucket.graphite_assets.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# CloudFront distribution for CDN
resource "aws_cloudfront_distribution" "graphite_cdn" {
  origin {
    domain_name = aws_s3_bucket.graphite_assets.bucket_regional_domain_name
    origin_id   = "graphite-s3-origin"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.graphite_oai.cloudfront_access_identity_path
    }
  }
  
  enabled = true
  
  default_cache_behavior {
    target_origin_id       = "graphite-s3-origin"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    cloudfront_default_certificate = true
  }
  
  tags = {
    Environment = var.environment
    Project     = "graphite"
  }
}

resource "aws_cloudfront_origin_access_identity" "graphite_oai" {
  comment = "OAI for GRAPHITE assets"
}

# Random string for S3 bucket uniqueness
resource "random_string" "bucket_suffix" {
  length  = 8
  special = false
  upper   = false
}

# Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

# Outputs
output "eks_cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "s3_bucket_name" {
  value = aws_s3_bucket.graphite_assets.bucket
}

output "cloudfront_domain_name" {
  value = aws_cloudfront_distribution.graphite_cdn.domain_name
}
```

#### 24.3.2 Azure Resource Manager Template
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "type": "string",
      "defaultValue": "production",
      "allowedValues": ["development", "staging", "production"]
    },
    "location": {
      "type": "string",
      "defaultValue": "[resourceGroup().location]"
    }
  },
  "variables": {
    "resourcePrefix": "[concat('graphite-', parameters('environment'))]",
    "storageAccountName": "[concat(replace(variables('resourcePrefix'), '-', ''), uniqueString(resourceGroup().id))]",
    "aksClusterName": "[concat(variables('resourcePrefix'), '-aks')]",
    "acrName": "[concat(replace(variables('resourcePrefix'), '-', ''), 'acr')]"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2021-09-01",
      "name": "[variables('storageAccountName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard_LRS"
      },
      "kind": "StorageV2",
      "properties": {
        "encryption": {
          "services": {
            "blob": {
              "enabled": true
            },
            "file": {
              "enabled": true
            }
          },
          "keySource": "Microsoft.Storage"
        },
        "supportsHttpsTrafficOnly": true
      },
      "tags": {
        "Environment": "[parameters('environment')]",
        "Project": "graphite"
      }
    },
    {
      "type": "Microsoft.ContainerRegistry/registries",
      "apiVersion": "2021-09-01",
      "name": "[variables('acrName')]",
      "location": "[parameters('location')]",
      "sku": {
        "name": "Standard"
      },
      "properties": {
        "adminUserEnabled": false
      },
      "tags": {
        "Environment": "[parameters('environment')]",
        "Project": "graphite"
      }
    },
    {
      "type": "Microsoft.ContainerService/managedClusters",
      "apiVersion": "2021-10-01",
      "name": "[variables('aksClusterName')]",
      "location": "[parameters('location')]",
      "identity": {
        "type": "SystemAssigned"
      },
      "properties": {
        "kubernetesVersion": "1.27.3",
        "dnsPrefix": "[variables('resourcePrefix')]",
        "agentPoolProfiles": [
          {
            "name": "general",
            "count": 3,
            "vmSize": "Standard_D4s_v3",
            "osDiskSizeGB": 128,
            "mode": "System",
            "type": "VirtualMachineScaleSets",
            "enableAutoScaling": true,
            "minCount": 2,
            "maxCount": 10
          },
          {
            "name": "compute",
            "count": 1,
            "vmSize": "Standard_F8s_v2",
            "osDiskSizeGB": 128,
            "mode": "User",
            "type": "VirtualMachineScaleSets",
            "enableAutoScaling": true,
            "minCount": 0,
            "maxCount": 5,
            "nodeLabels": {
              "workload": "compute-intensive"
            },
            "nodeTaints": [
              "compute-optimized=true:NoSchedule"
            ]
          }
        ],
        "servicePrincipalProfile": {
          "clientId": "msi"
        },
        "networkProfile": {
          "networkPlugin": "azure",
          "loadBalancerSku": "standard"
        }
      },
      "tags": {
        "Environment": "[parameters('environment')]",
        "Project": "graphite"
      }
    }
  ],
  "outputs": {
    "storageAccountName": {
      "type": "string",
      "value": "[variables('storageAccountName')]"
    },
    "aksClusterName": {
      "type": "string",
      "value": "[variables('aksClusterName')]"
    },
    "acrLoginServer": {
      "type": "string",
      "value": "[reference(variables('acrName')).loginServer]"
    }
  }
}
```

This completes sections 22-24. Would you like me to continue with the final sections 25-28?