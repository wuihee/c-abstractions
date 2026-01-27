# `parson`

## Summary

| Data Structure  | Type                               | File               |
| --------------- | ---------------------------------- | ------------------ |
| `json_array_t`  | Dynamic array (malloc+memcpy+free) | `parson.c:144-149` |
| `json_object_t` | Hash table with dynamic arrays     | `parson.c:132-142` |

Note: Parson uses `malloc+memcpy+free` instead of `realloc` for capacity growth.

---

## `json_array_t` - Dynamic Array

File: `repos/parson/parson.c:144-149`

```c
struct json_array_t {
    JSON_Value  *wrapping_value;
    JSON_Value **items;
    size_t       count;
    size_t       capacity;
};
```

A dynamic array of JSON values. Uses a doubling growth strategy. The `wrapping_value` points to the parent `JSON_Value` that contains this array.

```rust
struct JsonArray {
    items: Vec<JsonValue>,
}
```

### Initialization

File: `repos/parson/parson.c:721-731`

```c
static JSON_Array * json_array_make(JSON_Value *wrapping_value) {
    JSON_Array *new_array = (JSON_Array*)parson_malloc(sizeof(JSON_Array));
    if (new_array == NULL) {
        return NULL;
    }
    new_array->wrapping_value = wrapping_value;
    new_array->items = (JSON_Value**)NULL;
    new_array->capacity = 0;
    new_array->count = 0;
    return new_array;
}
```

Starts with zero capacity. The `items` pointer is initially NULL and allocated on first insert.

### Insert - `json_array_add()`

File: `repos/parson/parson.c:733-743`

```c
static JSON_Status json_array_add(JSON_Array *array, JSON_Value *value) {
    if (array->count >= array->capacity) {
        size_t new_capacity = MAX(array->capacity * 2, STARTING_CAPACITY);
        if (json_array_resize(array, new_capacity) != JSONSuccess) {
            return JSONFailure;
        }
    }
    value->parent = json_array_get_wrapping_value(array);
    array->items[array->count] = value;
    array->count++;
    return JSONSuccess;
}
```

Growth strategy: Doubles capacity (`capacity * 2`), with a minimum of `STARTING_CAPACITY` (32).

### Resize - `json_array_resize()`

File: `repos/parson/parson.c:746-762`

```c
static JSON_Status json_array_resize(JSON_Array *array, size_t new_capacity) {
    JSON_Value **new_items = NULL;
    if (new_capacity == 0) {
        return JSONFailure;
    }
    new_items = (JSON_Value**)parson_malloc(new_capacity * sizeof(JSON_Value*));
    if (new_items == NULL) {
        return JSONFailure;
    }
    if (array->items != NULL && array->count > 0) {
        memcpy(new_items, array->items, array->count * sizeof(JSON_Value*));
    }
    parson_free(array->items);
    array->items = new_items;
    array->capacity = new_capacity;
    return JSONSuccess;
}
```

Uses `malloc+memcpy+free` pattern instead of `realloc`. Allocates new buffer, copies existing items, frees old buffer.

---

## `json_object_t` - Hash Table

File: `repos/parson/parson.c:132-142`

```c
struct json_object_t {
    JSON_Value    *wrapping_value;
    size_t        *cells;
    unsigned long *hashes;
    char         **names;
    JSON_Value   **values;
    size_t        *cell_ixs;
    size_t         count;
    size_t         item_capacity;
    size_t         cell_capacity;
};
```

An open-addressed hash table for JSON objects. Uses multiple parallel arrays: `cells` for the hash table buckets, `names` and `values` for key-value storage, `hashes` for cached hash values, and `cell_ixs` for reverse lookup. The `item_capacity` is set to 70% of `cell_capacity` to maintain load factor.

```rust
struct JsonObject {
    entries: HashMap<String, JsonValue>,
}
```

### Growth - `json_object_grow_and_rehash()`

File: `repos/parson/parson.c:525-553`

```c
static JSON_Status json_object_grow_and_rehash(JSON_Object *object) {
    JSON_Value *wrapping_value = NULL;
    JSON_Object new_object;
    char *key = NULL;
    JSON_Value *value = NULL;
    unsigned int i = 0;
    size_t new_capacity = MAX(object->cell_capacity * 2, STARTING_CAPACITY);
    JSON_Status res = json_object_init(&new_object, new_capacity);
    if (res != JSONSuccess) {
        return JSONFailure;
    }

    wrapping_value = json_object_get_wrapping_value(object);
    new_object.wrapping_value = wrapping_value;

    for (i = 0; i < object->count; i++) {
        key = object->names[i];
        value = object->values[i];
        res = json_object_add(&new_object, key, value);
        // ...
    }
    json_object_deinit(object, PARSON_FALSE, PARSON_FALSE);
    *object = new_object;
    return JSONSuccess;
}
```

Growth strategy: Doubles `cell_capacity`, creates a new hash table, rehashes all entries, then replaces the old object. This is typical for open-addressed hash tables where rehashing is required on resize.
