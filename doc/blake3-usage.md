# BLAKE3 Usage in libfyaml

## Overview

libfyaml includes a **BLAKE3** cryptographic hash function implementation as part of its library. BLAKE3 is a fast, secure cryptographic hash function that is significantly faster than MD5, SHA-1, SHA-2, SHA-3, and BLAKE2.

**Key Point**: BLAKE3 is **not used for YAML parsing** itself. It is provided as a utility feature and public API for applications that need fast cryptographic hashing alongside YAML processing. The library uses XXHash for internal operations like memory deduplication.

## What is BLAKE3?

BLAKE3 is:
- **Fast**: Significantly faster than other cryptographic hash functions
- **Secure**: Based on the BLAKE2 design with improved security
- **Parallel**: Can efficiently use multiple CPU cores
- **SIMD-optimized**: Takes advantage of CPU vector instructions (SSE2, SSE4.1, AVX2, AVX-512, NEON)
- **Versatile**: Supports keyed hashing and key derivation modes

## How libfyaml Uses BLAKE3

libfyaml integrates BLAKE3 in several ways:

### 1. **Embedded Implementation**

The library includes a full BLAKE3 implementation with multiple SIMD backends:
- **Portable**: Works on all platforms (fallback implementation)
- **SSE2**: x86/x86-64 optimization
- **SSE4.1**: Enhanced x86/x86-64 optimization
- **AVX2**: Advanced x86/x86-64 optimization
- **AVX-512**: High-performance x86/x86-64 optimization
- **NEON**: ARM optimization

The implementation is located in `src/blake3/` and includes:
- `blake3.c/h` - Core BLAKE3 API
- `blake3_portable.c` - Portable C implementation
- `blake3_sse2.c`, `blake3_sse41.c`, `blake3_avx2.c`, `blake3_avx512.c` - x86 SIMD variants
- `blake3_neon.c` - ARM NEON optimization
- `blake3_host_state.c` - Host state management (threading, file I/O)
- `blake3_backend.c` - Backend selection and management
- `fy-blake3.c` - libfyaml wrapper API

### 2. **Public API**

libfyaml exposes a **minimal, user-friendly BLAKE3 API** through `libfyaml.h`:

```c
/* BLAKE3 constants */
#define FY_BLAKE3_KEY_LEN 32    /* Key length for keyed mode */
#define FY_BLAKE3_OUT_LEN 32    /* Output hash length */

/* Configuration structure */
struct fy_blake3_hasher_cfg {
    const char *backend;           /* NULL for default, or specific backend name */
    size_t file_buffer;            /* Buffer size for file I/O */
    size_t mmap_min_chunk;         /* Minimum chunk size for mmap */
    size_t mmap_max_chunk;         /* Maximum chunk size for mmap */
    bool no_mmap;                  /* Disable mmap for file access */
    const uint8_t *key;            /* Key for keyed mode (NULL otherwise) */
    const void *context;           /* Context for key derivation mode */
    size_t context_len;            /* Context length */
    struct fy_thread_pool *tp;    /* Thread pool (NULL to create private) */
    int num_threads;               /* Thread count: 0=default (CPUs×3/2), >0=specific, -1=disabled */
};

/* API functions */
struct fy_blake3_hasher *fy_blake3_hasher_create(const struct fy_blake3_hasher_cfg *cfg);
void fy_blake3_hasher_destroy(struct fy_blake3_hasher *fyh);
void fy_blake3_hasher_update(struct fy_blake3_hasher *fyh, const void *input, size_t input_len);
const uint8_t *fy_blake3_hasher_finalize(struct fy_blake3_hasher *fyh);
void fy_blake3_hasher_reset(struct fy_blake3_hasher *fyh);
const uint8_t *fy_blake3_hash(struct fy_blake3_hasher *fyh, const void *mem, size_t size);
const uint8_t *fy_blake3_hash_file(struct fy_blake3_hasher *fyh, const char *filename);
const char *fy_blake3_backend_iterate(const char **prevp);
```

### 3. **fy-tool b3sum Command**

The `fy-tool` utility includes a `b3sum` command (similar to the standalone b3sum tool) for hashing files:

```bash
# Hash a file
fy-tool b3sum file.txt

# Check hashes
fy-tool b3sum --check checksums.txt

# Use specific backend
fy-tool b3sum --backend=avx2 file.txt

# Keyed hashing
fy-tool b3sum --keyed --key=mykey file.txt
```

The b3sum functionality is implemented in `src/tool/fy-tool.c` using the libfyaml BLAKE3 API.

## Usage Examples

### Basic Hashing

```c
#include <libfyaml.h>

int main() {
    struct fy_blake3_hasher_cfg cfg = {0};
    struct fy_blake3_hasher *hasher;
    const uint8_t *hash;
    
    /* Create hasher with default configuration */
    hasher = fy_blake3_hasher_create(&cfg);
    if (!hasher) {
        fprintf(stderr, "Failed to create hasher\n");
        return 1;
    }
    
    /* Hash some data */
    const char *data = "Hello, BLAKE3!";
    hash = fy_blake3_hash(hasher, data, strlen(data));
    
    /* Print hash in hex */
    for (int i = 0; i < FY_BLAKE3_OUT_LEN; i++) {
        printf("%02x", hash[i]);
    }
    printf("\n");
    
    fy_blake3_hasher_destroy(hasher);
    return 0;
}
```

### Streaming Hashing

```c
struct fy_blake3_hasher_cfg cfg = {0};
struct fy_blake3_hasher *hasher = fy_blake3_hasher_create(&cfg);

/* Update hasher with multiple chunks */
fy_blake3_hasher_update(hasher, chunk1, chunk1_len);
fy_blake3_hasher_update(hasher, chunk2, chunk2_len);
fy_blake3_hasher_update(hasher, chunk3, chunk3_len);

/* Finalize and get hash */
const uint8_t *hash = fy_blake3_hasher_finalize(hasher);

fy_blake3_hasher_destroy(hasher);
```

### File Hashing

```c
struct fy_blake3_hasher_cfg cfg = {0};
struct fy_blake3_hasher *hasher = fy_blake3_hasher_create(&cfg);

/* Hash entire file (may use mmap for efficiency) */
const uint8_t *hash = fy_blake3_hash_file(hasher, "largefile.bin");
if (!hash) {
    fprintf(stderr, "Failed to hash file\n");
}

fy_blake3_hasher_destroy(hasher);
```

### Selecting a Specific Backend

```c
struct fy_blake3_hasher_cfg cfg = {
    .backend = "avx2",  /* Use AVX2 backend */
    .num_threads = 4     /* Use 4 threads */
};
struct fy_blake3_hasher *hasher = fy_blake3_hasher_create(&cfg);
```

### Iterating Available Backends

```c
const char *backend = NULL;
printf("Available BLAKE3 backends:\n");
while ((backend = fy_blake3_backend_iterate(&backend)) != NULL) {
    printf("  - %s\n", backend);
}
/* The last backend in the iteration is the default */
```

### Keyed Hashing (MAC)

```c
uint8_t key[FY_BLAKE3_KEY_LEN] = { /* 32-byte key */ };
struct fy_blake3_hasher_cfg cfg = {
    .key = key
};
struct fy_blake3_hasher *hasher = fy_blake3_hasher_create(&cfg);
const uint8_t *mac = fy_blake3_hash(hasher, data, data_len);
fy_blake3_hasher_destroy(hasher);
```

### Key Derivation

```c
const char *context = "myapp 2024-01-01";
struct fy_blake3_hasher_cfg cfg = {
    .context = context,
    .context_len = strlen(context)
};
struct fy_blake3_hasher *hasher = fy_blake3_hasher_create(&cfg);
const uint8_t *derived_key = fy_blake3_hash(hasher, input_key_material, ikm_len);
fy_blake3_hasher_destroy(hasher);
```

## Performance Considerations

### Threading

BLAKE3 can use multiple threads for hashing large inputs:

```c
struct fy_blake3_hasher_cfg cfg = {
    .num_threads = 8  /* Use 8 threads */
};
```

Thread count options:
- **0**: Default (number of CPUs × 3 / 2)
- **> 0**: Specific number of threads
- **-1**: Disable threading entirely

### Memory-Mapped I/O

For file hashing, BLAKE3 can use `mmap()` for better performance:

```c
struct fy_blake3_hasher_cfg cfg = {
    .no_mmap = false,              /* Enable mmap (default) */
    .mmap_min_chunk = 1048576,     /* 1MB minimum */
    .mmap_max_chunk = SIZE_MAX     /* No maximum */
};
```

### Backend Selection

The library automatically selects the best available backend based on CPU capabilities. The selection order (highest to lowest performance):
1. AVX-512 (if supported)
2. AVX2 (if supported)
3. SSE4.1 (if supported)
4. SSE2 (if supported)
5. NEON (on ARM, if supported)
6. Portable (always available)

## Build System Integration

### CMake

BLAKE3 is built as part of libfyaml. The CMake build system:
- Detects CPU capabilities at build time
- Compiles multiple SIMD variants as separate object libraries
- Links them into the main libfyaml library
- Each SIMD variant is compiled with appropriate compiler flags (e.g., `-msse2`, `-mavx2`)

### Source Files

The BLAKE3 implementation consists of:
- Core files (always compiled):
  - `blake3_host_state.c` - Host state and file I/O
  - `blake3_backend.c` - Backend management
  - `blake3_be_cpusimd.c` - CPU SIMD backend
  - `fy-blake3.c` - libfyaml wrapper
  
- Platform-specific SIMD variants (conditionally compiled):
  - `blake3_portable.c` + `blake3.c` - Portable
  - `blake3_sse2.c` + `blake3_sse2_x86-64_unix.S` + `blake3.c` - SSE2
  - `blake3_sse41.c` + `blake3_sse41_x86-64_unix.S` + `blake3.c` - SSE4.1
  - `blake3_avx2.c` + `blake3_avx2_x86-64_unix.S` + `blake3.c` - AVX2
  - `blake3_avx512.c` + `blake3_avx512_x86-64_unix.S` + `blake3.c` - AVX-512
  - `blake3_neon.c` + `blake3.c` - NEON

## Why BLAKE3 in libfyaml?

**Important Note**: BLAKE3 is **NOT currently used** in the actual YAML parsing, serialization, or document manipulation code. The deduplication allocator uses XXHash (XXH64), not BLAKE3.

BLAKE3 is included in libfyaml primarily as:

1. **A Utility Feature**: Provides a fast, high-quality hashing tool via the `fy-tool b3sum` command
2. **A Public API**: Exposed through `libfyaml.h` for users who need fast cryptographic hashing alongside YAML processing
3. **Future-Proofing**: Available for potential future internal optimizations if needed

### Potential Use Cases

While BLAKE3 doesn't participate in YAML parsing directly, users of libfyaml can leverage it for:

1. **Content Addressing**: Hash YAML document content for deduplication or caching systems
2. **File Integrity**: Verify integrity of YAML configuration files
3. **Checksumming**: Generate checksums for YAML files in build systems or deployment pipelines
4. **Key Derivation**: Derive keys from YAML configuration data
5. **Fast Hashing**: General-purpose cryptographic hashing in applications that also use libfyaml

### Why Not Use BLAKE3 for Internal Operations?

The library uses **XXHash (XXH64)** for internal deduplication in the allocator because:
- XXHash is extremely fast for non-cryptographic hashing
- The deduplication allocator doesn't need cryptographic security
- XXHash has lower overhead for small data chunks
- BLAKE3's cryptographic guarantees would be overkill for memory deduplication

## Version

The embedded BLAKE3 implementation is version **1.4.1** (as defined in `src/blake3/blake3.h`).

## References

- BLAKE3 Official Site: https://github.com/BLAKE3-team/BLAKE3
- BLAKE3 Paper: https://github.com/BLAKE3-team/BLAKE3-specs
- libfyaml Repository: https://github.com/pantoniou/libfyaml
