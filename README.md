# C Abstractions

A dynamic array candidate must satisfy:

Structural:

- A `struct` containing:
  - One pointer field.
  - One length field.
  - One capacity field.

Behavioral:

- Capacity growth via `realloc`.
- Insert logic increments length.
- Access via `data[index]` or `data[len]`.

Excluded:

- Macro-only arrays (stb_ds, uthash)
- Arrays without capacity tracking
- Pure stack arrays
