# Vectors

## `jq`

### Dynamic Array

Core Array Struct (jvp_array)
File: `repos/jq/src/jv.c:822-826`

```c
typedef struct {
  jv_refcnt refcnt;
  int length, alloc_length;
  jv elements[];
} jvp_array;
```

Array Reallocation
File: `repos/jq/src/jv.c:878-909`

```c
static jv* jvp_array_write(jv* a, int i) {
  assert(i >= 0);
  jvp_array* array = jvp_array_ptr(*a);

  int pos = i + jvp_array_offset(*a);
  if (pos < array->alloc_length && jvp_refcnt_unshared(a->u.ptr)) {
    // use existing array space
    for (int j = array->length; j <= pos; j++) {
      array->elements[j] = JV_NULL;
    }
    array->length = imax(pos + 1, array->length);
    a->size = imax(i + 1, a->size);
    return &array->elements[pos];
  } else {
    // allocate a new array
    int new_length = imax(i + 1, jvp_array_length(*a));
    jvp_array* new_array = jvp_array_alloc(ARRAY_SIZE_ROUND_UP(new_length));
    int j;
    for (j = 0; j < jvp_array_length(*a); j++) {
      new_array->elements[j] =
        jv_copy(array->elements[j + jvp_array_offset(*a)]);
    }
    for (; j < new_length; j++) {
      new_array->elements[j] = JV_NULL;
    }
    new_array->length = new_length;
    jvp_array_free(*a);
    jv new_jv = {JVP_FLAGS_ARRAY, 0, 0, new_length, {&new_array->refcnt}};
    *a = new_jv;
    return &new_array->elements[i];
  }
}
```

### Dynamic String

Dynamic String Struct (jvp_string)
File: `repos/jq/src/jv.c:1090-1098`

```c
typedef struct {
  jv_refcnt refcnt;
  uint32_t hash;
  uint32_t length_hashed;
  uint32_t alloc_length;
  char data[];
} jvp_string;
```

String Reallocation
File: `repos/jq/src/jv.c:1174-1198`

```C
static jv jvp_string_append(jv string, const char* data, uint32_t len) {
  jvp_string* s = jvp_string_ptr(string);
  uint32_t currlen = jvp_string_length(s);

  if (jvp_refcnt_unshared(string.u.ptr) &&
      jvp_string_remaining_space(s) >= len) {
    // the next string fits at the end of a
    memcpy(s->data + currlen, data, len);
    s->data[currlen + len] = 0;
    s->length_hashed = (currlen + len) << 1;
    return string;
  } else {
    // allocate a bigger buffer and copy
    uint32_t allocsz = (currlen + len) * 2;
    if (allocsz < 32) allocsz = 32;
    jvp_string* news = jvp_string_alloc(allocsz);
    news->length_hashed = (currlen + len) << 1;
    memcpy(news->data, s->data, currlen);
    memcpy(news->data + currlen, data, len);
    news->data[currlen + len] = 0;
    jvp_string_free(string);
    jv r = {JVP_FLAGS_STRING, 0, 0, 0, {&news->refcnt}};
    return r;
  }
}

```

### Hash Table

Hash Table Struct (jvp_object)
File: `repos/jq/src/jv.c:1572-1576`

```C
typedef struct {
  jv_refcnt refcnt;
  int next_free;
  struct object_slot elements[];
} jvp_object;
struct object_slot {
  int next;
  uint32_t hash;
  jv string;
  jv value;
};
```

### Stack

Stack Struct (struct stack)
File: `repos/jq/src/exec_stack.h:50-54`

```C
struct stack {
  char* mem_end;
  stack_ptr bound;
  stack_ptr limit;
};
```

Stack Reallocation
File: `repos/jq/src/exec_stack.h:83-93`

```C
static void stack_reallocate(struct stack* s, size_t sz) {
  int old_mem_length = -(s->bound) + ALIGNMENT;
  char* old_mem_start = (s->mem_end != NULL) ? (s->mem_end - old_mem_length) : NULL;

  int new_mem_length = align_round_up((old_mem_length + sz + 256) * 2);
  char* new_mem_start = jv_mem_realloc(old_mem_start, new_mem_length);
  memmove(new_mem_start + (new_mem_length - old_mem_length),
            new_mem_start, old_mem_length);
  s->mem_end = new_mem_start + new_mem_length;
  s->bound = -(new_mem_length - ALIGNMENT);
}
```
