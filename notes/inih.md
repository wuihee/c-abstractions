# `inih`

## Summary

| Data Structure         | Type                               | File            |
| ---------------------- | ---------------------------------- | --------------- |
| Line buffer            | Dynamic buffer (doubling strategy) | `ini.c:105-159` |
| `ini_parse_string_ctx` | String parsing context             | `ini.c:42-45`   |

**Note:** inih is a minimal INI parser with few data structures. The main dynamic buffer is implemented using local variables rather than a struct. Dynamic reallocation is disabled by default (`INI_ALLOW_REALLOC=0`).

---

## Dynamic Line Buffer

Buffer Variables
File: `repos/inih/ini.c:105-106`

```c
#if !INI_USE_STACK
    char* line;
    size_t max_line = INI_INITIAL_ALLOC;
#endif
```

Buffer Initialization
File: `repos/inih/ini.c:127-130`

```c
    line = (char*)ini_malloc(INI_INITIAL_ALLOC);
    if (!line) {
        return -2;
    }
```

Buffer Reallocation (Doubling Strategy)
File: `repos/inih/ini.c:143-159`

```c
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
            if (reader(line + offset, (int)(max_line - offset), stream) == NULL)
                break;
            offset += strlen(line + offset);
        }
#endif
```

Buffer Deallocation
File: `repos/inih/ini.c:260-262`

```c
#if !INI_USE_STACK
    ini_free(line);
#endif
```

---

## String Parsing Context

Context Struct (ini_parse_string_ctx)
File: `repos/inih/ini.c:42-45`

```c
typedef struct {
    const char* ptr;
    size_t num_left;
} ini_parse_string_ctx;
```

**Note:** This is a read-only context for tracking position during string parsing. No capacity field or dynamic growth.

---

## Configurable Allocators

Memory Allocation Functions
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

---

## Configuration Options

Relevant compile-time options affecting data structures:

| Option                 | Default | Description                          |
| ---------------------- | ------- | ------------------------------------ |
| `INI_USE_STACK`        | 1       | Use stack-allocated buffer (no heap) |
| `INI_ALLOW_REALLOC`    | 0       | Allow buffer growth via realloc      |
| `INI_INITIAL_ALLOC`    | 200     | Initial heap buffer size             |
| `INI_MAX_LINE`         | 200     | Maximum line length                  |
| `INI_CUSTOM_ALLOCATOR` | 0       | Use custom memory allocator          |
