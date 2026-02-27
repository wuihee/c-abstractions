# C Abstractions

The goal of this project is to improve C-to-Rust translation quality by systematically preprocessing common data structure abstractions in C before running c2rust. The core idea is to first identify recurring abstractions (such as dynamic arrays or hash maps) and translate or normalize them into well-defined C library equivalents with clear specifications about what should be transformed and how. This involves building a preprocessing pipeline: collecting representative test cases, analyzing how data structures are implemented and used, and then transforming these implementations to call into a standardized library API. Tools such as clang-tidy can assist in performing and enforcing these transformations automatically. Ultimately, the main objective is to formalize these rewrites as a well-defined transformation algorithm, so that programs are systematically rewritten to use consistent abstractions before translation, leading to cleaner and more reliable Rust output.

## Dynamic Array

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

## Transformation

We have three options for transforming data structures:

1. Replace data structures within existing interfaces.
2. Remove existing interfaces and use data structures inline.
3. Write our own interfaces.

## TODOs

- Inline then interface transformation for `parson`.
- Transform mpc.
