# `inih`

## Summary

### Layer 1: Primitive Data Structures

| Data Structure         | Type                               | File            |
| ---------------------- | ---------------------------------- | --------------- |
| Line buffer            | Dynamic buffer (doubling strategy) | `ini.c:105-159` |
| `ini_parse_string_ctx` | String parsing context             | `ini.c:42-45`   |

**Note:** inih is a minimal INI parser with few data structures. The main dynamic buffer is implemented using local variables rather than a struct. Dynamic reallocation is disabled by default (`INI_ALLOW_REALLOC=0`).

---

# Layer 1: Primitive Data Structures

## Line Buffer - Dynamic Buffer

File: `repos/inih/ini.c:105-106`, `repos/inih/ini.c:127-130`, `repos/inih/ini.c:143-159`

```c
#if !INI_USE_STACK
    char* line;
    size_t max_line = INI_INITIAL_ALLOC;
#endif

// Initialization
line = (char*)ini_malloc(INI_INITIAL_ALLOC);
if (!line) {
    return -2;
}

// Reallocation (doubling strategy)
#if INI_ALLOW_REALLOC && !INI_USE_STACK
    while (max_line < INI_MAX_LINE &&
           offset == max_line - 1 && line[offset - 1] != '\n') {
        max_line *= 2;
        if (max_line > INI_MAX_LINE)
            max_line = INI_MAX_LINE;
        new_line = ini_realloc(line, max_line);
        if (!new_line) {
            ini_free(line);
            return -2;
        }
        line = new_line;
        // continue reading...
    }
#endif
```

A heap-allocated line buffer using local variables (`line`, `max_line`) instead of a struct. Uses doubling growth strategy when `INI_ALLOW_REALLOC` is enabled. Capped at `INI_MAX_LINE` to prevent unbounded growth. The non-struct approach keeps the library minimal and avoids exposing internal types.

```rust
struct LineBuffer {
    data: Vec<u8>,
    max_size: usize,
}
```

---

## `ini_parse_string_ctx` - String Parsing Context

File: `repos/inih/ini.c:42-45`

```c
typedef struct {
    const char* ptr;
    size_t num_left;
} ini_parse_string_ctx;
```

A read-only context for parsing INI content from an in-memory string. Tracks the current position (`ptr`) and remaining bytes (`num_left`). No capacity field or dynamic growth since it only reads existing data. Used by `ini_parse_string()` to parse pre-loaded configuration.

```rust
struct IniParseStringCtx<'a> {
    remaining: &'a str,
}
```

---

# Configuration Options

Compile-time options affecting data structures:

| Option                 | Default | Description                          |
| ---------------------- | ------- | ------------------------------------ |
| `INI_USE_STACK`        | 1       | Use stack-allocated buffer (no heap) |
| `INI_ALLOW_REALLOC`    | 0       | Allow buffer growth via realloc      |
| `INI_INITIAL_ALLOC`    | 200     | Initial heap buffer size             |
| `INI_MAX_LINE`         | 200     | Maximum line length                  |
| `INI_CUSTOM_ALLOCATOR` | 0       | Use custom memory allocator          |

---

# Configurable Allocators

File: `repos/inih/ini.c:24-36`

```c
#if INI_CUSTOM_ALLOCATOR
#include <stddef.h>
void* ini_malloc(size_t size);
void ini_free(void* ptr);
void* ini_realloc(void* ptr, size_t size);
#else
#include <stdlib.h>
#define ini_malloc(sz)       malloc(sz)
#define ini_free(ptr)        free(ptr)
#define ini_realloc(ptr, sz) realloc((ptr), (sz))
#endif
```

Pluggable memory allocation for embedded systems. When `INI_CUSTOM_ALLOCATOR` is enabled, the user must provide `ini_malloc`, `ini_free`, and `ini_realloc` implementations. This allows integration with custom memory pools or RTOS allocators.

```rust
trait IniAllocator {
    fn alloc(&self, size: usize) -> *mut u8;
    fn free(&self, ptr: *mut u8);
    fn realloc(&self, ptr: *mut u8, size: usize) -> *mut u8;
}
```
