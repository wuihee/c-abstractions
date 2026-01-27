# `cJSON`

## Summary

| Data Structure | Type                               | File              |
| -------------- | ---------------------------------- | ----------------- |
| `printbuffer`  | Dynamic string (doubling strategy) | `cJSON.c:473-482` |

### Non-Qualifying Data Structures

| Data Structure   | Type                    | File              | Why Excluded                    |
| ---------------- | ----------------------- | ----------------- | ------------------------------- |
| `cJSON`          | JSON tree node          | `cJSON.h:103-123` | Linked list, no capacity growth |
| `internal_hooks` | Allocator function ptrs | `cJSON.c:156-161` | No dynamic behavior             |

---

## `printbuffer` - Dynamic String Buffer

File: `repos/cJSON/cJSON.c:473-482`

```c
typedef struct
{
    unsigned char *buffer;
    size_t length;
    size_t offset;
    size_t depth; /* current nesting depth (for formatted printing) */
    cJSON_bool noalloc;
    cJSON_bool format; /* is this print a formatted print */
    internal_hooks hooks;
} printbuffer;
```

A growable byte buffer used for JSON serialization. Uses a doubling growth strategy. The `offset` field tracks the current write position (used bytes), while `length` is the allocated capacity. The `noalloc` flag disables growth for preallocated buffers. Supports custom allocators via `hooks`.

```rust
struct PrintBuffer {
    data: Vec<u8>,
    depth: usize,
    format: bool,
}
```

### Reallocation - `ensure()`

File: `repos/cJSON/cJSON.c:485-568`

```c
/* realloc printbuffer if necessary to have at least "needed" bytes more */
static unsigned char* ensure(printbuffer * const p, size_t needed)
{
    unsigned char *newbuffer = NULL;
    size_t newsize = 0;

    if ((p == NULL) || (p->buffer == NULL))
    {
        return NULL;
    }

    if ((p->length > 0) && (p->offset >= p->length))
    {
        /* make sure that offset is valid */
        return NULL;
    }

    needed += p->offset + 1;
    if (needed <= p->length)
    {
        return p->buffer + p->offset;
    }

    if (p->noalloc) {
        return NULL;
    }

    /* calculate new buffer size */
    if (needed > (INT_MAX / 2))
    {
        if (needed <= INT_MAX)
        {
            newsize = INT_MAX;
        }
        else
        {
            return NULL;
        }
    }
    else
    {
        newsize = needed * 2;
    }

    if (p->hooks.reallocate != NULL)
    {
        /* reallocate with realloc if available */
        newbuffer = (unsigned char*)p->hooks.reallocate(p->buffer, newsize);
        // ...
    }
    p->length = newsize;
    p->buffer = newbuffer;

    return newbuffer + p->offset;
}
```

Growth strategy: Doubles the required size (`newsize = needed * 2`), capped at `INT_MAX`. Returns a pointer to the write position after ensuring capacity.

### Initialization

File: `repos/cJSON/cJSON.c:1234-1246`

```c
static unsigned char *print(const cJSON * const item, cJSON_bool format, const internal_hooks * const hooks)
{
    static const size_t default_buffer_size = 256;
    printbuffer buffer[1];

    memset(buffer, 0, sizeof(buffer));

    /* create buffer */
    buffer->buffer = (unsigned char*) hooks->allocate(default_buffer_size);
    buffer->length = default_buffer_size;
    buffer->format = format;
    buffer->hooks = *hooks;
    // ...
}
```

Default initial capacity is 256 bytes. The buffer is stack-allocated as a single-element array (`buffer[1]`) to allow passing by pointer.

---

## Non-Qualifying: `cJSON` - JSON Tree Node

File: `repos/cJSON/cJSON.h:103-123`

```c
typedef struct cJSON
{
    /* next/prev allow you to walk array/object chains. */
    struct cJSON *next;
    struct cJSON *prev;
    /* An array or object item will have a child pointer pointing to a chain of the items in the array/object. */
    struct cJSON *child;

    int type;

    char *valuestring;
    int valueint;
    double valuedouble;

    char *string;
} cJSON;
```

The main JSON node structure. Arrays and objects are represented as doubly-linked lists of children (accessed via `child`, `next`, `prev`), not as contiguous dynamic arrays.

**Why excluded:** No capacity field, no `realloc` growth. Uses linked list traversal rather than indexed access.
